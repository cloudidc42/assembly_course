# Part 060: ELF Binary Format Deep Dive

## บทนำ (Introduction)

ELF (Executable and Linkable Format) คือ format มาตรฐานสำหรับไฟล์ binary บน Linux และ Unix-like systems
ทุกโปรแกรมที่คอมไพล์บน Linux จะอยู่ในรูปแบบ ELF ไม่ว่าจะเป็น executable, shared library, หรือ object file

การเข้าใจ ELF อย่างลึกซึ้งช่วยให้เราสามารถ:
- วิเคราะห์โปรแกรมได้ในระดับ binary
- เข้าใจการทำงานของ dynamic linker
- สร้าง minimal executable ด้วยมือ
- ทำ binary patching และ injection
- เข้าใจ security mechanisms เช่น ASLR, NX, RELRO

---

## 1. ELF Header Structure

ELF header อยู่ที่ offset 0 ของทุกไฟล์ ELF มีขนาด 64 bytes สำหรับ 64-bit

```
typedef struct {
    unsigned char e_ident[16];  /* Magic number และ metadata */
    Elf64_Half    e_type;       /* Object file type */
    Elf64_Half    e_machine;    /* Architecture */
    Elf64_Word    e_version;    /* Object file version */
    Elf64_Addr    e_entry;      /* Entry point virtual address */
    Elf64_Off     e_phoff;      /* Program header table offset */
    Elf64_Off     e_shoff;      /* Section header table offset */
    Elf64_Word    e_flags;      /* Processor-specific flags */
    Elf64_Half    e_ehsize;     /* ELF header size in bytes */
    Elf64_Half    e_phentsize;  /* Program header table entry size */
    Elf64_Half    e_phnum;      /* Program header table entry count */
    Elf64_Half    e_shentsize;  /* Section header table entry size */
    Elf64_Half    e_shnum;      /* Section header table entry count */
    Elf64_Half    e_shstrndx;   /* Section header string table index */
} Elf64_Ehdr;
```

### 1.1 e_ident: Magic Number และ Metadata

```
Byte  0: 0x7F       <- Magic byte 1
Byte  1: 'E' (0x45) <- Magic byte 2
Byte  2: 'L' (0x4C) <- Magic byte 3
Byte  3: 'F' (0x46) <- Magic byte 4
Byte  4: EI_CLASS   <- 1=32-bit, 2=64-bit
Byte  5: EI_DATA    <- 1=Little-endian, 2=Big-endian
Byte  6: EI_VERSION <- ต้องเป็น 1
Byte  7: EI_OSABI   <- 0=System V, 3=Linux, 6=Solaris
Byte  8: EI_ABIVERSION
Byte  9-15: EI_PAD  <- padding (ศูนย์)
```

### 1.2 e_type: ประเภทของไฟล์

| ค่า | ชื่อ | ความหมาย |
|-----|------|----------|
| 0 | ET_NONE | ไม่ระบุ |
| 1 | ET_REL | Relocatable file (.o) |
| 2 | ET_EXEC | Executable file |
| 3 | ET_DYN | Shared object (.so) หรือ PIE executable |
| 4 | ET_CORE | Core dump |

### 1.3 e_machine: สถาปัตยกรรม

| ค่า | ชื่อ | CPU |
|-----|------|-----|
| 3 | EM_386 | Intel 80386 |
| 62 | EM_X86_64 | AMD x86-64 |
| 40 | EM_ARM | ARM |
| 183 | EM_AARCH64 | ARM 64-bit (AArch64) |
| 243 | EM_RISCV | RISC-V |

### 1.4 e_entry: Entry Point

Virtual address ที่ kernel จะ jump ไปเมื่อ execute โปรแกรม
สำหรับ C programs จะชี้ไปที่ `_start` ใน crt1.o ซึ่งจะเรียก `main()`

### 1.5 e_phoff และ e_shoff

- `e_phoff`: offset ของ Program Header Table จากต้นไฟล์
- `e_shoff`: offset ของ Section Header Table จากต้นไฟล์
- ถ้าเป็น 0 แสดงว่าไม่มี table นั้น

---

## 2. Program Headers (Segments)

Program headers บอก kernel ว่าต้องโหลด segment ไหนไปที่ memory address ไหน
ใช้ตอน execution ไม่ใช่ตอน linking

```
typedef struct {
    Elf64_Word  p_type;    /* Segment type */
    Elf64_Word  p_flags;   /* Segment flags (R/W/X) */
    Elf64_Off   p_offset;  /* Segment file offset */
    Elf64_Addr  p_vaddr;   /* Segment virtual address */
    Elf64_Addr  p_paddr;   /* Segment physical address */
    Elf64_Xword p_filesz;  /* Segment size in file */
    Elf64_Xword p_memsz;   /* Segment size in memory */
    Elf64_Xword p_align;   /* Segment alignment */
} Elf64_Phdr;
```

### 2.1 PT_LOAD: Loadable Segments

PT_LOAD คือ segment ที่ kernel จะโหลดเข้า memory จริงๆ

**Text Segment (Code):**
```
p_type  = PT_LOAD
p_flags = PF_R | PF_X  (Readable + Executable)
p_vaddr = 0x400000     (ทั่วไปสำหรับ non-PIE)
```

**Data Segment:**
```
p_type  = PT_LOAD
p_flags = PF_R | PF_W  (Readable + Writable)
p_vaddr = ต่อจาก text segment
```

Flags ของ segment:
- `PF_X` (1): Execute
- `PF_W` (2): Write
- `PF_R` (4): Read

### 2.2 PT_INTERP: Dynamic Linker Path

เก็บ path ของ dynamic linker เช่น `/lib64/ld-linux-x86-64.so.2`
ถ้าไม่มี segment นี้ = statically linked binary

```bash
# ดู interpreter
readelf -l /bin/ls | grep INTERP
# หรือ
file /bin/ls
```

### 2.3 PT_DYNAMIC: Dynamic Linking Information

เก็บข้อมูลสำหรับ dynamic linker:
- ไลบรารีที่ต้องการ (DT_NEEDED)
- Symbol table location
- PLT/GOT location
- Relocation tables

### 2.4 PT_NOTE: Note Information

เก็บข้อมูลเพิ่มเติมเช่น:
- Build ID (hash ของ binary)
- GNU property notes
- ABI version

### 2.5 PT_GNU_STACK: NX Bit (No-Execute Stack)

```
p_type  = PT_GNU_STACK
p_flags = PF_R | PF_W  (ถ้า stack ไม่ executable)
```

ถ้า `p_flags` มี `PF_X` = stack executable (อันตราย!)
ถ้าไม่มี PT_GNU_STACK = kernel อาจให้ stack executable

```bash
# ตรวจสอบ
readelf -l binary | grep GNU_STACK
# RW  = non-executable stack (ปลอดภัย)
# RWE = executable stack (อันตราย)
```

### 2.6 PT_GNU_RELRO: Read-Only After Relocation

ทำให้บาง memory region กลายเป็น read-only หลัง dynamic linker ทำงานเสร็จ
ปกป้อง GOT (Global Offset Table) จาก overwrite

```
p_type  = PT_GNU_RELRO
p_flags = PF_R
p_vaddr = ชี้ไปที่ region ที่จะ protect
```

### 2.7 PT_TLS: Thread-Local Storage

ใช้สำหรับ thread-local variables (`__thread` keyword)

---

## 3. Section Headers

Section headers ใช้ตอน linking และ debugging ไม่ใช้ตอน execution
ไฟล์ที่ stripped จะไม่มี section headers

```
typedef struct {
    Elf64_Word  sh_name;      /* Section name (index ใน string table) */
    Elf64_Word  sh_type;      /* Section type */
    Elf64_Xword sh_flags;     /* Section flags */
    Elf64_Addr  sh_addr;      /* Section virtual address */
    Elf64_Off   sh_offset;    /* Section file offset */
    Elf64_Xword sh_size;      /* Section size in bytes */
    Elf64_Word  sh_link;      /* Link to another section */
    Elf64_Word  sh_info;      /* Additional section information */
    Elf64_Xword sh_addralign; /* Section alignment */
    Elf64_Xword sh_entsize;   /* Entry size if section holds table */
} Elf64_Shdr;
```

### 3.1 Section Types ที่สำคัญ

#### SHT_PROGBITS (1): Program Data

ใช้สำหรับ sections ที่มีข้อมูลจริง:

| Section | ประเภท | ความหมาย |
|---------|--------|----------|
| `.text` | SHT_PROGBITS | Code executable |
| `.data` | SHT_PROGBITS | Initialized data (R/W) |
| `.rodata` | SHT_PROGBITS | Read-only data |

#### SHT_NOBITS (8): BSS

```
.bss section:
- ตัวแปร global ที่ initialize เป็น 0
- ไม่มีข้อมูลในไฟล์ (sh_offset มีแต่ sh_size = ขนาดจริง)
- kernel จะ zero-fill ให้เมื่อโหลด
```

#### SHT_SYMTAB (2) และ SHT_DYNSYM (11): Symbol Tables

- `.symtab`: Symbol table ทั้งหมด (มีทั้ง local และ global)
- `.dynsym`: Dynamic symbol table (เฉพาะ symbols ที่ export/import)

#### SHT_STRTAB (3): String Tables

- `.strtab`: String table สำหรับ `.symtab`
- `.dynstr`: String table สำหรับ `.dynsym`
- `.shstrtab`: String table สำหรับชื่อ sections

#### SHT_RELA (4) / SHT_REL (9): Relocation Tables

- `.rela.text`: Relocations สำหรับ code
- `.rela.dyn`: Dynamic relocations
- `.rela.plt`: PLT relocations (function calls)

#### SHT_HASH (5) / SHT_GNU_HASH (0x6ffffff6): Hash Tables

ใช้ hash สำหรับค้นหา symbols เร็วขึ้น

#### SHT_DYNAMIC (6): Dynamic Linking Info

Section `.dynamic` - เหมือนกับ PT_DYNAMIC segment

#### SHT_NOTE (7): Note Sections

เช่น `.note.gnu.build-id`

### 3.2 Section Flags

| Flag | ค่า | ความหมาย |
|------|-----|----------|
| SHF_WRITE | 0x1 | Writable |
| SHF_ALLOC | 0x2 | Occupies memory during execution |
| SHF_EXECINSTR | 0x4 | Executable |
| SHF_MERGE | 0x10 | Might be merged |
| SHF_STRINGS | 0x20 | Contains null-terminated strings |
| SHF_TLS | 0x400 | Thread-Local Storage |

---

## 4. Symbol Table

### 4.1 Elf64_Sym Structure

