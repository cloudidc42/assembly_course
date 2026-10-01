# หลักสูตร Assembly Programming ระดับมืออาชีพ (x86/ARM)
### Professional & World-Class Assembly Language Programming Course

---

```
     _                           _     _
    / \   ___ ___  ___ _ __ ___ | |__ | |_   _
   / _ \ / __/ __|/ _ \ '_ ` _ \| '_ \| | | | |
  / ___ \\__ \__ \  __/ | | | | | |_) | | |_| |
 /_/   \_\___/___/\___|_| |_| |_|_.__/|_|\__, |
  ____                                    |___/
 |  _ \ _ __ ___   __ _ _ __ __ _ _ __ ___  _ __ ___(_)_ __   __ _
 | |_) | '__/ _ \ / _` | '__/ _` | '_ ` _ \| '_ ` _ \ | '_ \ / _` |
 |  __/| | | (_) | (_| | | | (_| | | | | | | | | | | | | | | | (_| |
 |_|   |_|  \___/ \__, |_|  \__,_|_| |_| |_|_| |_| |_|_|_| |_|\__, |
                  |___/                                          |___/
```

> **"จากศูนย์ถึงระดับโลก — จาก bits สู่ระบบปฏิบัติการ"**
> *From Zero to World-Class — From Bits to Operating Systems*

---

## สารบัญหลักสูตร (Course Table of Contents)

| ข้อมูลหลักสูตร | รายละเอียด |
|---|---|
| ระดับ (Level) | Beginner → Expert → World-Class |
| จำนวน Parts | 100 Parts (1,000+ Steps) |
| ภาษาที่รองรับ | x86 (32/64-bit), ARM (32/64-bit) |
| Assemblers | NASM, GAS (AT&T), ARM Assembler |
| เนื้อหาต่อ Part | 500 – 3,000+ บรรทัด |
| ภาษาที่ใช้สอน | ไทย/อังกฤษ (Thai/English) |

---

## ปรัชญาหลักสูตร (Course Philosophy)

หลักสูตรนี้ถูกออกแบบมาเพื่อพาคุณจากการไม่รู้จัก Assembly เลย ไปจนถึงสามารถ:

- **เขียนโปรแกรมระดับ low-level** ที่ทำงานได้ใกล้ชิดกับฮาร์ดแวร์มากที่สุด
- **เข้าใจสถาปัตยกรรมคอมพิวเตอร์** ในระดับที่ deep ที่สุด
- **สร้างระบบปฏิบัติการ** ของตัวเองได้ตั้งแต่ต้น
- **Reverse engineer** โปรแกรมและมัลแวร์ได้
- **Optimize** โค้ดในระดับที่ compiler ไม่สามารถทำได้
- **เป็น world-class programmer** ที่เข้าใจทุก layer ของระบบคอมพิวเตอร์

```
Level 1: Basic Assembly (Parts 1-20)
    ↓
Level 2: Intermediate (Parts 21-50)
    ↓
Level 3: Advanced (Parts 51-70)
    ↓
Level 4: Expert (Parts 71-90)
    ↓
Level 5: World-Class (Parts 91-100)
```

---

## ความต้องการเบื้องต้น (Prerequisites)

### ความรู้ที่จำเป็น (Required Knowledge)
- [ ] พื้นฐาน Binary และ Hexadecimal (เลขฐาน 2 และ 16)
- [ ] ความเข้าใจพื้นฐาน C หรือภาษาโปรแกรมมิ่งใดก็ได้ (แนะนำ)
- [ ] ความเข้าใจพื้นฐาน Linux command line
- [ ] ความอยากรู้ว่าคอมพิวเตอร์ทำงานอย่างไร

### ความรู้ที่เป็นประโยชน์ (Helpful Knowledge)
- [ ] พื้นฐาน C Programming
- [ ] ความเข้าใจเรื่อง Memory Management
- [ ] พื้นฐาน Operating Systems
- [ ] ความรู้เรื่อง Computer Architecture

---

## การติดตั้งเครื่องมือ (Setup Instructions)

### 1. ติดตั้ง NASM (Netwide Assembler) สำหรับ x86

#### Ubuntu/Debian
```bash
sudo apt update
sudo apt install nasm -y
nasm --version  # ตรวจสอบการติดตั้ง
```