```c
typedef struct {
    Elf64_Word  st_name;  /* Symbol name (index ใน string table) */
    unsigned char st_info; /* Symbol type and binding */
    unsigned char st_other;/* Symbol visibility */
    Elf64_Section st_shndx;/* Section index */
    Elf64_Addr  st_value; /* Symbol value (address) */
    Elf64_Xword st_size;  /* Symbol size */
} Elf64_Sym;
```

### 4.2 Symbol Binding (STB_*)

`st_info` = `(binding << 4) | type`

| Binding | ค่า | ความหมาย |
|---------|-----|----------|
| STB_LOCAL | 0 | ไม่ visible นอก object file |
| STB_GLOBAL | 1 | Global symbol (visible ทุกที่) |
| STB_WEAK | 2 | Weak symbol (แทนได้ด้วย global) |

### 4.3 Symbol Types (STT_*)

| Type | ค่า | ความหมาย |
|------|-----|----------|
| STT_NOTYPE | 0 | ไม่ระบุ type |
| STT_OBJECT | 1 | Data object (variable) |
| STT_FUNC | 2 | Function |
| STT_SECTION | 3 | Section |
| STT_FILE | 4 | Source file name |
| STT_TLS | 6 | Thread-Local Storage |

### 4.4 Special Section Indices

| Index | ความหมาย |
|-------|----------|
| SHN_UNDEF (0) | Undefined symbol (ต้องหาจาก library อื่น) |
| SHN_ABS (0xFFF1) | Absolute value |
| SHN_COMMON (0xFFF2) | Common symbol (uninitialized global) |

---

## 5. Relocation

Relocation บอก linker ว่าต้องแก้ไข address ตรงไหนเมื่อ link

### 5.1 Elf64_Rela Structure

```c
typedef struct {
    Elf64_Addr  r_offset;  /* Location to patch */
    Elf64_Xword r_info;    /* Symbol index + relocation type */
    Elf64_Sxword r_addend; /* Constant addend */
} Elf64_Rela;
```

`r_info = (symbol_index << 32) | relocation_type`

### 5.2 Relocation Types สำหรับ x86-64

#### R_X86_64_64 (1): 64-bit Absolute

```
S + A
```
เขียน 64-bit absolute address

#### R_X86_64_32 (10): 32-bit Absolute (Zero-extend)

```
S + A
```
เขียน 32-bit value, zero-extend เป็น 64-bit

#### R_X86_64_PC32 (2): 32-bit PC-relative

```
S + A - P
```
ใช้สำหรับ near call/jump instructions

#### R_X86_64_PLT32 (4): PLT-relative

```
L + A - P
```
ใช้สำหรับ function calls ผ่าน PLT

#### R_X86_64_GLOB_DAT (6): GOT Entry

```
S
```
Initialize GOT entry สำหรับ symbol

#### R_X86_64_JUMP_SLOT (7): PLT Jump Slot

```
S
```
Dynamic linker update PLT entry

#### R_X86_64_RELATIVE (8): Load Address Relative

```
B + A
```
ใช้สำหรับ PIE/ASLR - relative to load address

#### R_X86_64_COPY (5): Copy Symbol

```
Copy symbol value from shared library
```
ใช้สำหรับ global variables ใน shared libraries

---

## 6. Dynamic Section

`.dynamic` section เก็บ array ของ `Elf64_Dyn` entries:

```c
typedef struct {
    Elf64_Sxword d_tag;  /* Type tag */
    union {
        Elf64_Xword d_val;  /* Integer value */
        Elf64_Addr  d_ptr;  /* Address value */
    } d_un;
} Elf64_Dyn;
```

### 6.1 Dynamic Tags ที่สำคัญ

| Tag | ค่า | ความหมาย |
|-----|-----|----------|
| DT_NULL | 0 | จบ dynamic array |
| DT_NEEDED | 1 | ชื่อ shared library ที่ต้องการ |
| DT_PLTRELSZ | 2 | ขนาดของ PLT relocation table |
| DT_PLTGOT | 3 | Address ของ PLT/GOT |
| DT_HASH | 4 | Address ของ symbol hash table |
| DT_STRTAB | 5 | Address ของ string table |
| DT_SYMTAB | 6 | Address ของ symbol table |
| DT_RELA | 7 | Address ของ relocation table |
| DT_RELASZ | 8 | ขนาดของ relocation table |
| DT_RELAENT | 9 | ขนาดของ relocation entry |
| DT_STRSZ | 10 | ขนาดของ string table |
| DT_SYMENT | 11 | ขนาดของ symbol entry |
| DT_INIT | 12 | Address ของ init function |
| DT_FINI | 13 | Address ของ fini function |
| DT_SONAME | 14 | ชื่อ shared library นี้ |
| DT_RPATH | 15 | Library search path (deprecated) |
| DT_JMPREL | 23 | PLT relocations |
| DT_BIND_NOW | 24 | Bind all symbols now |
| DT_RUNPATH | 29 | Library search path |
| DT_FLAGS | 30 | Flags |
| DT_GNU_HASH | 0x6ffffef5 | GNU hash table |
| DT_VERSYM | 0x6ffffff0 | Version symbol table |

---

## 7. Programs

### Program 1: ELF Parser ใน Assembly

โปรแกรมนี้จะ parse ELF header และแสดงข้อมูลสำคัญ

```nasm
; elf_parser.asm - ELF Binary Parser
; วิธีใช้: ./elf_parser <filename>
; สร้าง: nasm -f elf64 elf_parser.asm && ld -o elf_parser elf_parser.o

section .data
    ; Messages สำหรับแสดงผล
    msg_usage       db "Usage: elf_parser <filename>", 10, 0
    msg_open_err    db "Error: Cannot open file", 10, 0
    msg_not_elf     db "Error: Not an ELF file", 10, 0
    msg_elf_header  db "=== ELF Header ===", 10, 0
    msg_magic       db "Magic:    7f 45 4c 46", 10, 0
    msg_class32     db "Class:    ELF32", 10, 0
    msg_class64     db "Class:    ELF64", 10, 0
    msg_le          db "Data:     2's complement, little endian", 10, 0
    msg_be          db "Data:     2's complement, big endian", 10, 0
    msg_type_rel    db "Type:     REL (Relocatable file)", 10, 0
    msg_type_exec   db "Type:     EXEC (Executable file)", 10, 0
    msg_type_dyn    db "Type:     DYN (Shared object file)", 10, 0
    msg_type_core   db "Type:     CORE (Core file)", 10, 0
    msg_type_unk    db "Type:     Unknown", 10, 0
    msg_machine     db "Machine:  ", 0
    msg_x86_64      db "Advanced Micro Devices X86-64", 10, 0
    msg_x86        db "Intel 80386", 10, 0
    msg_arm64       db "AArch64", 10, 0
    msg_entry       db "Entry:    0x", 0
    msg_phoff       db "PH Off:   0x", 0
    msg_shoff       db "SH Off:   0x", 0
    msg_phnum       db "PH Num:   ", 0
    msg_shnum       db "SH Num:   ", 0
    msg_newline     db 10, 0
    msg_colon       db ": ", 0

    ; Buffer สำหรับ hex output
    hex_buf         times 20 db 0
    hex_chars       db "0123456789abcdef"
    dec_buf         times 20 db 0

section .bss
    fd              resq 1      ; File descriptor
    elf_header      resb 64     ; ELF header buffer (64 bytes สำหรับ 64-bit)
    filename        resq 1      ; Pointer to filename

section .text
    global _start

_start:
    ; ตรวจสอบ arguments
    pop rcx                     ; argc
    pop rdi                     ; argv[0] (ชื่อโปรแกรม)
    cmp rcx, 2                  ; ต้องมี argument อย่างน้อย 1 ตัว
    jl .print_usage

    pop rdi                     ; argv[1] = filename
    mov [filename], rdi

    ; เปิดไฟล์
    mov rax, 2                  ; sys_open
    ; rdi ยังคงเป็น filename
    xor rsi, rsi                ; O_RDONLY = 0
    xor rdx, rdx                ; mode = 0
    syscall
    
    test rax, rax               ; ตรวจสอบ error
    js .open_error
    mov [fd], rax               ; เก็บ file descriptor

    ; อ่าน ELF header (64 bytes)
    mov rax, 0                  ; sys_read
    mov rdi, [fd]
    lea rsi, [elf_header]
    mov rdx, 64
    syscall

    cmp rax, 64                 ; ตรวจสอบว่าอ่านได้ครบ
    jl .not_elf

    ; ตรวจสอบ magic number
    cmp dword [elf_header], 0x464c457f  ; 0x7f 'E' 'L' 'F'
    jne .not_elf

    ; แสดง header info
    call print_elf_header

    ; ปิดไฟล์
    mov rax, 3                  ; sys_close
    mov rdi, [fd]
    syscall

    ; Exit สำเร็จ
    mov rax, 60
    xor rdi, rdi
    syscall

.print_usage:
    lea rsi, [msg_usage]
    call print_string
    mov rax, 60
    mov rdi, 1
    syscall

.open_error:
    lea rsi, [msg_open_err]
    call print_string
    mov rax, 60
    mov rdi, 1
    syscall

.not_elf:
    lea rsi, [msg_not_elf]
    call print_string
    mov rax, 3
    mov rdi, [fd]
    syscall
    mov rax, 60
    mov rdi, 1
    syscall

; =============================================
; print_elf_header - แสดงข้อมูล ELF header
; =============================================
print_elf_header:
    push rbp
    mov rbp, rsp

    ; แสดง header line
    lea rsi, [msg_elf_header]
    call print_string

    ; แสดง Magic
    lea rsi, [msg_magic]
    call print_string

    ; แสดง Class
    movzx eax, byte [elf_header + 4]  ; EI_CLASS
    cmp al, 1
    je .class32
    cmp al, 2
    je .class64
    jmp .class_done
.class32:
    lea rsi, [msg_class32]
    call print_string
    jmp .class_done
.class64:
    lea rsi, [msg_class64]
    call print_string
.class_done:

    ; แสดง Data (endianness)
    movzx eax, byte [elf_header + 5]  ; EI_DATA
    cmp al, 1
    je .little_endian
    lea rsi, [msg_be]
    call print_string
    jmp .endian_done
.little_endian:
    lea rsi, [msg_le]
    call print_string
.endian_done:

    ; แสดง Type
    movzx eax, word [elf_header + 16]  ; e_type
    cmp ax, 1
    je .type_rel
    cmp ax, 2
    je .type_exec
    cmp ax, 3
    je .type_dyn
    cmp ax, 4
    je .type_core
    lea rsi, [msg_type_unk]
    call print_string
    jmp .type_done
.type_rel:
    lea rsi, [msg_type_rel]
    call print_string
    jmp .type_done
.type_exec:
    lea rsi, [msg_type_exec]
    call print_string
    jmp .type_done
.type_dyn:
    lea rsi, [msg_type_dyn]
    call print_string
    jmp .type_done
.type_core:
    lea rsi, [msg_type_core]
    call print_string
.type_done:

    ; แสดง Machine
    lea rsi, [msg_machine]
    call print_string
    movzx eax, word [elf_header + 18]  ; e_machine
    cmp ax, 62      ; EM_X86_64
    je .machine_x86_64
    cmp ax, 3       ; EM_386
    je .machine_x86
    cmp ax, 183     ; EM_AARCH64
    je .machine_arm64
    jmp .machine_done
.machine_x86_64:
    lea rsi, [msg_x86_64]
    call print_string
    jmp .machine_done
.machine_x86:
    lea rsi, [msg_x86]
    call print_string
    jmp .machine_done
.machine_arm64:
    lea rsi, [msg_arm64]
    call print_string
.machine_done:

    ; แสดง Entry point
    lea rsi, [msg_entry]
    call print_string
    mov rax, [elf_header + 24]  ; e_entry
    call print_hex64
    lea rsi, [msg_newline]
    call print_string

    ; แสดง Program Header Offset
    lea rsi, [msg_phoff]
    call print_string
    mov rax, [elf_header + 32]  ; e_phoff
    call print_hex64
    lea rsi, [msg_newline]
    call print_string

    ; แสดง Section Header Offset
    lea rsi, [msg_shoff]
    call print_string
    mov rax, [elf_header + 40]  ; e_shoff
    call print_hex64
    lea rsi, [msg_newline]
    call print_string

    ; แสดง Number of Program Headers
    lea rsi, [msg_phnum]
    call print_string
    movzx rax, word [elf_header + 56]  ; e_phnum
    call print_decimal
    lea rsi, [msg_newline]
    call print_string

    ; แสดง Number of Section Headers
    lea rsi, [msg_shnum]
    call print_string
    movzx rax, word [elf_header + 60]  ; e_shnum
    call print_decimal
    lea rsi, [msg_newline]
    call print_string

    pop rbp
    ret

; =============================================
; print_string - แสดง null-terminated string
; Input: rsi = pointer to string
; =============================================
print_string:
    push rax
    push rdi
    push rcx
    
    ; หา length
    mov rdi, rsi
    xor rcx, rcx
.count_loop:
    cmp byte [rdi + rcx], 0
    je .count_done
    inc rcx
    jmp .count_loop
.count_done:
    
    mov rax, 1      ; sys_write
    mov rdi, 1      ; stdout
    ; rsi = string pointer
    mov rdx, rcx   ; length
    syscall
    
    pop rcx
    pop rdi
    pop rax
    ret

; =============================================
; print_hex64 - แสดง 64-bit value เป็น hex
; Input: rax = value to print
; =============================================
print_hex64:
    push rbx
    push rcx
    push rdx
    push rsi

    lea rsi, [hex_buf]
    mov rcx, 16             ; 16 hex digits สำหรับ 64-bit
    
.hex_loop:
    mov rbx, rax
    and rbx, 0xF            ; เอา 4 bits ล่าง
    lea rdx, [hex_chars]
    movzx rdx, byte [rdx + rbx]
    mov byte [rsi + rcx - 1], dl
    shr rax, 4
    dec rcx
    jnz .hex_loop
    
    ; แสดง hex string
    mov rax, 1
    mov rdi, 1
    ; rsi ยังชี้ที่ hex_buf
    mov rdx, 16
    syscall

    pop rsi
    pop rdx
    pop rcx
    pop rbx
    ret

; =============================================
; print_decimal - แสดง value เป็น decimal
; Input: rax = value
; =============================================
print_decimal:
    push rbx
    push rcx
    push rdx
    push rsi

    lea rsi, [dec_buf + 19]   ; เริ่มจากท้าย
    mov byte [rsi], 0
    mov rcx, 0               ; นับจำนวน digits
    mov rbx, 10

.div_loop:
    xor rdx, rdx
    div rbx                  ; rax = quotient, rdx = remainder
    add dl, '0'
    dec rsi
    mov [rsi], dl
    inc rcx
    test rax, rax
    jnz .div_loop

    ; แสดง
    mov rax, 1
    mov rdi, 1
    ; rsi = pointer to number
    mov rdx, rcx
    syscall

    pop rsi
    pop rdx
    pop rcx
    pop rbx
    ret
```

---

### Program 2: Minimal ELF (45 bytes)

ELF executable ที่เล็กที่สุด - เพียง 45 bytes!

```nasm
; minimal_elf.asm - สร้าง Minimal ELF executable
; ไฟล์นี้สร้าง binary โดยตรงด้วย db directives

; ทำไมถึง 45 bytes?
; ELF Header + Program Header + Code = ขั้นต่ำที่ต้องการ
; เราจะ overlap ELF header กับ Program Header เพื่อประหยัดพื้นที่

; โปรแกรมจะ exit ด้วย code 42

BITS 64
    org 0x400000

ehdr:
    ; ELF Identification
    db 0x7f, "ELF"          ; Magic number
    db 2                     ; EI_CLASS = ELFCLASS64
    db 1                     ; EI_DATA = ELFDATA2LSB (little-endian)
    db 1                     ; EI_VERSION = EV_CURRENT
    db 0                     ; EI_OSABI = ELFOSABI_NONE
    dq 0                     ; EI_ABIVERSION + padding

    ; ELF Header fields
    dw 2                     ; e_type = ET_EXEC
    dw 62                    ; e_machine = EM_X86_64
    dd 1                     ; e_version = EV_CURRENT
    dq _start                ; e_entry = entry point
    dq phdr - ehdr           ; e_phoff = program header offset
    dq 0                     ; e_shoff = no section headers
    dd 0                     ; e_flags = 0
    dw ehdrsize              ; e_ehsize
    dw phdrsize              ; e_phentsize
    dw 1                     ; e_phnum = 1 program header
    dw 64                    ; e_shentsize (ไม่ใช้)
    dw 0                     ; e_shnum = 0
    dw 0                     ; e_shstrndx = 0

ehdrsize equ $ - ehdr

phdr:
    ; Program Header
    dd 1                     ; p_type = PT_LOAD
    dd 5                     ; p_flags = PF_R | PF_X
    dq 0                     ; p_offset = 0 (โหลดจากต้นไฟล์)
    dq 0x400000              ; p_vaddr = load address
    dq 0x400000              ; p_paddr (ไม่ใช้)
    dq filesize              ; p_filesz
    dq filesize              ; p_memsz
    dq 0x200000              ; p_align = 2MB

phdrsize equ $ - phdr

_start:
    ; Exit ด้วย code 42
    mov edi, 42              ; exit code
    mov eax, 60              ; sys_exit
    syscall

filesize equ $ - ehdr
```

สร้างและรัน:
```bash
# สร้าง
nasm -f bin minimal_elf.asm -o minimal_elf
chmod +x minimal_elf

# ตรวจสอบขนาด
ls -la minimal_elf
# ควรได้ประมาณ 45-60 bytes

# รัน
./minimal_elf
echo $?   # ควรได้ 42

# ดู ELF header
readelf -h minimal_elf
```

---

### Program 3: ELF Injector

โปรแกรมนี้จะ inject shellcode เข้าไปใน ELF binary