#### macOS
```bash
brew install nasm
nasm --version
```

#### Windows
```powershell
# ดาวน์โหลด installer จาก https://www.nasm.us/
# หรือใช้ Chocolatey:
choco install nasm
```

### 2. ติดตั้ง GAS (GNU Assembler) สำหรับ AT&T Syntax

```bash
# GAS มาพร้อมกับ binutils
sudo apt install binutils gcc -y

# ตรวจสอบ
as --version
ld --version
```

### 3. ติดตั้ง ARM Cross-Compiler

#### ARM 32-bit (arm-linux-gnueabihf)
```bash
sudo apt install gcc-arm-linux-gnueabihf -y
sudo apt install binutils-arm-linux-gnueabihf -y

# ตรวจสอบ
arm-linux-gnueabihf-as --version
arm-linux-gnueabihf-gcc --version
```

#### ARM 64-bit / AArch64
```bash
sudo apt install gcc-aarch64-linux-gnu -y
sudo apt install binutils-aarch64-linux-gnu -y

# ตรวจสอบ
aarch64-linux-gnu-as --version
```

#### QEMU สำหรับ Emulate ARM บน x86
```bash
sudo apt install qemu-user qemu-user-static -y
sudo apt install qemu-system-arm -y

# ตรวจสอบ
qemu-arm --version
qemu-aarch64 --version
```

### 4. เครื่องมือ Debug และ Disassembly

```bash
# GDB - GNU Debugger
sudo apt install gdb -y

# GDB with pwndbg (เพิ่ม features)
git clone https://github.com/pwndbg/pwndbg
cd pwndbg && ./setup.sh

# Radare2 - Reverse Engineering Framework
sudo apt install radare2 -y

# objdump - Disassembler
sudo apt install binutils -y  # มาพร้อม binutils

# strace - System Call Tracer
sudo apt install strace -y

# ltrace - Library Call Tracer
sudo apt install ltrace -y
```

### 5. Text Editor แนะนำ

```bash
# VS Code พร้อม Extension
# - x86 and x86_64 Assembly
# - ARM Assembly
# - Hex Editor

# Vim/Neovim พร้อม Assembly highlighting
sudo apt install vim -y

# nano (สำหรับผู้เริ่มต้น)
sudo apt install nano -y
```

### 6. ตรวจสอบการติดตั้งทั้งหมด

```bash
#!/bin/bash
echo "=== Assembly Course Tools Check ==="
echo ""

check_tool() {
    if command -v $1 &> /dev/null; then
        echo "[OK] $1: $(command -v $1)"
    else
        echo "[MISSING] $1: not found"
    fi
}

check_tool nasm
check_tool as
check_tool ld
check_tool gcc
check_tool gdb
check_tool objdump
check_tool arm-linux-gnueabihf-as
check_tool aarch64-linux-gnu-as
check_tool qemu-arm
check_tool qemu-aarch64
check_tool r2

echo ""
echo "=== Setup Complete Check ==="
```

---

## โครงสร้างไฟล์หลักสูตร (Course File Structure)

```
assembly_course/
├── README.md                    # ไฟล์นี้ - ภาพรวมหลักสูตร
│
├── section_01_foundations/      # Section 1: รากฐาน (Parts 1-10)
│   ├── part_001_binary_hex.md
│   ├── part_002_cpu_architecture.md
│   ├── part_003_registers_x86.md
│   ├── part_004_nasm_first_program.md
│   ├── part_005_mov_instructions.md
│   ├── part_006_arithmetic_operations.md
│   ├── part_007_flags_register.md
│   ├── part_008_jumps_branches.md
│   ├── part_009_loops.md
│   └── part_010_system_calls_linux.md
│
├── section_02_x86_data/         # Section 2: x86 Data (Parts 11-20)
│   ├── part_011_memory_addressing.md
│   ├── part_012_arrays_and_strings.md
│   ├── part_013_stack_operations.md
│   ├── part_014_bitwise_operations.md
│   ├── part_015_shift_rotate.md
│   ├── part_016_mul_div_operations.md
│   ├── part_017_string_instructions.md
│   ├── part_018_data_structures.md
│   ├── part_019_io_operations.md
│   └── part_020_mixed_c_assembly.md
│
├── section_03_procedures/       # Section 3: Procedures (Parts 21-30)
│   ├── part_021_functions_basics.md
│   ├── part_022_calling_conventions.md
│   ├── part_023_stack_frames.md
│   ├── part_024_recursive_functions.md
│   ├── part_025_local_variables.md
│   ├── part_026_parameter_passing.md
│   ├── part_027_return_values.md
│   ├── part_028_modular_programming.md
│   ├── part_029_macros.md
│   └── part_030_linking_multiple_files.md
│
├── section_04_arm/              # Section 4: ARM Assembly (Parts 31-40)
│   ├── part_031_arm_architecture.md
│   ├── part_032_arm_registers.md
│   ├── part_033_arm_instructions.md
│   ├── part_034_arm_addressing.md
│   ├── part_035_arm_conditions.md
│   ├── part_036_arm_procedures.md
│   ├── part_037_arm_thumb.md
│   ├── part_038_aarch64_basics.md
│   ├── part_039_arm_linux_syscalls.md
│   └── part_040_arm_vs_x86.md
│
├── section_05_simd_fpu/         # Section 5: SIMD & FPU (Parts 41-50)
│   ├── part_041_x87_fpu_basics.md
│   ├── part_042_sse_introduction.md
│   ├── part_043_sse2_packed_integers.md
│   ├── part_044_sse_floating_point.md
│   ├── part_045_avx_introduction.md
│   ├── part_046_avx2_advanced.md
│   ├── part_047_arm_neon.md
│   ├── part_048_arm_sve.md
│   ├── part_049_simd_optimization.md
│   └── part_050_multimedia_processing.md
│
├── section_06_system/           # Section 6: System Programming (Parts 51-60)
│   ├── part_051_linux_internals.md
│   ├── part_052_system_calls_advanced.md
│   ├── part_053_signals_assembly.md
│   ├── part_054_process_management.md
│   ├── part_055_memory_management.md
│   ├── part_056_file_operations.md
│   ├── part_057_network_programming.md
│   ├── part_058_threads_assembly.md
│   ├── part_059_interrupt_handling.md
│   └── part_060_device_drivers_intro.md
│
├── section_07_optimization/     # Section 7: Optimization (Parts 61-70)
│   ├── part_061_cpu_pipeline.md
│   ├── part_062_cache_optimization.md
│   ├── part_063_branch_prediction.md
│   ├── part_064_instruction_scheduling.md
│   ├── part_065_loop_optimization.md
│   ├── part_066_memory_alignment.md
│   ├── part_067_profiling_techniques.md
│   ├── part_068_compiler_output_analysis.md
│   ├── part_069_performance_benchmarking.md
│   └── part_070_real_world_optimization.md
│
├── section_08_security/         # Section 8: Security (Parts 71-80)
│   ├── part_071_reverse_engineering_basics.md
│   ├── part_072_disassembly_techniques.md
│   ├── part_073_debugging_advanced.md
│   ├── part_074_buffer_overflow.md
│   ├── part_075_shellcode_writing.md
│   ├── part_076_rop_chains.md
│   ├── part_077_malware_analysis.md
│   ├── part_078_anti_debugging.md
│   ├── part_079_obfuscation_techniques.md
│   └── part_080_ctf_challenges.md
│
├── section_09_os_dev/           # Section 9: OS Development (Parts 81-90)
│   ├── part_081_bootloader_basics.md
│   ├── part_082_real_mode_programming.md
│   ├── part_083_protected_mode.md
│   ├── part_084_long_mode_64bit.md
│   ├── part_085_gdt_idt_setup.md
│   ├── part_086_interrupt_handlers.md
│   ├── part_087_memory_paging.md
│   ├── part_088_kernel_basics.md
│   ├── part_089_process_scheduler.md
│   └── part_090_mini_os_project.md
│
├── section_10_expert/           # Section 10: Expert Level (Parts 91-100)
│   ├── part_091_compiler_design.md
│   ├── part_092_jit_compilation.md
│   ├── part_093_virtualization.md
│   ├── part_094_hypervisor_basics.md
│   ├── part_095_hardware_interfaces.md
│   ├── part_096_embedded_systems.md
│   ├── part_097_real_time_systems.md
│   ├── part_098_cryptography_assembly.md
│   ├── part_099_contributing_to_oss.md
│   └── part_100_capstone_project.md
│
├── exercises/                   # แบบฝึกหัด
│   ├── basic/
│   ├── intermediate/
│   ├── advanced/
│   └── expert/
│
├── projects/                    # โปรเจกต์จริง
│   ├── project_01_calculator/
│   ├── project_02_string_lib/
│   ├── project_03_sort_algorithms/
│   ├── project_04_mini_shell/
│   └── project_05_bootloader/
│
└── references/                  # เอกสารอ้างอิง
    ├── x86_instruction_set.md
    ├── arm_instruction_set.md
    ├── system_calls_linux.md
    ├── calling_conventions.md
    └── cheatsheets/
```