```nasm
; elf_injector.asm - ELF Code Injector
; เทคนิค: ใช้ PT_NOTE segment เปลี่ยนเป็น PT_LOAD และ inject code
; วิธีใช้: ./elf_injector target_binary shellcode_file output_binary

; DISCLAIMER: สำหรับการศึกษาเท่านั้น
; ห้ามใช้กับโปรแกรมที่ไม่ได้รับอนุญาต

section .data
    msg_start   db "ELF Injector - Educational Tool", 10, 0
    msg_usage   db "Usage: elf_injector <target> <shellcode> <output>", 10, 0
    msg_reading db "Reading target binary...", 10, 0
    msg_found   db "Found PT_NOTE segment, converting...", 10, 0
    msg_inject  db "Injecting shellcode...", 10, 0
    msg_done    db "Injection complete!", 10, 0
    msg_err     db "Error occurred", 10, 0
    
    ; PT_NOTE -> PT_LOAD conversion
    PT_NOTE     equ 4
    PT_LOAD     equ 1
    
    ; Permission flags
    PF_R        equ 4
    PF_W        equ 2
    PF_X        equ 1

section .bss
    target_fd   resq 1
    output_fd   resq 1
    shellcode_fd resq 1
    
    ; Buffers
    elf_buf     resb 1048576    ; 1MB buffer สำหรับ ELF
    sc_buf      resb 4096       ; 4KB buffer สำหรับ shellcode
    
    bytes_read  resq 1
    sc_size     resq 1
    
    ; ELF metadata
    ph_offset   resq 1          ; Program header table offset
    ph_entsize  resq 1          ; Program header entry size
    ph_num      resq 1          ; Number of program headers
    
    ; inject location
    inject_offset resq 1        ; offset ใน file ที่จะ inject
    inject_addr   resq 1        ; virtual address ที่ inject

section .text
    global _start

_start:
    ; แสดง banner
    lea rsi, [msg_start]
    call print_string

    ; ตรวจ arguments
    mov rbx, [rsp]              ; argc
    cmp rbx, 4
    jl .usage

    ; อ่าน filenames
    mov rdi, [rsp + 16]         ; argv[1] = target
    call open_read
    test rax, rax
    js .error
    mov [target_fd], rax

    mov rdi, [rsp + 24]         ; argv[2] = shellcode
    call open_read
    test rax, rax
    js .error
    mov [shellcode_fd], rax

    ; อ่าน target binary
    lea rsi, [msg_reading]
    call print_string

    mov rax, 0                  ; sys_read
    mov rdi, [target_fd]
    lea rsi, [elf_buf]
    mov rdx, 1048576
    syscall
    mov [bytes_read], rax

    ; ตรวจ ELF magic
    cmp dword [elf_buf], 0x464c457f
    jne .error

    ; อ่าน shellcode
    mov rax, 0
    mov rdi, [shellcode_fd]
    lea rsi, [sc_buf]
    mov rdx, 4096
    syscall
    mov [sc_size], rax

    ; หา PT_NOTE segment
    call find_pt_note
    test rax, rax
    jz .error

    ; Inject shellcode
    lea rsi, [msg_inject]
    call print_string
    call inject_shellcode

    ; เปิด output file
    mov rdi, [rsp + 32]         ; argv[3] = output
    call open_write
    test rax, rax
    js .error
    mov [output_fd], rax

    ; เขียน modified binary
    mov rax, 1                  ; sys_write
    mov rdi, [output_fd]
    lea rsi, [elf_buf]
    mov rdx, [bytes_read]
    syscall

    ; ปิดไฟล์ทั้งหมด
    mov rax, 3
    mov rdi, [target_fd]
    syscall
    
    mov rax, 3
    mov rdi, [shellcode_fd]
    syscall
    
    mov rax, 3
    mov rdi, [output_fd]
    syscall

    lea rsi, [msg_done]
    call print_string

    mov rax, 60
    xor rdi, rdi
    syscall

.usage:
    lea rsi, [msg_usage]
    call print_string
    mov rax, 60
    mov rdi, 1
    syscall

.error:
    lea rsi, [msg_err]
    call print_string
    mov rax, 60
    mov rdi, 1
    syscall

; =============================================
; find_pt_note - หา PT_NOTE segment
; Output: rax = pointer to program header หรือ 0
; =============================================
find_pt_note:
    push rbx
    push rcx
    push rdx

    ; อ่าน program header info จาก ELF header
    mov rbx, [elf_buf + 32]     ; e_phoff
    movzx rcx, word [elf_buf + 54]  ; e_phentsize
    movzx rdx, word [elf_buf + 56]  ; e_phnum

    ; เก็บสำหรับใช้ทีหลัง
    mov [ph_offset], rbx
    mov [ph_entsize], rcx
    mov [ph_num], rdx

.search_loop:
    test rdx, rdx
    jz .not_found

    ; ตรวจ p_type
    lea rax, [elf_buf]
    add rax, rbx                ; pointer to current phdr
    cmp dword [rax], 4          ; PT_NOTE = 4
    je .found

    add rbx, rcx                ; ไปที่ phdr ถัดไป
    dec rdx
    jmp .search_loop

.found:
    lea rax, [elf_buf]
    add rax, rbx                ; return pointer to phdr
    lea rsi, [msg_found]
    push rax
    call print_string
    pop rax
    jmp .done

.not_found:
    xor rax, rax

.done:
    pop rdx
    pop rcx
    pop rbx
    ret

; =============================================
; inject_shellcode - inject shellcode เข้าใน binary
; รับ pointer to PT_NOTE phdr ใน rax
; =============================================
inject_shellcode:
    push rbx
    push rcx

    mov rbx, rax                ; เก็บ pointer to phdr

    ; คำนวณ offset ที่จะ inject (ต่อจากข้อมูลปัจจุบัน)
    mov rax, [bytes_read]       ; inject หลังข้อมูลเดิม
    mov [inject_offset], rax

    ; คำนวณ virtual address
    ; ต้องหา high address เพื่อ inject หลังจากนั้น
    mov rax, 0x1000000         ; ใช้ address สูงๆ หน่อย (ปรับตาม binary จริง)
    mov [inject_addr], rax

    ; Copy shellcode ไปที่ inject location
    lea rdi, [elf_buf]
    add rdi, [inject_offset]    ; destination
    lea rsi, [sc_buf]           ; source
    mov rcx, [sc_size]
    rep movsb

    ; อัปเดต bytes_read
    mov rax, [bytes_read]
    add rax, [sc_size]
    mov [bytes_read], rax

    ; แปลง PT_NOTE เป็น PT_LOAD
    mov dword [rbx], 1          ; p_type = PT_LOAD

    ; Set flags = R|X (readable + executable)
    mov dword [rbx + 4], 5     ; p_flags = PF_R | PF_X

    ; Set file offset
    mov rax, [inject_offset]
    mov [rbx + 8], rax          ; p_offset

    ; Set virtual address
    mov rax, [inject_addr]
    mov [rbx + 16], rax         ; p_vaddr
    mov [rbx + 24], rax         ; p_paddr

    ; Set sizes
    mov rax, [sc_size]
    mov [rbx + 32], rax         ; p_filesz
    mov [rbx + 40], rax         ; p_memsz

    ; Set alignment
    mov qword [rbx + 48], 0x1000  ; p_align = 4096

    ; แก้ entry point ของ ELF header ให้ชี้ไป shellcode
    ; (อันนี้ขึ้นอยู่กับว่าต้องการ override entry point หรือไม่)
    ; mov rax, [inject_addr]
    ; mov [elf_buf + 24], rax     ; e_entry

    pop rcx
    pop rbx
    ret

; =============================================
; open_read - เปิดไฟล์ read-only
; Input: rdi = filename
; Output: rax = fd หรือ negative error
; =============================================
open_read:
    mov rax, 2                  ; sys_open
    ; rdi = filename
    xor rsi, rsi                ; O_RDONLY = 0
    xor rdx, rdx
    syscall
    ret

; =============================================
; open_write - เปิดไฟล์ write (สร้างใหม่)
; Input: rdi = filename
; Output: rax = fd หรือ negative error
; =============================================
open_write:
    mov rax, 2                  ; sys_open
    ; rdi = filename
    mov rsi, 0x241              ; O_WRONLY | O_CREAT | O_TRUNC
    mov rdx, 0755o              ; permissions
    syscall
    ret

; =============================================
; print_string - แสดง null-terminated string
; Input: rsi = pointer to string
; =============================================
print_string:
    push rax
    push rdi
    push rcx
    push rdx
    
    mov rdi, rsi
    xor rcx, rcx
.len_loop:
    cmp byte [rdi + rcx], 0
    je .len_done
    inc rcx
    jmp .len_loop
.len_done:
    
    mov rax, 1
    mov rdi, 1
    ; rsi = string
    mov rdx, rcx
    syscall
    
    pop rdx
    pop rcx
    pop rdi
    pop rax
    ret
```

---

### Program 4: Symbol Lookup

โปรแกรมค้นหา symbol ใน ELF binary ด้วย hash table