---

## สารบัญเนื้อหาทั้งหมด (Complete Table of Contents)

---

### SECTION 1: รากฐาน Assembly — Foundations (Parts 1–10)

> รากฐานที่แข็งแกร่งคือกุญแจสู่ความเชี่ยวชาญ เริ่มต้นจากสิ่งพื้นฐานที่สุด

| Part | ชื่อเรื่อง | เนื้อหาสำคัญ |
|------|-----------|------------|
| **001** | Binary, Hex และ Data Representation | เลขฐาน 2, 8, 10, 16 — Two's complement — IEEE 754 |
| **002** | สถาปัตยกรรม CPU และ Von Neumann | ALU, CU, Registers, Bus — Memory Hierarchy — Clock cycles |
| **003** | Registers ใน x86/x86-64 | General purpose, Segment, Control registers — 8/16/32/64-bit |
| **004** | โปรแกรมแรกด้วย NASM | Hello World — Assemble/Link/Run — ELF format basics |
| **005** | MOV Instruction และ Addressing Modes | Immediate, Register, Memory — Effective address calculation |
| **006** | Arithmetic Operations | ADD, SUB, INC, DEC, NEG — Signed vs Unsigned arithmetic |
| **007** | FLAGS Register และการทำงาน | CF, ZF, SF, OF, PF — EFLAGS/RFLAGS — Condition checking |
| **008** | Jump Instructions และ Branching | JMP, JE, JNE, JL, JG — Conditional jumps — Branch tables |
| **009** | Loops และ Iteration | LOOP instruction — CX/RCX counter — Loop patterns |
| **010** | System Calls บน Linux | INT 0x80 vs SYSCALL — SYS_write, SYS_read, SYS_exit |

---

### SECTION 2: x86 การจัดการข้อมูล — x86 Data Manipulation (Parts 11–20)

> ควบคุม memory และข้อมูลได้อย่างแม่นยำในระดับ bit

| Part | ชื่อเรื่อง | เนื้อหาสำคัญ |
|------|-----------|------------|
| **011** | Memory Addressing Modes ขั้นสูง | Base+Index+Scale+Offset — LEA — Segment overrides |
| **012** | Arrays และ String Handling | Array traversal — String copy/compare — Null-termination |
| **013** | Stack Operations | PUSH, POP — Stack frame — Stack alignment — Red zone |
| **014** | Bitwise Operations | AND, OR, XOR, NOT — Bit masking — Flag testing |
| **015** | Shift และ Rotate Instructions | SHL, SHR, SAL, SAR, ROL, ROR, RCL, RCR |
| **016** | Multiplication และ Division | MUL, IMUL, DIV, IDIV — 64-bit multiply — Division tricks |
| **017** | String Instructions | MOVS, CMPS, SCAS, LODS, STOS — REP prefix — Direction flag |
| **018** | Data Structures ใน Assembly | Struct, Union, Arrays of structs — Manual layout |
| **019** | I/O Operations | BIOS interrupts — Linux I/O syscalls — Port I/O (IN/OUT) |
| **020** | Mixed C/Assembly Programming | Inline asm — External asm functions — ABI compliance |

---

### SECTION 3: Procedures และ Modular Code (Parts 21–30)

> สร้างโปรแกรมขนาดใหญ่ที่จัดการได้ด้วย modular design

| Part | ชื่อเรื่อง | เนื้อหาสำคัญ |
|------|-----------|------------|
| **021** | Functions พื้นฐาน | CALL, RET — Near vs Far calls — Prologue/Epilogue |
| **022** | Calling Conventions | cdecl, stdcall, fastcall, System V AMD64 ABI |
| **023** | Stack Frames | EBP/RBP frame — Frame pointer — Stack unwinding |
| **024** | Recursive Functions | Fibonacci — Tower of Hanoi — Tail recursion |
| **025** | Local Variables | Stack allocation — Variable lifetime — Frame layout |
| **026** | Parameter Passing | Register params — Stack params — Mixed passing |
| **027** | Return Values | Single/multiple returns — Struct returns — Floating point |
| **028** | Modular Programming | Separate files — Object files — Namespacing |
| **029** | Macros และ Preprocessor | %define, %macro, %rep — Conditional assembly |
| **030** | Linking Multiple Files | Static/dynamic linking — Symbol resolution — Libraries |

---

### SECTION 4: ARM Assembly (Parts 31–40)

> เชี่ยวชาญ ARM — สถาปัตยกรรมที่ขับเคลื่อนโลก mobile และ embedded

| Part | ชื่อเรื่อง | เนื้อหาสำคัญ |
|------|-----------|------------|
| **031** | ARM Architecture Overview | RISC vs CISC — ARM pipeline — Cortex families |
| **032** | ARM Registers (ARM32) | R0-R15 — SP, LR, PC — CPSR — Banked registers |
| **033** | ARM Instruction Set Basics | Data processing — Load/Store — Branch |
| **034** | ARM Addressing Modes | Offset, Pre-indexed, Post-indexed — Barrel shifter |
| **035** | ARM Conditional Execution | Condition codes — Conditional instructions — CPSR flags |
| **036** | ARM Procedures และ ABI | AAPCS — Function calls — Stack management |
| **037** | Thumb และ Thumb-2 | 16-bit Thumb — Thumb-2 extensions — Interworking |
| **038** | AArch64 (ARM64) Basics | 64-bit registers — New instruction set — SP alignment |
| **039** | ARM Linux System Calls | SVC instruction — ARM syscall table — 32/64-bit |
| **040** | ARM vs x86 Comparison | Key differences — Porting code — Performance trade-offs |

---

### SECTION 5: SIMD และ FPU (Parts 41–50)

> ประมวลผลข้อมูลหลายชุดพร้อมกันด้วย vector instructions

| Part | ชื่อเรื่อง | เนื้อหาสำคัญ |
|------|-----------|------------|
| **041** | x87 FPU Basics | Floating-point stack — FLD, FST, FADD, FMUL — Control word |
| **042** | SSE Introduction | XMM registers — Packed vs Scalar — MOVAPS, MOVUPS |
| **043** | SSE2 Packed Integers | Integer vectors — PADDB, PADDW, PADDD — Saturation |
| **044** | SSE Floating-Point | ADDSS, ADDPS, MULSS, MULPS — Single/double precision |
| **045** | AVX Introduction | YMM registers — 256-bit vectors — Three-operand form |
| **046** | AVX2 Advanced | 256-bit integer ops — Gather/Scatter — FMA |
| **047** | ARM NEON | NEON registers — VLD/VST — VADD, VMUL — Intrinsics |
| **048** | ARM SVE (Scalable Vector Extension) | Variable vector length — Predicate registers |
| **049** | SIMD Optimization Patterns | Horizontal ops — Shuffles — Auto-vectorization hints |
| **050** | Multimedia Processing | Image processing — Audio DSP — Video codecs |

---

### SECTION 6: System Programming (Parts 51–60)

> เจาะลึก Linux kernel interface และ low-level system programming

| Part | ชื่อเรื่อง | เนื้อหาสำคัญ |
|------|-----------|------------|
| **051** | Linux Internals | Kernel architecture — User/Kernel space — vDSO |
| **052** | System Calls ขั้นสูง | mmap, brk, clone — Error handling — POSIX compliance |
| **053** | Signals ใน Assembly | signal/sigaction — Signal handlers — SA_RESTORER |
| **054** | Process Management | fork, exec, wait — Process state — Zombie/Orphan |
| **055** | Memory Management | Virtual memory — Page tables — mmap — Memory mapped I/O |
| **056** | File Operations | open, read, write, close — mmap files — Async I/O |
| **057** | Network Programming | socket, bind, connect — Syscall-level networking |
| **058** | Threads ใน Assembly | clone syscall — Thread-local storage — Synchronization |
| **059** | Interrupt Handling | Hardware interrupts — IDT — ISR writing |
| **060** | Device Drivers (Intro) | Character devices — Module basics — ioctl |