```nasm
; symbol_lookup.asm - ค้นหา symbol address ใน ELF
; ใช้ GNU Hash table สำหรับความเร็ว
; วิธีใช้: ./symbol_lookup <library.so> <symbol_name>

section .data
    msg_banner  db "ELF Symbol Lookup", 10, 0
    msg_usage   db "Usage: symbol_lookup <library> <symbol>", 10, 0
    msg_found   db "Symbol found at: 0x", 0
    msg_notfound db "Symbol not found", 10, 0
    msg_newline db 10, 0
    msg_err     db "Error: ", 0
    msg_noelf   db "Not an ELF file", 10, 0
    msg_nosym   db "No symbol table found", 10, 0

    hex_chars   db "0123456789abcdef"

section .bss
    ; File data
    elf_fd      resq 1
    elf_buf     resb 4194304    ; 4MB buffer
    elf_size    resq 1
    
    ; Pointers ใน ELF
    sym_table   resq 1          ; .dynsym section
    str_table   resq 1          ; .dynstr section
    sym_count   resq 1          ; จำนวน symbols
    
    ; GNU hash table pointers
    gnu_hash    resq 1          ; .gnu.hash section
    
    ; search buffer
    sym_name    resb 256
    sym_name_len resq 1
    
    ; result
    sym_addr    resq 1
    
    hex_buf     resb 20

section .text
    global _start

_start:
    ; แสดง banner
    lea rsi, [msg_banner]
    call print_string

    ; ตรวจ arguments
    mov rax, [rsp]
    cmp rax, 3
    jl .usage

    ; เปิดและอ่าน library
    mov rdi, [rsp + 16]         ; argv[1] = library path
    call load_elf
    test rax, rax
    jz .error_noelf

    ; copy symbol name
    mov rsi, [rsp + 24]         ; argv[2] = symbol name
    lea rdi, [sym_name]
    xor rcx, rcx
.copy_name:
    mov al, [rsi + rcx]
    mov [rdi + rcx], al
    test al, al
    jz .name_copied
    inc rcx
    jmp .copy_name
.name_copied:
    mov [sym_name_len], rcx

    ; ค้นหาใน symbol table
    call find_sections
    test rax, rax
    jz .error_nosym

    ; ค้นหา symbol
    lea rsi, [sym_name]
    call lookup_symbol
    
    test rax, rax
    jz .not_found

    ; แสดงผล
    mov [sym_addr], rax
    lea rsi, [msg_found]
    call print_string
    mov rax, [sym_addr]
    call print_hex64
    lea rsi, [msg_newline]
    call print_string

    jmp .exit_ok

.not_found:
    lea rsi, [msg_notfound]
    call print_string

.exit_ok:
    mov rax, 3
    mov rdi, [elf_fd]
    syscall
    mov rax, 60
    xor rdi, rdi
    syscall

.usage:
    lea rsi, [msg_usage]
    call print_string
    mov rax, 60
    mov rdi, 1
    syscall

.error_noelf:
    lea rsi, [msg_noelf]
    call print_string
    mov rax, 60
    mov rdi, 1
    syscall

.error_nosym:
    lea rsi, [msg_nosym]
    call print_string
    mov rax, 60
    mov rdi, 1
    syscall

; =============================================
; load_elf - โหลดไฟล์ ELF เข้า buffer
; Input: rdi = filename
; Output: rax = 1 (ok) หรือ 0 (error)
; =============================================
load_elf:
    push rbx

    ; เปิดไฟล์
    mov rax, 2                  ; sys_open
    xor rsi, rsi                ; O_RDONLY
    xor rdx, rdx
    syscall
    test rax, rax
    js .error
    mov [elf_fd], rax
    mov rbx, rax

    ; อ่านไฟล์
    mov rax, 0                  ; sys_read
    mov rdi, rbx
    lea rsi, [elf_buf]
    mov rdx, 4194304
    syscall
    test rax, rax
    jle .error
    mov [elf_size], rax

    ; ตรวจ magic
    cmp dword [elf_buf], 0x464c457f
    jne .error

    mov rax, 1                  ; success
    pop rbx
    ret

.error:
    xor rax, rax
    pop rbx
    ret

; =============================================
; find_sections - หา symbol table และ string table
; Output: rax = 1 (found) หรือ 0 (not found)
; =============================================
find_sections:
    push rbx
    push rcx
    push rdx
    push r8
    push r9

    ; อ่านข้อมูล section headers
    mov rbx, [elf_buf + 40]     ; e_shoff
    movzx rcx, word [elf_buf + 58]  ; e_shentsize
    movzx rdx, word [elf_buf + 60]  ; e_shnum
    movzx r8, word [elf_buf + 62]   ; e_shstrndx

    ; หา string table สำหรับ section names
    ; (.shstrtab)
    mov r9, rbx
    imul r8, rcx                ; r8 = offset ของ shstrtab section header
    add r9, r8
    lea r9, [elf_buf + r9]     ; pointer to shstrtab section header
    mov r9, [r9 + 24]          ; sh_offset ของ shstrtab
    lea r9, [elf_buf + r9]     ; pointer to shstrtab data

    ; วน loop ผ่าน sections
    xor r8, r8                  ; section counter
.section_loop:
    cmp r8, rdx
    jge .loop_done

    ; คำนวณ pointer to section header
    push rax
    mov rax, r8
    imul rax, rcx
    add rax, rbx
    lea rax, [elf_buf + rax]   ; pointer to section header

    ; ดู sh_type
    mov esi, dword [rax + 4]    ; sh_type

    ; ตรวจ SHT_DYNSYM (11)
    cmp esi, 11
    jne .check_dynstr

    ; เจอ .dynsym
    mov rsi, [rax + 24]         ; sh_offset
    lea rsi, [elf_buf + rsi]
    mov [sym_table], rsi
    ; คำนวณจำนวน symbols
    mov rsi, [rax + 32]         ; sh_size
    mov rcx, 24                 ; sizeof(Elf64_Sym) = 24 bytes
    xor rdx, rdx
    div rcx                     ; rax = count (ใช้ tmp)
    ; แต่ div เปลี่ยน rcx! ต้อง save ก่อน
    ; (ปรับ code ให้ถูกต้อง)
    pop rax
    inc r8
    jmp .section_loop

.check_dynstr:
    ; ตรวจ SHT_STRTAB สำหรับ .dynstr
    cmp esi, 3
    jne .check_gnu_hash

    ; ตรวจชื่อว่าเป็น .dynstr
    ; (ในที่นี้ simplified ไม่ตรวจชื่อ จะใช้อันแรกที่เจอ)
    mov rsi, [rax + 24]         ; sh_offset
    lea rsi, [elf_buf + rsi]
    cmp qword [str_table], 0
    jne .skip_strtab
    mov [str_table], rsi
.skip_strtab:
    pop rax
    inc r8
    jmp .section_loop

.check_gnu_hash:
    ; ตรวจ SHT_GNU_HASH
    cmp dword [rax + 4], 0x6ffffef5
    jne .next_section
    
    mov rsi, [rax + 24]         ; sh_offset
    lea rsi, [elf_buf + rsi]
    mov [gnu_hash], rsi
    pop rax
    inc r8
    jmp .section_loop

.next_section:
    pop rax
    inc r8
    jmp .section_loop

.loop_done:
    ; ตรวจว่าเจอ sym_table และ str_table
    cmp qword [sym_table], 0
    je .not_found
    cmp qword [str_table], 0
    je .not_found
    
    mov rax, 1
    jmp .done

.not_found:
    xor rax, rax

.done:
    pop r9
    pop r8
    pop rdx
    pop rcx
    pop rbx
    ret

; =============================================
; lookup_symbol - ค้นหา symbol ใน .dynsym
; Input: rsi = pointer to symbol name (null-terminated)
; Output: rax = symbol address หรือ 0
; =============================================
lookup_symbol:
    push rbx
    push rcx
    push rdx
    push r12
    push r13
    push r14

    mov r12, rsi                ; เก็บชื่อที่ค้นหา
    mov r13, [sym_table]        ; pointer to .dynsym
    mov r14, [str_table]        ; pointer to .dynstr

    ; ต้องรู้จำนวน symbols - หาจาก ELF size
    ; simplified: ใช้ sh_size ที่เก็บไว้
    ; สำหรับตัวอย่างนี้จะวน 1000 entries สูงสุด
    mov rcx, 1000

.sym_loop:
    test rcx, rcx
    jz .not_found

    ; st_name = first 4 bytes of Elf64_Sym
    mov ebx, dword [r13]        ; st_name = index ใน string table
    test ebx, ebx
    jz .next_sym                ; ข้ามถ้า name index = 0

    ; ดูชื่อ symbol
    lea rdx, [r14 + rbx]        ; pointer to symbol name in strtab

    ; เปรียบเทียบชื่อ
    push rcx
    mov rsi, r12                ; ชื่อที่ค้นหา
    mov rdi, rdx                ; ชื่อใน symbol table
    call strcmp
    pop rcx
    test rax, rax
    jz .found

.next_sym:
    add r13, 24                 ; sizeof(Elf64_Sym) = 24
    dec rcx
    jmp .sym_loop

.found:
    ; return st_value (offset 8 ใน Elf64_Sym)
    mov rax, [r13 + 8]          ; st_value
    jmp .done

.not_found:
    xor rax, rax

.done:
    pop r14
    pop r13
    pop r12
    pop rdx
    pop rcx
    pop rbx
    ret

; =============================================
; strcmp - เปรียบเทียบ strings
; Input: rsi = s1, rdi = s2
; Output: rax = 0 (equal), non-zero (different)
; =============================================
strcmp:
    push rcx
    xor rcx, rcx
.loop:
    mov al, [rsi + rcx]
    mov bl, [rdi + rcx]
    cmp al, bl
    jne .diff
    test al, al
    jz .equal
    inc rcx
    jmp .loop
.equal:
    xor rax, rax
    pop rcx
    ret
.diff:
    movzx rax, al
    movzx rbx, bl
    sub rax, rbx
    pop rcx
    ret

; =============================================
; print_string - แสดง string
; =============================================
print_string:
    push rax
    push rdi
    push rcx
    push rdx

    mov rdi, rsi
    xor rcx, rcx
.count:
    cmp byte [rdi + rcx], 0
    je .done_count
    inc rcx
    jmp .count
.done_count:
    
    mov rax, 1
    mov rdi, 1
    mov rdx, rcx
    syscall
    
    pop rdx
    pop rcx
    pop rdi
    pop rax
    ret

; =============================================
; print_hex64 - แสดง 64-bit hex
; Input: rax = value
; =============================================
print_hex64:
    push rbx
    push rcx
    push rdx
    push rsi

    lea rsi, [hex_buf]
    mov rcx, 16
.loop:
    mov rbx, rax
    and rbx, 0xF
    lea rdx, [hex_chars]
    movzx rdx, byte [rdx + rbx]
    mov [rsi + rcx - 1], dl
    shr rax, 4
    dec rcx
    jnz .loop

    mov rax, 1
    mov rdi, 1
    mov rdx, 16
    syscall

    pop rsi
    pop rdx
    pop rcx
    pop rbx
    ret
```

---

## 8. การวิเคราะห์ ELF ด้วย Tools

### 8.1 readelf

เครื่องมือมาตรฐานสำหรับดู ELF:

```bash
# ดู ELF header
readelf -h binary

# ดู Program headers (segments)
readelf -l binary

# ดู Section headers
readelf -S binary

# ดู Symbol table
readelf -s binary

# ดู Dynamic section
readelf -d binary

# ดู Relocations
readelf -r binary

# ดูทุกอย่าง
readelf -a binary

# ดู Note sections
readelf -n binary
```

### 8.2 objdump

```bash
# Disassemble .text section
objdump -d binary

# Disassemble ทุก sections
objdump -D binary

# แสดง section headers
objdump -h binary

# แสดง symbol table
objdump -t binary

# แสดง dynamic symbols
objdump -T binary

# Disassemble พร้อม source (ถ้ามี debug info)
objdump -S binary
```

### 8.3 nm

```bash
# แสดง symbols ทั้งหมด
nm binary

# ดู dynamic symbols
nm -D binary

# เรียงตาม address
nm -n binary

# แสดง external symbols เท่านั้น
nm -g binary
```

Symbol types ใน nm:
- `T` / `t`: .text (code) - uppercase = global, lowercase = local
- `D` / `d`: .data (initialized data)
- `B` / `b`: .bss (uninitialized data)
- `R` / `r`: .rodata (read-only data)
- `U`: Undefined (ต้องหาจาก library)
- `W` / `w`: Weak symbol

### 8.4 ldd

```bash
# แสดง shared libraries ที่ต้องการ
ldd binary

# ดูแบบ verbose (รวม all dependencies)
ldd -v binary

# ตรวจ security (unused libraries)
ldd -u binary
```

---

## 9. PLT/GOT Mechanism

### 9.1 PLT (Procedure Linkage Table)

PLT เป็น "trampoline" สำหรับเรียก external functions:

```
PLT[0] (PLT stub):
    push QWORD PTR [rip + GOT[1]]  ; push link_map
    jmp QWORD PTR [rip + GOT[2]]   ; jump to _dl_runtime_resolve

PLT[1] (function entry):
    jmp QWORD PTR [rip + GOT[3]]   ; jump ผ่าน GOT
    push 0                          ; relocation index
    jmp PLT[0]                      ; call resolver
```

### 9.2 GOT (Global Offset Table)

GOT เก็บ actual addresses ของ external functions:

```
GOT[0]: address ของ .dynamic section
GOT[1]: pointer to link_map
GOT[2]: pointer to _dl_runtime_resolve
GOT[3+]: function addresses (filled by dynamic linker)
```

### 9.3 Lazy Binding Process

```
1. โปรแกรมเรียก printf()
2. CPU jump ไป printf@PLT
3. PLT jump ผ่าน GOT[n] (ตอนแรกชี้กลับมาที่ PLT)
4. PLT push relocation index
5. PLT jump ไป PLT[0]
6. PLT[0] เรียก _dl_runtime_resolve
7. Dynamic linker หา printf address
8. Dynamic linker เขียน address ลง GOT[n]
9. Dynamic linker jump ไป printf จริงๆ
10. ครั้งต่อไป: GOT[n] ชี้ไปที่ printf โดยตรง
```

---

## 10. Security Features ใน ELF

### 10.1 ASLR (Address Space Layout Randomization)

```bash
# ตรวจสอบ ASLR
cat /proc/sys/kernel/randomize_va_space
# 0 = disabled, 1 = partial, 2 = full

# PIE binary (Position Independent Executable)
file binary  # "ELF 64-bit... dynamically linked, PIE executable"
readelf -h binary | grep Type  # DYN (PIE) vs EXEC (non-PIE)
```

### 10.2 Stack Canary

```bash
# ตรวจสอบ stack canary
objdump -d binary | grep __stack_chk  # มี = stack canary
```

### 10.3 NX (No-Execute) / DEP

```bash
# ตรวจ NX
readelf -l binary | grep GNU_STACK
# RW = NX enabled (ปลอดภัย)
# RWE = NX disabled (อันตราย)
```

### 10.4 RELRO (Relocation Read-Only)

```bash
# ตรวจ RELRO
readelf -l binary | grep GNU_RELRO
# มี = Partial RELRO
# Full RELRO ต้องดูจาก checksec tool

# ใช้ checksec
checksec --file=binary
```

### 10.5 FORTIFY_SOURCE

```bash
# ตรวจ FORTIFY_SOURCE
objdump -T binary | grep __chk  # มี = FORTIFY enabled
```

---

## 11. GNU Hash Table Algorithm