---

### SECTION 7: Optimization Techniques (Parts 61–70)

> เขียนโค้ดที่เร็วที่สุดในโลก — เหนือกว่า compiler

| Part | ชื่อเรื่อง | เนื้อหาสำคัญ |
|------|-----------|------------|
| **061** | CPU Pipeline Architecture | Stages — Hazards — Out-of-order execution |
| **062** | Cache Optimization | Cache lines — Spatial/Temporal locality — Prefetching |
| **063** | Branch Prediction | Prediction algorithms — Branch-free code — CMOV |
| **064** | Instruction Scheduling | Dependency chains — Throughput vs Latency — Port pressure |
| **065** | Loop Optimization | Unrolling — Loop fusion — Strength reduction |
| **066** | Memory Alignment | SIMD alignment — Structure padding — Alignment hints |
| **067** | Profiling Techniques | RDTSC — Performance counters — VTune/perf |
| **068** | Compiler Output Analysis | Reading objdump — Godbolt — Optimization flags |
| **069** | Performance Benchmarking | Microbenchmarking — Avoiding pitfalls — Statistical analysis |
| **070** | Real-World Optimization | Case studies — Crypto, codec, math library optimization |

---

### SECTION 8: Security & Reverse Engineering (Parts 71–80)

> เข้าใจทั้ง attack และ defense ในระดับ binary

| Part | ชื่อเรื่อง | เนื้อหาสำคัญ |
|------|-----------|------------|
| **071** | Reverse Engineering Basics | Static vs Dynamic analysis — Tools overview |
| **072** | Disassembly Techniques | objdump, IDA, Ghidra — Code recognition patterns |
| **073** | Advanced Debugging (GDB) | Breakpoints, watchpoints — pwndbg — Heap inspection |
| **074** | Buffer Overflow Basics | Stack smashing — NX/ASLR — Return address overwrite |
| **075** | Shellcode Writing | Position-independent code — Null-free shellcode — Encoding |
| **076** | Return-Oriented Programming (ROP) | Gadgets — ROP chains — ret2libc — ASLR bypass |
| **077** | Malware Analysis | Unpacking — Anti-VM — Persistence mechanisms |
| **078** | Anti-Debugging Techniques | PTRACE detection — Timing attacks — IsDebuggerPresent |
| **079** | Obfuscation Techniques | Control flow flattening — Opaque predicates — Encryption |
| **080** | CTF Challenges | Binary exploitation — Pwn techniques — Challenge walkthrough |

---

### SECTION 9: OS Development (Parts 81–90)

> สร้างระบบปฏิบัติการของตัวเองตั้งแต่บรรทัดแรก

| Part | ชื่อเรื่อง | เนื้อหาสำคัญ |
|------|-----------|------------|
| **081** | Bootloader Basics | MBR — BIOS boot process — Stage 1/2 bootloaders |
| **082** | Real Mode Programming | 16-bit registers — BIOS interrupts — Memory map |
| **083** | Protected Mode | GDT — Segment descriptors — Privilege levels (rings) |
| **084** | Long Mode (64-bit) | PAE — IA-32e mode — Switching to 64-bit |
| **085** | GDT/IDT Setup | Descriptor tables — TSS — Gate descriptors |
| **086** | Interrupt Handlers | PIC — APIC — Timer, Keyboard, Spurious interrupts |
| **087** | Memory Paging | Page directory/tables — TLB — Identity mapping |
| **088** | Kernel Basics | Kernel entry — C kernel — VGA framebuffer — Serial output |
| **089** | Process Scheduler | Context switching — Round-robin — Priority scheduling |
| **090** | Mini OS Project | Integrated project — Shell — File system basics |

---

### SECTION 10: Expert Level & World-Class (Parts 91–100)

> ระดับสูงสุด — สร้าง, วิเคราะห์ และนวัตกรรมระดับโลก

| Part | ชื่อเรื่อง | เนื้อหาสำคัญ |
|------|-----------|------------|
| **091** | Compiler Design | Lexer/Parser/Code gen — Assembly backend — SSA form |
| **092** | JIT Compilation | Runtime code generation — Assembly injection — Trampoline |
| **093** | Virtualization Basics | VM concepts — CPUID — VMX operations |
| **094** | Hypervisor Development | VT-x/AMD-V — VMCS — Guest/Host transitions |
| **095** | Hardware Interfaces | MMIO — PCI enumeration — USB protocol |
| **096** | Embedded Systems | Bare-metal ARM — Peripheral control — RTOS basics |
| **097** | Real-Time Systems | Deterministic execution — Interrupt latency — RT scheduling |
| **098** | Cryptography in Assembly | AES-NI — SHA extensions — Constant-time code |
| **099** | Contributing to Open Source | LLVM backend — Linux kernel patches — libc contributions |
| **100** | Capstone Project | Full system — Bootloader + OS + Apps + Documentation |

---

## สิ่งที่คุณจะได้เรียนรู้ (Learning Outcomes)

### หลังจากเรียนจบหลักสูตรนี้ คุณจะสามารถ:

#### Technical Skills
```
[x] เขียนโปรแกรม x86-64 Assembly ได้อย่างคล่องแคล่ว
[x] เขียนโปรแกรม ARM32/ARM64 Assembly ได้อย่างคล่องแคล่ว
[x] ใช้ SIMD instructions (SSE/AVX/NEON) เพื่อ optimize performance
[x] เขียน system programs ที่ทำงานโดยตรงกับ OS
[x] Reverse engineer binary executables
[x] เขียน shellcode และ exploits (สำหรับ ethical hacking)
[x] สร้าง bootloader และ minimal OS
[x] Optimize code ได้เหนือกว่า compiler ทั่วไป
[x] วิเคราะห์ malware ในระดับ binary
[x] เขียน JIT compiler
```

#### Deep Understanding
```
[x] เข้าใจว่า CPU ทำงานอย่างไรในระดับ transistor
[x] เข้าใจ memory hierarchy และ cache behavior
[x] เข้าใจ ABI และ calling conventions ทุก platform
[x] เข้าใจ virtual memory และ paging
[x] เข้าใจ OS internals จาก assembly perspective
[x] เข้าใจ security vulnerabilities ในระดับ binary
```

---

## วิธีใช้หลักสูตรนี้ (How to Use This Course)

### สำหรับผู้เริ่มต้น (Beginners)
1. เริ่มจาก **Part 001** และทำตาม step-by-step
2. อ่านทุก concept และลองพิมพ์โค้ดด้วยตัวเอง (อย่า copy-paste)
3. ทำแบบฝึกหัดทุกข้อก่อนไปต่อ
4. ถ้าไม่เข้าใจ ให้กลับไปอ่าน part ก่อนหน้าอีกครั้ง

### สำหรับผู้มีประสบการณ์ (Intermediate)
1. ข้ามไปที่ section ที่ต้องการได้
2. ใช้เป็น reference เพื่อ deepen knowledge
3. โฟกัสที่ optimization sections (61-70)
4. ลองทำ CTF challenges ใน Section 8

### สำหรับผู้เชี่ยวชาญ (Advanced/Expert)
1. โฟกัสที่ Section 9 (OS Development) และ Section 10 (Expert)
2. ใช้ course นี้เพื่อสร้าง world-class projects
3. Contribute กลับมาที่หลักสูตร
4. เชื่อมต่อกับ community

### Tips สำคัญ
```
IMPORTANT TIPS:
1. ใช้ REAL hardware เมื่อทำได้ — อย่าใช้แค่ emulator
2. Debug ทุกอย่างด้วย GDB — เรียนรู้จาก registers/memory จริง
3. อ่าน Intel Manual และ ARM ARM (Architecture Reference Manual)
4. เข้าร่วม community: OSDev, Reddit r/asm, Discord servers
5. สร้าง project จริงๆ อย่างน้อย 1 project ต่อ section
6. เขียน notes ด้วยลายมือ — ช่วย remember ได้ดีกว่า
```

---

## เครื่องมืออ้างอิงสำคัญ (Key References)