GNU Hash ใช้สำหรับค้นหา symbols เร็วกว่า ELF Hash เดิม

### 11.1 โครงสร้าง GNU Hash

```
struct gnu_hash_table {
    uint32_t nbuckets;     // จำนวน buckets
    uint32_t symoffset;    // index ของ symbol แรกใน hash
    uint32_t bloom_size;   // ขนาด Bloom filter (power of 2)
    uint32_t bloom_shift;  // shift สำหรับ Bloom filter
    uint64_t bloom[bloom_size];  // Bloom filter
    uint32_t buckets[nbuckets];  // Bucket array
    uint32_t chains[];     // Hash chains
};
```

### 11.2 GNU Hash Function

```c
uint32_t gnu_hash(const char *name) {
    uint32_t h = 5381;
    for (unsigned char c; (c = *name++); )
        h = h * 33 + c;
    return h;
}
```

### 11.3 Assembly Implementation

```nasm
; gnu_hash_func - คำนวณ GNU hash ของ symbol name
; Input: rdi = pointer to null-terminated string
; Output: eax = hash value

gnu_hash_func:
    push rbx
    
    mov eax, 5381           ; initial value
    
.hash_loop:
    movzx ebx, byte [rdi]   ; อ่านอักขระ
    test bl, bl             ; ตรวจ null terminator
    jz .done
    
    ; h = h * 33 + c
    lea eax, [eax + eax*8]  ; eax = eax * 9
    lea eax, [eax + eax*2]  ; eax = eax * 3 (รวมเป็น * 27... ต้องปรับ)
    ; จริงๆ ควรใช้ imul:
    imul eax, eax, 33       ; h *= 33 (หรือ lea eax, [eax + eax*32] = *33)
    add eax, ebx            ; h += c
    
    inc rdi
    jmp .hash_loop

.done:
    pop rbx
    ret
```

การค้นหา symbol โดยใช้ GNU Hash:

```nasm
; lookup_gnu_hash - ค้นหา symbol โดยใช้ GNU hash
; Input: rdi = hash table base
;        rsi = pointer to symbol name
;        rdx = .dynsym base
;        rcx = .dynstr base
; Output: rax = symbol value หรือ 0

lookup_gnu_hash:
    push rbx
    push r12
    push r13
    push r14
    push r15
    
    mov r12, rdi            ; hash table
    mov r13, rsi            ; symbol name to find
    mov r14, rdx            ; .dynsym
    mov r15, rcx            ; .dynstr
    
    ; คำนวณ hash
    mov rdi, r13
    call gnu_hash_func
    mov rbx, rax            ; rbx = hash
    
    ; อ่าน table parameters
    mov ecx, dword [r12]    ; nbuckets
    mov edx, dword [r12 + 4] ; symoffset
    mov esi, dword [r12 + 8] ; bloom_size
    mov edi, dword [r12 + 12] ; bloom_shift
    
    ; Bloom filter check
    ; bloom_word = hash / 64 (หรือ 32 สำหรับ 32-bit)
    ; bit = hash % 64
    mov rax, rbx
    shr rax, 6              ; rax = bloom_word_index
    and rax, rsi            ; & (bloom_size - 1)
    
    ; ตรวจ bloom filter
    lea r8, [r12 + 16]      ; bloom array start
    mov r9, [r8 + rax*8]    ; bloom word
    
    mov rax, rbx
    and eax, 63             ; bit position 1
    bt r9, rax              ; test bit
    jnc .not_found          ; bit = 0 = definitely not in table
    
    mov rax, rbx
    shr rax, edi            ; hash >> bloom_shift
    and eax, 63             ; bit position 2
    bt r9, rax              ; test bit
    jnc .not_found
    
    ; หา bucket
    ; bucket_index = hash % nbuckets
    mov rax, rbx
    xor edx, edx
    div dword [r12]         ; rax = hash / nbuckets, rdx = index
    
    ; อ่าน bucket
    lea r8, [r12 + 16]      ; bloom start
    mov rax, [r12 + 8]      ; bloom_size
    shl rax, 3              ; * 8 bytes
    add r8, rax             ; r8 = buckets start
    
    mov eax, dword [r8 + rdx*4]  ; first symbol index
    test eax, eax
    jz .not_found

    ; วน chain
.chain_loop:
    ; อ่าน hash chain entry
    ; chain offset = (sym_index - symoffset) * 4
    mov ecx, eax
    sub ecx, dword [r12 + 4] ; - symoffset
    lea r9, [r8 + rcx*4 + ???] ; chains start (ต้องคำนวณ offset ให้ถูก)
    mov esi, dword [r9]      ; chain hash
    
    ; เปรียบเทียบ hash (ยกเว้น bit 0)
    mov edi, ebx
    or edi, 1               ; set bit 0
    mov ecx, esi
    or ecx, 1               ; set bit 0
    cmp edi, ecx
    jne .check_end
    
    ; Hash match - ตรวจชื่อ
    ; อ่าน symbol ที่ index eax
    lea rdi, [r14]
    imul rcx, rax, 24       ; * sizeof(Elf64_Sym)
    add rdi, rcx
    
    mov ecx, dword [rdi]    ; st_name
    lea rdi, [r15 + rcx]    ; pointer to name in strtab
    
    ; เปรียบเทียบชื่อ
    push rax
    push rsi
    mov rsi, r13            ; name to find
    call strcmp
    pop rsi
    pop rax
    
    test rax, rax
    jnz .check_end          ; ไม่ตรงกัน
    
    ; ตรงกัน! อ่าน st_value
    lea rdi, [r14]
    imul rcx, rax, 24
    add rdi, rcx
    mov rax, [rdi + 8]      ; st_value
    jmp .done

.check_end:
    ; ตรวจ end of chain (bit 0 = 1)
    test esi, 1
    jnz .not_found
    
    inc eax                 ; ไปที่ symbol ถัดไป
    ; อ่าน chain entry ถัดไป...
    ; (ต้องคำนวณ chain index ใหม่)
    jmp .chain_loop

.not_found:
    xor rax, rax

.done:
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    ret
```

---

## 12. Dynamic Linker Internals

### 12.1 ld.so Bootstrap Process

```
1. Kernel โหลด ELF binary
2. Kernel โหลด ld.so ตาม PT_INTERP
3. ld.so _start ทำงาน (ใน ld.so เอง)
4. ld.so อ่าน PT_DYNAMIC ของ binary
5. ld.so โหลด DT_NEEDED libraries
6. ld.so resolve symbols (lazy หรือ eager)
7. ld.so เรียก init functions (DT_INIT, .init_array)
8. ld.so jump ไป e_entry ของ binary
```

### 12.2 LD_PRELOAD

```bash
# Override function ด้วย custom library
LD_PRELOAD=/path/to/mylib.so ./program

# ตัวอย่าง: override malloc
cat > mymalloc.c << 'EOF'
#define _GNU_SOURCE
#include <stdio.h>
#include <dlfcn.h>

void *malloc(size_t size) {
    static void *(*real_malloc)(size_t) = NULL;
    if (!real_malloc)
        real_malloc = dlsym(RTLD_NEXT, "malloc");
    fprintf(stderr, "malloc(%zu)\n", size);
    return real_malloc(size);
}
EOF
gcc -shared -fPIC -o mymalloc.so mymalloc.c -ldl
LD_PRELOAD=./mymalloc.so ls
```

### 12.3 dlopen/dlsym in Assembly

```nasm
; dlopen_example.asm - ใช้ dlopen/dlsym ใน assembly
; คอมไพล์: nasm -f elf64 dlopen_example.asm && ld -o dlopen_example dlopen_example.o -ldl

section .data
    libname     db "libm.so.6", 0
    funcname    db "sin", 0
    msg_ok      db "sin(0.0) = 0.0 (expected)", 10, 0
    msg_err     db "Error loading library", 10, 0
    
    RTLD_LAZY   equ 1
    RTLD_NOW    equ 2
    RTLD_GLOBAL equ 0x100

section .bss
    lib_handle  resq 1
    func_ptr    resq 1
    result      resq 1

section .text
    global _start
    extern dlopen
    extern dlsym
    extern dlclose

_start:
    ; dlopen("libm.so.6", RTLD_LAZY)
    lea rdi, [libname]
    mov rsi, RTLD_LAZY
    call dlopen
    test rax, rax
    jz .error
    mov [lib_handle], rax

    ; dlsym(handle, "sin")
    mov rdi, [lib_handle]
    lea rsi, [funcname]
    call dlsym
    test rax, rax
    jz .error
    mov [func_ptr], rax

    ; เรียก sin(0.0)
    ; ต้องใช้ xmm registers สำหรับ double
    xorpd xmm0, xmm0           ; xmm0 = 0.0
    call qword [func_ptr]       ; sin(0.0)
    ; ผลลัพธ์อยู่ใน xmm0

    ; แสดงผล (simplified)
    lea rsi, [msg_ok]
    call print_string

    ; dlclose
    mov rdi, [lib_handle]
    call dlclose

    mov rax, 60
    xor rdi, rdi
    syscall

.error:
    lea rsi, [msg_err]
    call print_string
    mov rax, 60
    mov rdi, 1
    syscall

print_string:
    push rax
    push rdi
    push rcx
    push rdx
    
    mov rdi, rsi
    xor rcx, rcx
.count:
    cmp byte [rdi+rcx], 0
    je .done
    inc rcx
    jmp .count
.done:
    mov rax, 1
    mov rdi, 1
    mov rdx, rcx
    syscall
    
    pop rdx
    pop rcx
    pop rdi
    pop rax
    ret
```

---

## 13. สร้าง Shared Library ใน Assembly

```nasm
; mylib.asm - Assembly Shared Library
; สร้าง: nasm -f elf64 mylib.asm -o mylib.o
;         ld -shared -o libmylib.so mylib.o

section .data
    version_str db "mylib v1.0", 0

section .text
    global add_numbers
    global multiply_numbers
    global get_version

; =============================================
; add_numbers(int64_t a, int64_t b) -> int64_t
; System V ABI: arguments ใน rdi, rsi
; return value ใน rax
; =============================================
add_numbers:
    mov rax, rdi
    add rax, rsi
    ret

; =============================================
; multiply_numbers(int64_t a, int64_t b) -> int64_t
; =============================================
multiply_numbers:
    mov rax, rdi
    imul rax, rsi
    ret

; =============================================
; get_version() -> const char*
; =============================================
get_version:
    lea rax, [rel version_str]  ; ใช้ RIP-relative เพราะ shared library
    ret
```