### Official Documentation
- [Intel 64 and IA-32 Architectures Software Developer Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [ARM Architecture Reference Manual (ARM ARM)](https://developer.arm.com/documentation/)
- [NASM Documentation](https://www.nasm.us/doc/)
- [GAS (GNU Assembler) Documentation](https://sourceware.org/binutils/docs/as/)
- [Linux Syscall Reference](https://syscalls.kernelgrok.com/)
- [X86-64 ABI Specification](https://gitlab.com/x86-psABIs/x86-64-ABI)

### เว็บไซต์ที่มีประโยชน์
- **Compiler Explorer (Godbolt)**: https://godbolt.org/ — ดูว่า compiler สร้าง assembly อะไร
- **Felix Cloutier x86 Reference**: https://www.felixcloutier.com/x86/
- **OSDev Wiki**: https://wiki.osdev.org/
- **Agner Fog's Optimization Manuals**: https://www.agner.org/optimize/

### หนังสือแนะนำ
1. "Programming from the Ground Up" — Jonathan Bartlett (Free PDF)
2. "Computer Systems: A Programmer's Perspective" — Bryant & O'Hallaron
3. "The Art of Assembly Language" — Randall Hyde
4. "Professional Assembly Language" — Richard Blum
5. "Hacking: The Art of Exploitation" — Jon Erickson
6. "Modern X86 Assembly Language Programming" — Daniel Kusswurm

---

## Community และการสนับสนุน (Community & Support)

### ช่องทางการเรียนรู้เพิ่มเติม
- **Stack Overflow**: tag `assembly`, `x86`, `arm`
- **Reddit**: r/asm, r/osdev, r/netsec, r/ReverseEngineering
- **OSDev Forums**: https://forum.osdev.org/
- **CTF Competitions**: picoCTF, pwnable.kr, exploit.education

---

## สรุป Roadmap (Learning Roadmap Summary)

```
WEEK 1-4:    Parts 001-010  → Binary/Hex, CPU, Registers, Basic Programs
WEEK 5-8:    Parts 011-020  → Data Manipulation, Stack, I/O, C Integration  
WEEK 9-12:   Parts 021-030  → Functions, Calling Conventions, Modular Code
WEEK 13-16:  Parts 031-040  → ARM Assembly (Mobile/Embedded)
WEEK 17-20:  Parts 041-050  → SIMD/FPU (Multimedia/Math Optimization)
WEEK 21-24:  Parts 051-060  → System Programming (OS Interface)
WEEK 25-28:  Parts 061-070  → Performance Optimization (World-class Speed)
WEEK 29-32:  Parts 071-080  → Security & Reverse Engineering
WEEK 33-36:  Parts 081-090  → OS Development (Build Your Own OS)
WEEK 37-40:  Parts 091-100  → Expert/World-Class Projects

TOTAL: ~10 months for complete mastery
       ~3 months for foundations (Parts 1-30)
       ~6 months for professional level (Parts 1-70)
```

---

## การวัดผลและ Milestones

### Milestone 1: Foundation (Part 10)
- [ ] เขียน Hello World ด้วย NASM ได้
- [ ] เข้าใจ registers ทุกตัว
- [ ] ใช้ system calls ได้

### Milestone 2: Intermediate (Part 30)
- [ ] เขียน functions ที่ compatible กับ C ได้
- [ ] Implement data structures ใน assembly ได้
- [ ] Link assembly กับ C code ได้

### Milestone 3: Advanced (Part 60)
- [ ] เขียน ARM code ได้
- [ ] ใช้ SIMD instructions ได้
- [ ] เขียน system-level programs ได้

### Milestone 4: Expert (Part 90)
- [ ] สร้าง bootloader ได้
- [ ] เขียน kernel code ได้
- [ ] Reverse engineer binaries ได้

### Milestone 5: World-Class (Part 100)
- [ ] สร้าง mini OS ได้
- [ ] Contribute to open source projects ได้
- [ ] เป็นผู้เชี่ยวชาญ Assembly ระดับสากล

---

## License และ Credits

```
หลักสูตรนี้สร้างขึ้นเพื่อการศึกษา (Educational Purpose)
เนื้อหาทั้งหมดสามารถนำไปใช้เพื่อการเรียนรู้ได้อย่างอิสระ

ขอให้สนุกกับการเรียนรู้ Assembly!
"The closer to the metal, the more power you have."
```

---

*หลักสูตรนี้อัพเดทอย่างต่อเนื่อง — Parts ใหม่จะถูกเพิ่มเรื่อยๆ จนครบ 100 Parts และมากกว่า*

**เริ่มต้นการเดินทางสู่ความเป็น World-Class Assembly Programmer ได้เลย!**

[→ ไปที่ Part 001: Binary, Hex และ Data Representation](./section_01_foundations/part_001_binary_hex.md)