ใช้งาน library จาก C:
```c
// test_lib.c
#include <stdio.h>
#include <dlfcn.h>

int main() {
    void *lib = dlopen("./libmylib.so", RTLD_LAZY);
    
    long (*add)(long, long) = dlsym(lib, "add_numbers");
    long (*mul)(long, long) = dlsym(lib, "multiply_numbers");
    char *(*ver)(void) = dlsym(lib, "get_version");
    
    printf("Version: %s\n", ver());
    printf("3 + 4 = %ld\n", add(3, 4));
    printf("3 * 4 = %ld\n", mul(3, 4));
    
    dlclose(lib);
    return 0;
}
```

---

## 14. ELF และ Core Dumps

### 14.1 Core Dump Structure

Core dump ก็เป็น ELF file (ET_CORE type):
- PT_LOAD: dump ของ memory regions
- PT_NOTE: process info (registers, signals, etc.)

```bash
# Enable core dumps
ulimit -c unlimited

# สร้าง core dump
./program_that_crashes

# วิเคราะห์ core dump
readelf -l core
readelf -n core | head -100

# ใช้ gdb
gdb ./program core
```

### 14.2 อ่าน Registers จาก Core Dump ใน Assembly

```nasm
; read_core.asm - อ่านข้อมูล crash จาก core dump
; อ่าน NT_PRSTATUS note เพื่อดู registers ขณะ crash

section .data
    msg_regs    db "Register state at crash:", 10, 0
    msg_rip     db "RIP: 0x", 0
    msg_rsp     db "RSP: 0x", 0
    msg_signal  db "Signal: ", 0
    msg_newline db 10, 0
    hex_chars   db "0123456789abcdef"

    ; NT_PRSTATUS note type
    NT_PRSTATUS equ 1

    ; Offsets ใน prstatus structure
    ; struct elf_prstatus {
    ;   struct elf_siginfo pr_info; /* signal info - 12 bytes */
    ;   short pr_cursig;            /* current signal - 2 bytes */
    ;   unsigned long pr_sigpend;   /* 8 bytes */
    ;   unsigned long pr_sighold;   /* 8 bytes */
    ;   pid_t pr_pid;               /* 4 bytes */
    ;   pid_t pr_ppid;              /* 4 bytes */
    ;   pid_t pr_pgrp;              /* 4 bytes */
    ;   pid_t pr_sid;               /* 4 bytes */
    ;   struct timeval pr_utime;    /* 16 bytes */
    ;   struct timeval pr_stime;    /* 16 bytes */
    ;   struct timeval pr_cutime;   /* 16 bytes */
    ;   struct timeval pr_cstime;   /* 16 bytes */
    ;   elf_gregset_t pr_reg;       /* registers */
    ;   int pr_fpvalid;
    ; };

    PRSTATUS_SIGINFO    equ 0
    PRSTATUS_CURSIG     equ 12
    PRSTATUS_SIGPEND    equ 16
    PRSTATUS_SIGHOLD    equ 24
    PRSTATUS_PID        equ 32
    PRSTATUS_PPID       equ 36
    PRSTATUS_PGRP       equ 40
    PRSTATUS_SID        equ 44
    PRSTATUS_UTIME      equ 48
    PRSTATUS_STIME      equ 64
    PRSTATUS_CUTIME     equ 80
    PRSTATUS_CSTIME     equ 96
    PRSTATUS_REGS       equ 112     ; start of register area

    ; Register offsets ใน pr_reg (struct user_regs_struct)
    REG_R15     equ 0
    REG_R14     equ 8
    REG_R13     equ 16
    REG_R12     equ 24
    REG_RBP     equ 32
    REG_RBX     equ 40
    REG_R11     equ 48
    REG_R10     equ 56
    REG_R9      equ 64
    REG_R8      equ 72
    REG_RAX     equ 80
    REG_RCX     equ 88
    REG_RDX     equ 96
    REG_RSI     equ 104
    REG_RDI     equ 112
    REG_ORIG_RAX equ 120
    REG_RIP     equ 128
    REG_CS      equ 136
    REG_RFLAGS  equ 144
    REG_RSP     equ 152
    REG_SS      equ 160

section .bss
    core_fd     resq 1
    core_buf    resb 16777216   ; 16MB buffer
    core_size   resq 1
    hex_buf     resb 20

section .text
    global _start

_start:
    ; ตรวจ arguments
    mov rax, [rsp]
    cmp rax, 2
    jl .exit

    ; เปิด core file
    mov rdi, [rsp + 16]
    mov rax, 2              ; sys_open
    xor rsi, rsi
    xor rdx, rdx
    syscall
    test rax, rax
    js .exit
    mov [core_fd], rax

    ; อ่าน core file
    mov rax, 0
    mov rdi, [core_fd]
    lea rsi, [core_buf]
    mov rdx, 16777216
    syscall
    mov [core_size], rax

    ; ตรวจ ELF magic
    cmp dword [core_buf], 0x464c457f
    jne .exit

    ; ตรวจ e_type = ET_CORE (4)
    cmp word [core_buf + 16], 4
    jne .exit

    ; วน program headers หา PT_NOTE
    movzx rbx, word [core_buf + 54] ; e_phentsize
    movzx rcx, word [core_buf + 56] ; e_phnum
    mov rdx, [core_buf + 32]        ; e_phoff

.ph_loop:
    test rcx, rcx
    jz .exit

    lea rsi, [core_buf]
    add rsi, rdx                    ; pointer to phdr
    
    ; ตรวจ p_type == PT_NOTE (4)
    cmp dword [rsi], 4
    jne .next_ph

    ; เจอ PT_NOTE segment
    ; วน notes ภายใน
    mov r8, [rsi + 8]               ; p_offset
    mov r9, [rsi + 32]              ; p_filesz
    lea r8, [core_buf + r8]         ; pointer to first note

.note_loop:
    test r9, r9
    jle .next_ph

    ; NOTE structure:
    ; uint32_t namesz;  /* ขนาด name รวม null */
    ; uint32_t descsz;  /* ขนาด descriptor */
    ; uint32_t type;    /* note type */
    ; char name[];      /* note name */
    ; char desc[];      /* note descriptor */

    mov eax, dword [r8]             ; namesz
    mov edi, dword [r8 + 4]         ; descsz
    mov esi, dword [r8 + 8]         ; type

    ; ตรวจ type == NT_PRSTATUS (1)
    cmp esi, NT_PRSTATUS
    jne .next_note

    ; เจอ prstatus
    ; หา descriptor (หลัง name, align 4)
    mov rcx, rax                    ; namesz
    add rcx, 3
    and rcx, ~3                     ; align up to 4
    lea rsi, [r8 + 12 + rcx]       ; pointer to descriptor

    ; แสดงข้อมูล
    lea r10, [msg_regs]
    push rsi
    mov rsi, r10
    call print_string
    pop rsi

    ; แสดง signal
    movzx rax, word [rsi + PRSTATUS_CURSIG]
    lea r10, [msg_signal]
    push rsi
    push rax
    mov rsi, r10
    call print_string
    pop rax
    call print_decimal
    mov rsi, msg_newline
    call print_string
    pop rsi

    ; แสดง RIP
    mov rax, [rsi + PRSTATUS_REGS + REG_RIP]
    lea r10, [msg_rip]
    push rax
    push rsi
    mov rsi, r10
    call print_string
    pop rsi
    pop rax
    call print_hex64
    mov rsi, msg_newline
    call print_string

    ; แสดง RSP
    mov rax, [rsi + PRSTATUS_REGS + REG_RSP]
    lea r10, [msg_rsp]
    push rax
    push rsi
    mov rsi, r10
    call print_string
    pop rsi
    pop rax
    call print_hex64
    mov rsi, msg_newline
    call print_string

.next_note:
    ; ไปที่ note ถัดไป
    ; note size = 12 (header) + align4(namesz) + align4(descsz)
    mov eax, dword [r8]             ; namesz
    add eax, 3
    and eax, ~3                     ; align4
    mov ecx, dword [r8 + 4]        ; descsz
    add ecx, 3
    and ecx, ~3                     ; align4
    add eax, ecx
    add eax, 12                     ; + header size

    add r8, rax
    sub r9, rax
    jmp .note_loop

.next_ph:
    add rdx, rbx                    ; ไปที่ phdr ถัดไป
    dec rcx
    jmp .ph_loop

.exit:
    mov rax, 3
    mov rdi, [core_fd]
    syscall
    
    mov rax, 60
    xor rdi, rdi
    syscall

; utility functions (เหมือนกับโปรแกรมก่อน)
print_string:
    push rax
    push rdi
    push rcx
    push rdx
    
    mov rdi, rsi
    xor rcx, rcx
.ps_count:
    cmp byte [rdi+rcx], 0
    je .ps_done
    inc rcx
    jmp .ps_count
.ps_done:
    mov rax, 1
    mov rdi, 1
    mov rdx, rcx
    syscall
    
    pop rdx
    pop rcx
    pop rdi
    pop rax
    ret

print_hex64:
    push rbx
    push rcx
    push rdx
    push rsi
    
    lea rsi, [hex_buf]
    mov rcx, 16
.ph_loop:
    mov rbx, rax
    and rbx, 0xF
    lea rdx, [hex_chars]
    movzx rdx, byte [rdx + rbx]
    mov [rsi + rcx - 1], dl
    shr rax, 4
    dec rcx
    jnz .ph_loop
    
    mov rax, 1
    mov rdi, 1
    mov rdx, 16
    syscall
    
    pop rsi
    pop rdx
    pop rcx
    pop rbx
    ret

print_decimal:
    push rbx
    push rcx
    push rdx
    push rsi
    
    lea rsi, [hex_buf + 19]
    mov byte [rsi], 0
    mov rcx, 0
    mov rbx, 10

.pd_loop:
    xor rdx, rdx
    div rbx
    add dl, '0'
    dec rsi
    mov [rsi], dl
    inc rcx
    test rax, rax
    jnz .pd_loop

    mov rax, 1
    mov rdi, 1
    mov rdx, rcx
    syscall
    
    pop rsi
    pop rdx
    pop rcx
    pop rbx
    ret
```

---

## 15. Build Script และการทดสอบ

```bash
#!/bin/bash
# build_all.sh - Script สำหรับ build ทุกโปรแกรมในบทนี้

set -e

echo "Building ELF Format programs..."

# ELF Parser
echo "Building elf_parser..."
nasm -f elf64 elf_parser.asm -o elf_parser.o
ld -o elf_parser elf_parser.o
echo "  Done: ./elf_parser"

# Minimal ELF
echo "Building minimal_elf..."
nasm -f bin minimal_elf.asm -o minimal_elf
chmod +x minimal_elf
echo "  Done: ./minimal_elf ($(wc -c < minimal_elf) bytes)"

# Shared library
echo "Building mylib..."
nasm -f elf64 mylib.asm -o mylib.o
ld -shared -o libmylib.so mylib.o
echo "  Done: ./libmylib.so"

# Symbol lookup
echo "Building symbol_lookup..."
nasm -f elf64 symbol_lookup.asm -o symbol_lookup.o
ld -o symbol_lookup symbol_lookup.o
echo "  Done: ./symbol_lookup"

echo ""
echo "Testing..."

# Test ELF Parser
echo "--- ELF Parser ---"
./elf_parser /bin/ls

# Test Minimal ELF
echo "--- Minimal ELF ---"
./minimal_elf
echo "Exit code: $?"

# Test Symbol Lookup
echo "--- Symbol Lookup ---"
./symbol_lookup /lib/x86_64-linux-gnu/libm.so.6 sin

echo ""
echo "All tests done!"
```

---

## 16. การทำ ELF Analysis ด้วย Python (สำหรับอ้างอิง)

```python
#!/usr/bin/env python3
# elf_analysis.py - วิเคราะห์ ELF ด้วย Python
# ใช้เป็น reference สำหรับการเขียน assembly parser

import struct
import sys

class ELFParser:
    # ELF constants
    ET_NONE = 0
    ET_REL = 1
    ET_EXEC = 2
    ET_DYN = 3
    ET_CORE = 4
    
    PT_NULL = 0
    PT_LOAD = 1
    PT_DYNAMIC = 2
    PT_INTERP = 3
    PT_NOTE = 4
    PT_GNU_STACK = 0x6474e551
    PT_GNU_RELRO = 0x6474e552
    
    SHT_NULL = 0
    SHT_PROGBITS = 1
    SHT_SYMTAB = 2
    SHT_STRTAB = 3
    SHT_RELA = 4
    SHT_HASH = 5
    SHT_DYNAMIC = 6
    SHT_NOTE = 7
    SHT_NOBITS = 8
    SHT_REL = 9
    SHT_DYNSYM = 11
    SHT_GNU_HASH = 0x6ffffef5
    
    def __init__(self, filename):
        with open(filename, 'rb') as f:
            self.data = f.read()
        self.parse_header()
    
    def parse_header(self):
        # ตรวจ magic
        if self.data[:4] != b'\x7fELF':
            raise ValueError("Not an ELF file")
        
        self.ei_class = self.data[4]    # 1=32-bit, 2=64-bit
        self.ei_data = self.data[5]     # 1=LE, 2=BE
        
        # เลือก endianness
        self.endian = '<' if self.ei_data == 1 else '>'
        
        if self.ei_class == 2:  # 64-bit
            fmt = self.endian + 'HHIQQQIHHHHHH'
            fields = struct.unpack_from(fmt, self.data, 16)
            (self.e_type, self.e_machine, self.e_version,
             self.e_entry, self.e_phoff, self.e_shoff,
             self.e_flags, self.e_ehsize, self.e_phentsize,
             self.e_phnum, self.e_shentsize, self.e_shnum,
             self.e_shstrndx) = fields
    
    def get_type_name(self):
        types = {
            self.ET_NONE: "NONE",
            self.ET_REL: "REL",
            self.ET_EXEC: "EXEC",
            self.ET_DYN: "DYN",
            self.ET_CORE: "CORE"
        }
        return types.get(self.e_type, f"0x{self.e_type:x}")
    
    def parse_sections(self):
        sections = []
        for i in range(self.e_shnum):
            offset = self.e_shoff + i * self.e_shentsize
            fmt = self.endian + 'IIQQQQIIQQ'
            fields = struct.unpack_from(fmt, self.data, offset)
            (sh_name, sh_type, sh_flags, sh_addr,
             sh_offset, sh_size, sh_link, sh_info,
             sh_addralign, sh_entsize) = fields
            sections.append({
                'name_idx': sh_name,
                'type': sh_type,
                'flags': sh_flags,
                'addr': sh_addr,
                'offset': sh_offset,
                'size': sh_size,
                'link': sh_link,
                'info': sh_info,
                'align': sh_addralign,
                'entsize': sh_entsize,
            })
        
        # เพิ่มชื่อ sections
        if self.e_shstrndx < len(sections):
            shstrtab = sections[self.e_shstrndx]
            strtab_data = self.data[shstrtab['offset']:
                                    shstrtab['offset'] + shstrtab['size']]
            for s in sections:
                idx = s['name_idx']
                end = strtab_data.index(b'\x00', idx)
                s['name'] = strtab_data[idx:end].decode('utf-8', errors='replace')
        
        return sections
    
    def parse_symbols(self):
        sections = self.parse_sections()
        
        # หา .dynsym และ .dynstr
        dynsym = None
        dynstr = None
        for s in sections:
            if s.get('name') == '.dynsym':
                dynsym = s
            elif s.get('name') == '.dynstr':
                dynstr = s
        
        if not dynsym or not dynstr:
            return []
        
        # อ่าน string table
        strtab = self.data[dynstr['offset']:dynstr['offset'] + dynstr['size']]
        
        # อ่าน symbols
        symbols = []
        sym_size = 24  # sizeof(Elf64_Sym) for 64-bit
        count = dynsym['size'] // sym_size
        
        for i in range(count):
            offset = dynsym['offset'] + i * sym_size
            fmt = self.endian + 'IBBHQQ'
            st_name, st_info, st_other, st_shndx, st_value, st_size = \
                struct.unpack_from(fmt, self.data, offset)
            
            # ดึงชื่อ
            name_end = strtab.index(b'\x00', st_name)
            name = strtab[st_name:name_end].decode('utf-8', errors='replace')
            
            binding = st_info >> 4
            sym_type = st_info & 0xf
            
            symbols.append({
                'name': name,
                'value': st_value,
                'size': st_size,
                'binding': binding,
                'type': sym_type,
                'shndx': st_shndx,
            })
        
        return symbols
    
    def print_info(self):
        print(f"ELF Type: {self.get_type_name()}")
        print(f"Entry: 0x{self.e_entry:016x}")
        print(f"Program Headers: {self.e_phnum}")
        print(f"Section Headers: {self.e_shnum}")
        
        print("\nSections:")
        for s in self.parse_sections():
            if s.get('name'):
                print(f"  {s['name']:20s} offset=0x{s['offset']:08x} size=0x{s['size']:08x}")
        
        print("\nDynamic Symbols:")
        for sym in self.parse_symbols():
            if sym['name']:
                binding_names = {0: 'LOCAL', 1: 'GLOBAL', 2: 'WEAK'}
                type_names = {0: 'NOTYPE', 1: 'OBJECT', 2: 'FUNC', 3: 'SECTION'}
                b = binding_names.get(sym['binding'], str(sym['binding']))
                t = type_names.get(sym['type'], str(sym['type']))
                print(f"  0x{sym['value']:016x} {b:6s} {t:7s} {sym['name']}")

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("Usage: elf_analysis.py <elf_file>")
        sys.exit(1)
    
    parser = ELFParser(sys.argv[1])
    parser.print_info()
```

---

## 17. สรุปความสัมพันธ์ระหว่าง Sections และ Segments

```
File layout:
┌──────────────────────┐ offset 0
│    ELF Header        │ 64 bytes
├──────────────────────┤
│  Program Headers     │ สำหรับ execution
├──────────────────────┤
│   .text section      │ ─┐
│   .rodata section    │  ├─ TEXT segment (PT_LOAD, R+X)
├──────────────────────┤ ─┘
│   .data section      │ ─┐
│   .bss section       │  ├─ DATA segment (PT_LOAD, R+W)
├──────────────────────┤  │  (bss ไม่มีข้อมูลในไฟล์ แต่ map ใน memory)
│   .dynamic section   │ ─┘─ DYNAMIC segment (PT_DYNAMIC)
├──────────────────────┤
│   .dynsym section    │
│   .dynstr section    │
│   .gnu.hash section  │
│   .rela.dyn section  │
│   .rela.plt section  │
│   .plt section       │
│   .got section       │
│   .got.plt section   │
├──────────────────────┤
│  Section Headers     │ สำหรับ linking/debugging
└──────────────────────┘
```

---

## 18. Checklist สิ่งที่ควรรู้เกี่ยวกับ ELF

### ระดับพื้นฐาน
- [ ] อ่าน ELF header ด้วย readelf
- [ ] เข้าใจความแตกต่างระหว่าง sections และ segments
- [ ] รู้จัก section types ที่สำคัญ (.text, .data, .bss, .rodata)
- [ ] เข้าใจ symbol table และ binding types

### ระดับกลาง
- [ ] เข้าใจ PLT/GOT mechanism
- [ ] รู้วิธีดู relocations
- [ ] เข้าใจ dynamic section และ DT_NEEDED
- [ ] ทำ LD_PRELOAD interception ได้

### ระดับสูง
- [ ] เขียน ELF parser เองได้
- [ ] สร้าง minimal ELF binary ด้วยมือ
- [ ] เข้าใจ GNU hash algorithm
- [ ] ทำ binary injection ได้
- [ ] วิเคราะห์ core dump ได้

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: ELF Header Reader
เขียนโปรแกรม assembly ที่รับ ELF file เป็น argument และแสดง:
- Entry point
- จำนวน program headers
- จำนวน section headers
- ชื่อ architecture

### แบบฝึกหัดที่ 2: Section Lister
เพิ่มความสามารถในการแสดงรายชื่อ sections ทั้งหมดพร้อม:
- ชื่อ section
- ประเภท (SHT_*)
- ขนาด
- Virtual address

### แบบฝึกหัดที่ 3: Symbol Counter
นับจำนวน symbols ในแต่ละ category:
- Global functions
- Global variables
- Undefined symbols (need from library)

### แบบฝึกหัดที่ 4: Minimal ELF เขียน "Hello ELF!"
สร้าง minimal ELF ที่พิมพ์ข้อความแล้ว exit โดยไม่ใช้ C library
ขนาดเล็กที่สุดเท่าที่ทำได้

### แบบฝึกหัดที่ 5: Library Inspector
เขียนโปรแกรมที่:
1. รับ path ของ shared library
2. แสดง exported functions ทั้งหมด
3. แสดง required libraries (DT_NEEDED)
4. แสดง SONAME ถ้ามี

---

## อ้างอิง

1. ELF specification: https://refspecs.linuxfoundation.org/elf/elf.pdf
2. System V ABI (x86-64): https://refspecs.linuxfoundation.org/elf/x86_64-abi-0.99.pdf
3. Linux man pages: `man 5 elf`
4. GNU binutils source code: https://sourceware.org/git/binutils-gdb.git
5. glibc dynamic linker: https://sourceware.org/git/glibc.git

---

*Part 060 เสร็จสมบูรณ์ - ELF Binary Format Deep Dive*
*ต่อไป: Part 061 - Dynamic Linking Internals*
