# Part 027: NASM Macros อย่างละเอียด (NASM Macros In Depth)

## บทนำ (Introduction)

Macro เป็นหนึ่งในเครื่องมือที่ทรงพลังที่สุดของ NASM (Netwide Assembler) ช่วยให้เราสามารถ
เขียนโค้ดที่ซ้ำกันได้อย่างมีประสิทธิภาพ ลดข้อผิดพลาด และทำให้โค้ด Assembly อ่านง่ายขึ้น
เหมือนกับ Preprocessor Macro ใน C แต่มีความสามารถที่ลึกและยืดหยุ่นกว่ามาก

### ทำไมต้องใช้ Macro?

```
โดยไม่ใช้ Macro:                  โดยใช้ Macro:
push rbp                          PROC_ENTER
mov rbp, rsp                      ; เรียบร้อย! 1 บรรทัด
sub rsp, 64
; ... ซ้ำทุก function
```

### ประเภทของ Macro ใน NASM

| ประเภท | คำสั่ง | คำอธิบาย |
|--------|--------|----------|
| Simple substitution | `%define` | แทนที่ข้อความง่ายๆ |
| Numeric assignment | `%assign` | กำหนดค่าตัวเลข |
| Single-line with args | `%define name(args)` | Macro บรรทัดเดียวมี parameter |
| Multi-line | `%macro/%endmacro` | Macro หลายบรรทัด |
| Conditional | `%if/%ifdef` | Macro แบบมีเงื่อนไข |
| Repetition | `%rep/%endrep` | ทำซ้ำ |

---

## หัวข้อที่ 1: %define — Simple Substitution Macros

`%define` ทำงานเหมือน `#define` ใน C โดยทำการแทนที่ข้อความก่อนการ assemble

### รูปแบบพื้นฐาน

```nasm
%define ชื่อ ค่าที่จะแทนที่
```

### ตัวอย่างที่ 1: Constants และ Register Aliases

```nasm
; ไฟล์: define_basic.asm
; คอมไพล์: nasm -f elf64 define_basic.asm -o define_basic.o
;           ld define_basic.o -o define_basic

section .data
    msg     db "Hello, NASM Macros!", 10
    msglen  equ $ - msg

section .text
    global _start

; ========================================
; %define สำหรับ System Call Numbers (Linux x86-64)
; ========================================
%define SYS_WRITE   1
%define SYS_EXIT    60
%define STDOUT      1
%define STDIN       0
%define STDERR      2

; ========================================
; %define สำหรับ Register Aliases
; ========================================
%define argc    rdi         ; argument count
%define argv    rsi         ; argument vector
%define retval  rax         ; return value (convention)

; ========================================
; %define สำหรับ Common Values
; ========================================
%define NULL        0
%define TRUE        1
%define FALSE       0
%define EXIT_OK     0
%define EXIT_FAIL   1

; ========================================
; %define สำหรับ Memory Sizes
; ========================================
%define BYTE_SIZE   1
%define WORD_SIZE   2
%define DWORD_SIZE  4
%define QWORD_SIZE  8

; ========================================
; %define สำหรับ Buffer Sizes
; ========================================
%define SMALL_BUF   64
%define MED_BUF     256
%define LARGE_BUF   1024
%define HUGE_BUF    4096

_start:
    ; ใช้ SYS_WRITE แทนตัวเลข 1 โดยตรง
    mov rax, SYS_WRITE      ; ชัดเจนกว่า: mov rax, 1
    mov rdi, STDOUT         ; ชัดเจนกว่า: mov rdi, 1
    mov rsi, msg
    mov rdx, msglen
    syscall

    ; ออกจากโปรแกรม
    mov rax, SYS_EXIT
    mov rdi, EXIT_OK        ; ชัดเจนกว่า: mov rdi, 0
    syscall
```

### ตัวอย่างที่ 2: %define สำหรับ Complex Expressions

```nasm
; ไฟล์: define_complex.asm
; แสดงการใช้ %define กับ expression ที่ซับซ้อน

; ========================================
; Memory addressing macros
; ========================================
%define byte_ptr(addr)      byte [addr]
%define word_ptr(addr)      word [addr]
%define dword_ptr(addr)     dword [addr]
%define qword_ptr(addr)     qword [addr]

; ========================================
; Stack frame access macros
; ========================================
%define local(offset)       [rbp - offset]
%define arg(offset)         [rbp + offset + 16]   ; +16 เพื่อข้าม return address และ saved rbp

; ========================================
; Array access macro
; ========================================
%define array_elem(base, idx, size)   [base + idx*size]

section .bss
    buffer  resb 256        ; จอง 256 bytes

section .data
    values  dd 10, 20, 30, 40, 50   ; array ของ 32-bit integers

section .text
    global _start

_start:
    ; ตัวอย่างการใช้ dword_ptr
    mov eax, dword_ptr(values)      ; อ่านค่าแรก = 10

    ; ตัวอย่าง array access
    mov ecx, 2                      ; index = 2
    mov eax, array_elem(values, ecx, 4)  ; values[2] = 30

    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

### ข้อควรระวัง: %define ไม่มี Scope

```nasm
; %define มีผลตลอดไฟล์ (global scope)
%define VAL 100

section .text
_start:
    mov rax, VAL        ; = 100

; แต่สามารถ redefine ได้
%define VAL 200
    mov rbx, VAL        ; = 200 (ค่าใหม่)

; และสามารถ undefine ได้
%undef VAL
; mov rcx, VAL        ; จะ error! VAL ไม่มีแล้ว
```

---

## หัวข้อที่ 2: %assign — Numeric Assignment

`%assign` คล้าย `%define` แต่ใช้สำหรับค่าตัวเลขโดยเฉพาะ และสามารถ
ทำการคำนวณได้ในขณะ assemble (preprocessor time)

### ความแตกต่างระหว่าง %define และ %assign

```nasm
%define  FOO  1+2     ; FOO แทนที่ด้วย "1+2" (ยังไม่คำนวณ)
%assign  BAR  1+2     ; BAR = 3 (คำนวณทันที)

; ผลต่าง:
mov rax, FOO * 3    ; = (1+2) * 3 = 9  (ถูกต้อง แต่ขึ้นกับ operator precedence)
mov rax, BAR * 3    ; = 3 * 3 = 9      (ชัดเจนกว่า)

; ปัญหากับ %define:
mov rax, FOO + FOO  ; = 1+2 + 1+2 = 6  (ดูเหมือนตรง)
; แต่: mov rax, FOO * FOO = 1+2 * 1+2 = 1+2+2 = 5 !! (ไม่ใช่ 9!)
; ในขณะที่ %assign:
mov rax, BAR * BAR  ; = 3 * 3 = 9      (ถูกต้อง)
```

### ตัวอย่าง: Counter และ Enum ด้วย %assign

```nasm
; ไฟล์: assign_demo.asm
; คอมไพล์: nasm -f elf64 assign_demo.asm -o assign_demo.o && ld assign_demo.o -o assign_demo

; ========================================
; สร้าง Enum ด้วย %assign
; ========================================
%assign COLOR_RED       0
%assign COLOR_GREEN     1
%assign COLOR_BLUE      2
%assign COLOR_YELLOW    3
%assign COLOR_WHITE     4
%assign COLOR_BLACK     5
%assign COLOR_COUNT     6       ; จำนวนสีทั้งหมด

; ========================================
; สร้าง Flags ด้วย %assign (bit positions)
; ========================================
%assign FLAG_CARRY      1       ; bit 0
%assign FLAG_ZERO       2       ; bit 1
%assign FLAG_SIGN       4       ; bit 2
%assign FLAG_OVERFLOW   8       ; bit 3
%assign FLAG_ALL        15      ; ทุก flag (OR ของทั้งหมด)

; ========================================
; คำนวณค่าที่เกี่ยวข้องกัน
; ========================================
%assign PAGE_SIZE       4096
%assign PAGES           8
%assign TOTAL_MEM       PAGE_SIZE * PAGES   ; = 32768

%assign KB              1024
%assign MB              KB * KB             ; = 1048576
%assign STACK_SIZE      2 * MB              ; = 2097152

; ========================================
; Bit shift calculations
; ========================================
%assign BIT0    1 << 0      ; = 1
%assign BIT1    1 << 1      ; = 2
%assign BIT2    1 << 2      ; = 4
%assign BIT7    1 << 7      ; = 128
%assign BIT31   1 << 31     ; = 2147483648

section .data
    ; ใช้ค่าจาก %assign ใน data section
    color_count     dd COLOR_COUNT      ; = 6
    total_memory    dq TOTAL_MEM        ; = 32768
    stack_sz        dq STACK_SIZE       ; = 2097152

section .text
    global _start

_start:
    ; ทดสอบ flags
    mov eax, FLAG_CARRY | FLAG_ZERO     ; OR ของ flags = 3
    test eax, FLAG_CARRY                ; ตรวจสอบ carry flag
    jnz .carry_set

    jmp .done

.carry_set:
    ; carry flag ถูก set

.done:
    mov rax, 60
    xor rdi, rdi
    syscall
```

### %assign กับ Counting

```nasm
; นับจำนวน macro calls ด้วย %assign
%assign call_count 0

%macro COUNT_CALL 0
    %assign call_count call_count + 1
%endmacro

COUNT_CALL      ; call_count = 1
COUNT_CALL      ; call_count = 2
COUNT_CALL      ; call_count = 3

; ใช้ค่า call_count
section .data
    num_calls dd call_count     ; = 3
```

---

## หัวข้อที่ 3: Single-line Macros with Arguments

Macro บรรทัดเดียวที่รับ parameter ทำให้ยืดหยุ่นมากขึ้น

```nasm
%define macro_name(param1, param2, ...)  ผลลัพธ์ที่ใช้ param
```

### ตัวอย่างที่ 1: Function-like Macros

```nasm
; ไฟล์: single_line_macros.asm

; ========================================
; Arithmetic macros
; ========================================
%define MIN(a,b)        (((a) < (b)) ? (a) : (b))   ; ไม่ได้ใช้ใน code แต่ใช้ใน data
%define MAX(a,b)        (((a) > (b)) ? (a) : (b))
%define ABS(x)          (((x) >= 0) ? (x) : -(x))
%define SQUARE(x)       ((x) * (x))
%define CUBE(x)         ((x) * (x) * (x))

; ========================================
; Alignment macros
; ========================================
%define ALIGN_UP(val, align)    (((val) + (align) - 1) & ~((align) - 1))
%define ALIGN_DOWN(val, align)  ((val) & ~((align) - 1))
%define IS_ALIGNED(val, align)  (((val) & ((align) - 1)) == 0)

; ========================================
; Bit manipulation macros
; ========================================
%define SET_BIT(val, bit)       ((val) | (1 << (bit)))
%define CLEAR_BIT(val, bit)     ((val) & ~(1 << (bit)))
%define TOGGLE_BIT(val, bit)    ((val) ^ (1 << (bit)))
%define CHECK_BIT(val, bit)     (((val) >> (bit)) & 1)
%define MASK(bits)              ((1 << (bits)) - 1)

; ========================================
; Memory offset macros
; ========================================
%define FIELD_OFFSET(struct_name, field)    struct_name %+ . %+ field

; ========================================
; Register operation shortcuts
; ========================================
%define LOAD_ADDR(reg, label)   lea reg, [rel label]
%define ZERO_REG(reg)           xor reg, reg

section .data
    ; ใช้ macros ใน data section
    sq5     dd SQUARE(5)        ; = 25
    cu3     dd CUBE(3)          ; = 27
    align64 dq ALIGN_UP(100, 64)    ; = 128
    mask4   dq MASK(4)          ; = 15 (0b1111)

section .text
    global _start

_start:
    ; ใช้ ZERO_REG macro
    ZERO_REG(rax)               ; xor rax, rax

    ; ใช้ LOAD_ADDR macro
    LOAD_ADDR(rsi, sq5)         ; lea rsi, [rel sq5]

    ; ใช้ SET_BIT ใน constant expression
    mov rax, SET_BIT(0, 3)      ; rax = 8

    mov rax, 60
    xor rdi, rdi
    syscall
```

### ตัวอย่างที่ 2: Struct-like Access Macros

```nasm
; ไฟล์: struct_macros.asm
; สร้าง struct โดยใช้ macro

; ========================================
; กำหนด struct Point { x: int32, y: int32, z: int32 }
; ========================================
%define Point.x     0           ; offset ของ x
%define Point.y     4           ; offset ของ y
%define Point.z     8           ; offset ของ z
%define Point.size  12          ; ขนาดของ struct

; Accessor macros
%define pt_x(base)  dword [base + Point.x]
%define pt_y(base)  dword [base + Point.y]
%define pt_z(base)  dword [base + Point.z]

; ========================================
; กำหนด struct Rectangle
; ========================================
%define Rect.x      0
%define Rect.y      4
%define Rect.w      8
%define Rect.h      12
%define Rect.size   16

%define rect_x(base)    dword [base + Rect.x]
%define rect_y(base)    dword [base + Rect.y]
%define rect_w(base)    dword [base + Rect.w]
%define rect_h(base)    dword [base + Rect.h]

section .bss
    point1  resb Point.size     ; จอง space สำหรับ 1 Point
    rect1   resb Rect.size      ; จอง space สำหรับ 1 Rectangle

section .text
    global _start

_start:
    ; ตั้งค่า point1
    mov pt_x(point1), 10        ; point1.x = 10
    mov pt_y(point1), 20        ; point1.y = 20
    mov pt_z(point1), 30        ; point1.z = 30

    ; อ่านค่า
    mov eax, pt_x(point1)       ; eax = 10
    add eax, pt_y(point1)       ; eax = 30
    add eax, pt_z(point1)       ; eax = 60

    ; ตั้งค่า rect1
    mov rect_x(rect1), 0
    mov rect_y(rect1), 0
    mov rect_w(rect1), 800
    mov rect_h(rect1), 600

    ; คำนวณพื้นที่ (area = w * h)
    mov eax, rect_w(rect1)      ; eax = 800
    imul eax, rect_h(rect1)     ; eax = 480000

    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## หัวข้อที่ 4: Multi-line Macros — %macro/%endmacro

Multi-line macros คือ macro ที่มีหลายบรรทัด เป็นแกนหลักของการใช้ macro ใน NASM

### รูปแบบ

```nasm
%macro ชื่อ จำนวน_arguments
    ; เนื้อหา macro
    ; ใช้ %1, %2, %3, ... สำหรับ arguments
%endmacro
```

### ตัวอย่างที่ 1: System Call Wrapper Macros

```nasm
; ไฟล์: multiline_macros.asm
; คอมไพล์: nasm -f elf64 multiline_macros.asm -o multiline_macros.o
;           ld multiline_macros.o -o multiline_macros

; ========================================
; System call macros
; ========================================

; sys_write(fd, buf, len)
%macro sys_write 3
    mov rax, 1          ; SYS_WRITE
    mov rdi, %1         ; file descriptor
    mov rsi, %2         ; buffer address
    mov rdx, %3         ; length
    syscall
%endmacro

; sys_read(fd, buf, len)
%macro sys_read 3
    mov rax, 0          ; SYS_READ
    mov rdi, %1         ; file descriptor
    mov rsi, %2         ; buffer address
    mov rdx, %3         ; max length
    syscall
%endmacro

; sys_exit(code)
%macro sys_exit 1
    mov rax, 60         ; SYS_EXIT
    mov rdi, %1         ; exit code
    syscall
%endmacro

; print_str(label, length)
%macro print_str 2
    sys_write 1, %1, %2
%endmacro

section .data
    hello   db "Hello, World!", 10
    hlen    equ $ - hello

    msg1    db "NASM Macros are powerful!", 10
    mlen1   equ $ - msg1

section .text
    global _start

_start:
    ; ใช้ macro แทนการเขียน syscall เอง
    print_str hello, hlen
    print_str msg1, mlen1

    sys_exit 0
```

### ตัวอย่างที่ 2: Stack Frame Macros

```nasm
; ไฟล์: stack_frame_macros.asm
; คอมไพล์: nasm -f elf64 stack_frame_macros.asm -o stack_frame_macros.o
;           ld stack_frame_macros.o -o stack_frame_macros

; ========================================
; Function prologue/epilogue macros
; ========================================

; ตั้งค่า stack frame พร้อมจอง local variables
; PROC_BEGIN local_size
%macro PROC_BEGIN 1
    push rbp                    ; บันทึก base pointer
    mov rbp, rsp                ; ตั้ง base pointer ใหม่
    sub rsp, %1                 ; จอง space สำหรับ local variables
    ; จัดตำแหน่ง stack ให้ align 16 bytes
    and rsp, -16
%endmacro

; ทำความสะอาด stack frame และ return
%macro PROC_END 0
    mov rsp, rbp                ; คืน stack pointer
    pop rbp                     ; คืน base pointer
    ret                         ; return
%endmacro

; PROC_BEGIN_SAVE_REGS: บันทึก callee-saved registers
%macro PROC_BEGIN_SAVE 1
    push rbp
    mov rbp, rsp
    sub rsp, %1
    and rsp, -16
    ; บันทึก callee-saved registers (System V ABI)
    push rbx
    push r12
    push r13
    push r14
    push r15
%endmacro

%macro PROC_END_SAVE 0
    ; คืน callee-saved registers (ลำดับย้อนกลับ)
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    mov rsp, rbp
    pop rbp
    ret
%endmacro

; ========================================
; ตัวอย่างการใช้งาน
; ========================================

section .data
    result_fmt  db "Result: ", 0

section .bss
    output  resb 32

section .text
    global _start
    global compute_sum
    global compute_product

; ฟังก์ชัน: compute_sum(a, b) -> a + b
; parameters: rdi=a, rsi=b
; return: rax
compute_sum:
    PROC_BEGIN 0            ; ไม่ต้องการ local variables

    mov rax, rdi            ; rax = a
    add rax, rsi            ; rax = a + b

    PROC_END

; ฟังก์ชัน: compute_product(a, b) -> a * b
; parameters: rdi=a, rsi=b
; return: rax
compute_product:
    PROC_BEGIN 16           ; จอง 16 bytes local

    mov rax, rdi            ; rax = a
    imul rax, rsi           ; rax = a * b

    PROC_END

; ฟังก์ชัน: complex_calc(x, y, z) -> (x + y) * z
; parameters: rdi=x, rsi=y, rdx=z
complex_calc:
    PROC_BEGIN_SAVE 32      ; จอง 32 bytes, บันทึก regs

    mov r12, rdx            ; บันทึก z ไว้ใน callee-saved reg

    ; เรียก compute_sum(x, y)
    ; rdi, rsi ยังมีค่า x, y อยู่
    call compute_sum        ; rax = x + y

    ; เรียก compute_product(sum, z)
    mov rdi, rax            ; rdi = x + y
    mov rsi, r12            ; rsi = z
    call compute_product    ; rax = (x + y) * z

    PROC_END_SAVE

_start:
    ; ทดสอบ compute_sum(3, 4)
    mov rdi, 3
    mov rsi, 4
    call compute_sum        ; rax = 7

    ; ทดสอบ compute_product(6, 7)
    mov rdi, 6
    mov rsi, 7
    call compute_product    ; rax = 42

    ; ทดสอบ complex_calc(2, 3, 4)
    mov rdi, 2
    mov rsi, 3
    mov rdx, 4
    call complex_calc       ; rax = (2+3)*4 = 20

    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## หัวข้อที่ 5: Local Labels in Macros — %%label

ปัญหาใหญ่ของ multi-line macros คือถ้า macro มี label และเรียกใช้หลายครั้ง
จะเกิด label ซ้ำกัน ทำให้ assembler error `%%label` แก้ปัญหานี้

### ปัญหาที่เกิดขึ้นโดยไม่ใช้ %%

```nasm
; WRONG! - เรียก macro นี้ 2 ครั้งจะเกิด "duplicate label: .loop"
%macro BAD_LOOP 2
.loop:                      ; label ซ้ำถ้าเรียกหลายครั้ง!
    add %1, 1
    cmp %1, %2
    jl .loop
%endmacro
```

### การแก้ปัญหาด้วย %%label

```nasm
; ไฟล์: local_labels.asm
; คอมไพล์: nasm -f elf64 local_labels.asm -o local_labels.o && ld local_labels.o -o local_labels

; ========================================
; Loop macro ที่ถูกต้อง ใช้ %%label
; ========================================

; REPEAT n, reg: ทำซ้ำ n ครั้ง โดย reg เป็น counter
%macro REPEAT 2
    xor %2, %2              ; counter = 0
%%loop_start:
    ; เนื้อหาจะถูกแทรกตรงนี้โดย caller ไม่ได้
    ; (นี้เป็น skeleton สำหรับ counting)
    inc %2                  ; counter++
    cmp %2, %1              ; counter < n?
    jl %%loop_start         ; ถ้าใช่ วนซ้ำ
%endmacro

; COUNT_TO macro: นับ 0 ถึง n
%macro COUNT_TO 1
    xor rcx, rcx            ; rcx = 0
%%count_loop:
    ; ตรงนี้ทำอะไรก็ได้กับ rcx
    inc rcx
    cmp rcx, %1
    jl %%count_loop
%endmacro

; ========================================
; MAX macro ที่ถูกต้อง
; ========================================
; MAX dest, src: dest = max(dest, src)
%macro MAX 2
    cmp %1, %2
    jge %%already_max       ; ถ้า dest >= src ไม่ต้องทำอะไร
    mov %1, %2              ; dest = src
%%already_max:
%endmacro

; MIN dest, src: dest = min(dest, src)
%macro MIN 2
    cmp %1, %2
    jle %%already_min       ; ถ้า dest <= src ไม่ต้องทำอะไร
    mov %1, %2              ; dest = src
%%already_min:
%endmacro

; ========================================
; ABS_VAL macro
; ========================================
; ABS_VAL reg: reg = |reg|
%macro ABS_VAL 1
    test %1, %1             ; ตรวจสอบ sign
    jns %%positive          ; ถ้า positive ข้ามไป
    neg %1                  ; ถ้า negative, negate
%%positive:
%endmacro

; ========================================
; SAFE_DIV macro
; ========================================
; SAFE_DIV dividend, divisor, result_reg
; ถ้า divisor = 0 ให้ result = 0
%macro SAFE_DIV 3
    test %2, %2             ; ตรวจสอบ divisor
    jz %%div_zero           ; ถ้า 0 ข้ามไป
    mov rax, %1
    cqo                     ; sign-extend rax เป็น rdx:rax
    idiv %2                 ; หาร
    mov %3, rax             ; เก็บผลลัพธ์
    jmp %%div_done
%%div_zero:
    xor %3, %3              ; result = 0
%%div_done:
%endmacro

section .text
    global _start

_start:
    ; ทดสอบ MAX macro
    mov rax, 10
    mov rbx, 20
    MAX rax, rbx            ; rax = 20

    ; ทดสอบ MAX อีกครั้ง (%%label สร้าง label ใหม่โดยอัตโนมัติ)
    mov rax, 50
    mov rbx, 30
    MAX rax, rbx            ; rax = 50

    ; ทดสอบ MIN
    mov rax, 10
    mov rbx, 5
    MIN rax, rbx            ; rax = 5

    ; ทดสอบ ABS_VAL
    mov rax, -42
    ABS_VAL rax             ; rax = 42

    mov rax, 15
    ABS_VAL rax             ; rax = 15 (ไม่เปลี่ยน)

    ; ทดสอบ SAFE_DIV
    mov rax, 100
    mov rbx, 5
    SAFE_DIV rax, rbx, rcx  ; rcx = 20

    mov rax, 100
    xor rbx, rbx            ; divisor = 0
    SAFE_DIV rax, rbx, rcx  ; rcx = 0 (ปลอดภัย ไม่ crash)

    ; COUNT_TO เรียก 2 ครั้ง ไม่มี label conflict
    COUNT_TO 10
    COUNT_TO 5

    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## หัวข้อที่ 6: Default Arguments

NASM อนุญาตให้ macro มี default arguments สำหรับ parameter ที่ไม่ได้ระบุ

```nasm
; รูปแบบ: %macro name min_args-max_args
;         หรือ:  %macro name min_args+  (รับได้ไม่จำกัด)
```

### ตัวอย่างการใช้ Default Arguments

```nasm
; ไฟล์: default_args.asm
; คอมไพล์: nasm -f elf64 default_args.asm -o default_args.o && ld default_args.o -o default_args

; ========================================
; Macro ที่รับ 1-3 arguments
; ========================================
; MOVE_DATA dest, src [, size]
; size: 1=byte, 2=word, 4=dword, 8=qword (default=8)
%macro MOVE_DATA 2-3
    %if %0 == 3
        ; ขนาดถูกระบุมา
        %if %3 == 1
            mov byte [%1], byte [%2]
        %elif %3 == 2
            mov ax, word [%2]
            mov word [%1], ax
        %elif %3 == 4
            mov eax, dword [%2]
            mov dword [%1], eax
        %else
            mov rax, qword [%2]
            mov qword [%1], rax
        %endif
    %else
        ; ไม่ได้ระบุขนาด ใช้ default = 8 bytes (qword)
        mov rax, qword [%2]
        mov qword [%1], rax
    %endif
%endmacro

; ========================================
; PRINT macro ที่รับ 1-2 args
; ========================================
; PRINT buffer [, length]
; ถ้าไม่ระบุ length ใช้ null-terminated string
%macro PRINT 1-2
    %if %0 == 2
        ; length ถูกระบุมา
        mov rax, 1          ; sys_write
        mov rdi, 1          ; stdout
        mov rsi, %1         ; buffer
        mov rdx, %2         ; length
        syscall
    %else
        ; ต้องหา length เอง (null-terminated)
        ; บันทึก registers
        push rdi
        push rsi
        push rcx
        push rax

        ; หา string length
        mov rdi, %1         ; pointer ไปยัง string
        xor rcx, rcx        ; counter = 0
        xor al, al          ; ค้นหา null byte
        mov rsi, rdi        ; บันทึก start pointer
%%find_null:
        cmp byte [rdi], 0
        je %%found_null
        inc rdi
        inc rcx
        jmp %%find_null
%%found_null:

        ; print ด้วย length ที่หามาได้
        mov rdx, rcx        ; length
        mov rsi, %1         ; buffer
        mov rdi, 1          ; stdout
        mov rax, 1          ; sys_write
        syscall

        ; คืน registers
        pop rax
        pop rcx
        pop rsi
        pop rdi
    %endif
%endmacro

; ========================================
; PUSH_MULTIPLE: push หลาย register
; ========================================
%macro PUSH_MULTIPLE 1-8
    %rep %0
        push %1
        %rotate 1
    %endrep
%endmacro

%macro POP_MULTIPLE 1-8
    %rep %0
        %rotate -1
        pop %1
    %endrep
%endmacro

section .data
    str1    db "Hello with null terminator", 0
    str2    db "Hello with explicit length", 10
    slen2   equ $ - str2

    val1    dq 0
    val2    dq 12345678

section .text
    global _start

_start:
    ; PRINT กับ null-terminated string
    PRINT str1              ; หา length เอง

    ; PRINT กับ explicit length
    PRINT str2, slen2       ; ระบุ length

    ; MOVE_DATA
    MOVE_DATA val1, val2    ; copy 8 bytes (default)
    ; val1 ควรเป็น 12345678 แล้ว

    ; PUSH/POP multiple registers
    PUSH_MULTIPLE rax, rbx, rcx, rdx
    ; ... ทำอะไรบางอย่าง
    POP_MULTIPLE rax, rbx, rcx, rdx

    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## หัวข้อที่ 7: Overloaded Macros (Multiple Argument Counts)

NASM ให้เราสร้าง macro ที่มีชื่อเดียวกันแต่รับ argument ต่างจำนวน

```nasm
; ตัวอย่าง: macro "ADD" ที่รับ 2 หรือ 3 arguments
%macro ADD 2
    add %1, %2
%endmacro

%macro ADD 3
    mov rax, %2
    add rax, %3
    mov %1, rax
%endmacro
```

### ตัวอย่าง: Overloaded PRINT Macro

```nasm
; ไฟล์: overloaded_macros.asm
; คอมไพล์: nasm -f elf64 overloaded_macros.asm -o overloaded_macros.o
;           ld overloaded_macros.o -o overloaded_macros

; ========================================
; LOG macro: 1, 2, หรือ 3 arguments
; ========================================

; LOG msg — พิมพ์ null-terminated string
%macro LOG 1
    push rax
    push rdi
    push rsi
    push rdx
    push rcx

    ; หาความยาว
    mov rsi, %1
    xor rdx, rdx
%%log1_len:
    cmp byte [rsi + rdx], 0
    je %%log1_print
    inc rdx
    jmp %%log1_len
%%log1_print:
    mov rax, 1
    mov rdi, 2          ; stderr
    syscall

    pop rcx
    pop rdx
    pop rsi
    pop rdi
    pop rax
%endmacro

; LOG fd, msg — พิมพ์ไปยัง file descriptor ที่ระบุ
%macro LOG 2
    push rax
    push rdi
    push rsi
    push rdx
    push rcx

    ; หาความยาว
    mov rsi, %2
    xor rdx, rdx
%%log2_len:
    cmp byte [rsi + rdx], 0
    je %%log2_print
    inc rdx
    jmp %%log2_len
%%log2_print:
    mov rax, 1
    mov rdi, %1         ; fd ที่ระบุ
    syscall

    pop rcx
    pop rdx
    pop rsi
    pop rdi
    pop rax
%endmacro

; LOG fd, msg, len — พิมพ์ด้วย length ที่รู้แล้ว
%macro LOG 3
    push rax
    push rdi
    push rsi
    push rdx

    mov rax, 1
    mov rdi, %1         ; fd
    mov rsi, %2         ; msg
    mov rdx, %3         ; len
    syscall

    pop rdx
    pop rsi
    pop rdi
    pop rax
%endmacro

; ========================================
; STORE macro: เก็บข้อมูลลงหน่วยความจำ
; ========================================

; STORE addr, value — เก็บ qword
%macro STORE 2
    mov qword [%1], %2
%endmacro

; STORE addr, value, size — เก็บตามขนาดที่ระบุ
%macro STORE 3
    %if %3 == 1
        mov byte [%1], %2
    %elif %3 == 2
        mov word [%1], %2
    %elif %3 == 4
        mov dword [%1], %2
    %else
        mov qword [%1], %2
    %endif
%endmacro

section .data
    msg_stderr  db "[ERROR] This is stderr", 10, 0
    msg_stdout  db "[INFO] This is stdout", 10, 0
    msg_len     db "[FAST] Quick message", 10
    msg_len_sz  equ $ - msg_len

section .bss
    storage8    resq 1
    storage4    resd 1
    storage1    resb 1

section .text
    global _start

_start:
    ; LOG ด้วย 1 argument (ไปยัง stderr โดย default)
    LOG msg_stderr

    ; LOG ด้วย 2 arguments (ระบุ fd)
    LOG 1, msg_stdout       ; fd=1 คือ stdout

    ; LOG ด้วย 3 arguments (ระบุ fd และ length)
    LOG 1, msg_len, msg_len_sz

    ; STORE ด้วย 2 arguments (qword default)
    STORE storage8, 0xDEADBEEF

    ; STORE ด้วย 3 arguments (ระบุขนาด)
    STORE storage4, 42, 4   ; เก็บ dword
    STORE storage1, 'A', 1  ; เก็บ byte

    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## หัวข้อที่ 8: %if/%elif/%else/%endif — Conditional Macros

ใช้สำหรับ conditional compilation — ตัดสินใจว่าจะสร้างโค้ดอะไรตอน assemble

```nasm
%if expression          ; ถ้า expression != 0
    ; code ที่สร้างถ้า true
%elif expression2       ; ถ้า expression2 != 0 (optional)
    ; code สำหรับกรณีนี้
%else                   ; ถ้าทุกเงื่อนไขเป็น false (optional)
    ; code สำหรับกรณี else
%endif
```

### ตัวอย่าง: Platform-specific Code

```nasm
; ไฟล์: conditional_macros.asm
; คอมไพล์: nasm -f elf64 -DPLATFORM=64 conditional_macros.asm -o cond.o && ld cond.o -o cond
; หรือ:    nasm -f elf64 -DDEBUG_MODE conditional_macros.asm -o cond.o && ld cond.o -o cond

; ========================================
; ตรวจสอบ platform
; ========================================
%ifndef PLATFORM
    %assign PLATFORM 64     ; default: 64-bit
%endif

%if PLATFORM == 64
    ; 64-bit specific macros
    %define WORD_TYPE   qword
    %define WORD_REG    rax
    %define WORD_SIZE   8
    %define STACK_ALIGN 16
    %define PTR_SIZE    8
%elif PLATFORM == 32
    ; 32-bit specific macros
    %define WORD_TYPE   dword
    %define WORD_REG    eax
    %define WORD_SIZE   4
    %define STACK_ALIGN 4
    %define PTR_SIZE    4
%else
    %error "ไม่รู้จัก PLATFORM ต้องเป็น 32 หรือ 64"
%endif

; ========================================
; Debug level macros
; ========================================
%ifndef DEBUG_LEVEL
    %assign DEBUG_LEVEL 0   ; default: no debug
%endif

%macro DPRINT 2
    %if DEBUG_LEVEL >= %1
        ; พิมพ์ debug message ที่ level >= ที่ระบุ
        push rax
        push rdi
        push rsi
        push rdx
        mov rax, 1
        mov rdi, 2          ; stderr
        mov rsi, %2
        ; หา length
        push rcx
        xor rdx, rdx
%%dprint_len:
        cmp byte [rsi + rdx], 0
        je %%dprint_go
        inc rdx
        jmp %%dprint_len
%%dprint_go:
        syscall
        pop rcx
        pop rdx
        pop rsi
        pop rdi
        pop rax
    %endif
%endmacro

; ========================================
; Optimization level macros
; ========================================
%ifndef OPT_LEVEL
    %assign OPT_LEVEL 1
%endif

; FAST_MULTIPLY x, y — คูณ โดยใช้ shift ถ้าเป็น power of 2
%macro FAST_MULTIPLY 2
    %if %2 == 1
        ; คูณด้วย 1 ไม่ต้องทำอะไร
    %elif %2 == 2
        sal %1, 1               ; x << 1 เร็วกว่า imul
    %elif %2 == 4
        sal %1, 2               ; x << 2
    %elif %2 == 8
        sal %1, 3               ; x << 3
    %elif %2 == 16
        sal %1, 4               ; x << 4
    %elif %2 == 32
        sal %1, 5               ; x << 5
    %elif %2 == 64
        sal %1, 6               ; x << 6
    %elif %2 == 128
        sal %1, 7               ; x << 7
    %elif %2 == 256
        sal %1, 8               ; x << 8
    %else
        ; ไม่ใช่ power of 2 ใช้ imul ปกติ
        imul %1, %2
    %endif
%endmacro

section .data
    dbg_start   db "[DEBUG:1] Program started", 10, 0
    dbg_calc    db "[DEBUG:2] Calculating...", 10, 0

section .text
    global _start

_start:
    ; Debug prints (จะแสดงก็ต่อเมื่อ DEBUG_LEVEL >= ระดับที่ระบุ)
    DPRINT 1, dbg_start     ; แสดงเมื่อ DEBUG_LEVEL >= 1
    DPRINT 2, dbg_calc      ; แสดงเมื่อ DEBUG_LEVEL >= 2

    ; FAST_MULTIPLY
    mov rax, 5
    FAST_MULTIPLY rax, 8    ; rax = 5 * 8 = 40 (ใช้ shift)

    mov rax, 7
    FAST_MULTIPLY rax, 3    ; rax = 7 * 3 = 21 (ใช้ imul)

    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## หัวข้อที่ 9: %ifdef/%ifndef — Existence Checks

ตรวจสอบว่า macro ถูกกำหนดแล้วหรือยัง

```nasm
%ifdef NAME        ; ถ้า NAME ถูกกำหนดแล้ว (ไม่สนค่า)
%ifndef NAME       ; ถ้า NAME ยังไม่ถูกกำหนด
```

### ตัวอย่าง: Include Guards และ Feature Flags

```nasm
; ไฟล์: ifdef_demo.asm
; คอมไพล์: nasm -f elf64 ifdef_demo.asm -o ifdef_demo.o && ld ifdef_demo.o -o ifdef_demo
; คอมไพล์ด้วย feature: nasm -f elf64 -DENABLE_SSE2 ifdef_demo.asm -o ifdef_demo.o && ld ifdef_demo.o -o ifdef_demo

; ========================================
; Include Guard — ป้องกัน include ซ้ำ
; ========================================
%ifndef _MYLIB_ASM_
%define _MYLIB_ASM_

    ; เนื้อหาของ library ที่นี่

%endif

; ========================================
; Feature detection
; ========================================
%ifdef ENABLE_SSE2
    %define VECTOR_ADD addpd    ; double precision SIMD
    %define VECTOR_MUL mulpd
    %message "Compiling with SSE2 support"
%else
    %define VECTOR_ADD addsd    ; scalar fallback
    %define VECTOR_MUL mulsd
    %message "Compiling without SSE2 (scalar mode)"
%endif

%ifdef ENABLE_AVX
    %message "AVX support enabled"
%endif

; ========================================
; OS Detection
; ========================================
%ifdef WINDOWS
    %define CALL_CONV   win64
    %define SHADOW_SPACE 32     ; Windows requires 32-byte shadow space
%else
    %define CALL_CONV   sysv64
    %define SHADOW_SPACE 0      ; Linux/macOS ไม่ต้องการ
%endif

; ========================================
; Conditional function definitions
; ========================================

; ถ้า DEBUG ถูก define ให้สร้าง debug version
%ifdef DEBUG
%macro CHECK_NULL 1
    test %1, %1
    jnz %%not_null
    ; pointer เป็น null!
    mov rax, 60
    mov rdi, 1      ; exit code 1 (error)
    syscall
%%not_null:
%endmacro
%else
; release version: ไม่มี check (เร็วกว่า)
%macro CHECK_NULL 1
    ; ไม่ทำอะไร ใน release mode
%endmacro
%endif

section .data
    some_ptr    dq 0x1000   ; non-null pointer

section .text
    global _start

_start:
    mov rax, [some_ptr]
    CHECK_NULL rax          ; ใน debug: ตรวจ null, ใน release: ไม่มีโค้ด

    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## หัวข้อที่ 10: %rep/%endrep — Repetition Macros

`%rep` ทำให้ block ของโค้ดถูก assemble ซ้ำกัน N ครั้ง

```nasm
%rep N
    ; โค้ดที่จะทำซ้ำ N ครั้ง
%endrep
```

### ตัวอย่าง: การใช้ %rep

```nasm
; ไฟล์: rep_demo.asm
; คอมไพล์: nasm -f elf64 rep_demo.asm -o rep_demo.o && ld rep_demo.o -o rep_demo

; ========================================
; สร้าง lookup table ด้วย %rep
; ========================================
section .data
    ; สร้าง table ของ squares (0^2 ถึง 15^2)
    square_table:
    %assign i 0
    %rep 16
        dd i*i          ; i^2
        %assign i i+1
    %endrep
    ; ผลลัพธ์: 0, 1, 4, 9, 16, 25, 36, 49, 64, 81, 100, 121, 144, 169, 196, 225

    ; สร้าง table ของ powers of 2
    pow2_table:
    %assign bit 0
    %rep 32
        dd 1 << bit     ; 2^bit
        %assign bit bit+1
    %endrep

    ; String ที่ซ้ำกัน
    separator:  times 40 db '-'     ; 40 dashes
                db 10               ; newline
    sep_len     equ $ - separator

; ========================================
; NOP sled (สำหรับ alignment)
; ========================================
section .text
    global _start

    ; NOP sled: 8 NOPs เพื่อ alignment
    %rep 8
        nop
    %endrep

_start:
    ; อ่านค่าจาก square_table
    ; index = 5, square = square_table[5] = 25
    mov rcx, 5
    lea rsi, [rel square_table]
    mov eax, [rsi + rcx*4]      ; eax = 25

    ; อ่านค่าจาก pow2_table
    ; 2^10 = 1024
    mov rcx, 10
    lea rsi, [rel pow2_table]
    mov eax, [rsi + rcx*4]      ; eax = 1024

    ; พิมพ์ separator
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel separator]
    mov rdx, sep_len
    syscall

    mov rax, 60
    xor rdi, rdi
    syscall
```

### ตัวอย่าง: %rep กับ %exitrep

```nasm
; %exitrep ออกจาก loop ก่อนกำหนด
section .data
    ; สร้าง fibonacci sequence ด้วย %rep
    fib_table:
    %assign fib_a 0
    %assign fib_b 1
    %assign fib_count 0
    %rep 20
        %if fib_a > 1000        ; หยุดเมื่อค่าเกิน 1000
            %exitrep
        %endif
        dq fib_a
        %assign fib_tmp fib_b
        %assign fib_b fib_a + fib_b
        %assign fib_a fib_tmp
        %assign fib_count fib_count + 1
    %endrep
    fib_count_val dd fib_count      ; จำนวน fibonacci numbers ที่สร้างได้
```

---

## หัวข้อที่ 11: %rotate — Rotating Macro Arguments

`%rotate` เลื่อน arguments ของ macro ทำให้สามารถประมวลผล argument list ได้

```nasm
%rotate N       ; เลื่อน arguments ไปทางซ้าย N ตำแหน่ง
                ; %rotate -N เลื่อนขวา
```

### ตัวอย่าง: PUSH/POP Multiple Registers

```nasm
; ไฟล์: rotate_demo.asm
; คอมไพล์: nasm -f elf64 rotate_demo.asm -o rotate_demo.o && ld rotate_demo.o -o rotate_demo

; ========================================
; PUSHALL / POPALL: บันทึกและคืน registers
; ========================================

%macro PUSHALL 1-8
    ; push ทุก argument ตามลำดับ
    %rep %0
        push %1
        %rotate 1           ; เลื่อนไปยัง argument ถัดไป
    %endrep
%endmacro

%macro POPALL 1-8
    ; pop ในลำดับย้อนกลับ (ใช้ %rotate -1)
    %rep %0
        %rotate -1          ; เลื่อนไปยัง argument ท้ายสุดก่อน
        pop %1
    %endrep
%endmacro

; ========================================
; SUM_LIST: บวกค่าทั้งหมดใน list
; ========================================
; SUM_LIST reg, val1, val2, ...
; เก็บผลรวมไว้ใน reg
%macro SUM_LIST 2-9
    xor %1, %1          ; reg = 0
    %rotate 1           ; เลื่อนไปยัง val แรก
    %rep %0-1           ; วนซ้ำสำหรับทุก value (%0-1 เพราะข้ามตัวแรก)
        add %1, %2      ; !! ผิด! ควรเป็น %1 ของ current rotation
    %endrep
%endmacro

; วิธีที่ถูกต้อง:
%macro SUM_VALS 2-9
    ; %1 = destination register
    ; %2, %3, ... = values
    xor %1, %1          ; เคลียร์ destination
    %assign %%_count %0 - 1     ; จำนวน values
    %rotate 1           ; เลื่อน: %1 กลายเป็น value แรก
    %rep %%_count
        add %1, %1      ; !! ยังไม่ถูก ต้องใช้ temp
    %endrep
%endmacro

; วิธีที่ถูกต้องและใช้งานได้จริง:
; (เก็บค่าใน rax แล้วรวม)
%macro STORE_SUM 1-8
    ; เก็บผลรวมของ arguments ทั้งหมดไว้ใน rax
    xor rax, rax
    %rep %0
        add rax, %1
        %rotate 1
    %endrep
%endmacro

; ========================================
; PRINT_LIST: พิมพ์หลาย string
; ========================================
%macro PRINT_LIST 1-8
    %rep %0
        push rsi
        push rdx
        push rdi
        push rax
        ; หา string length
        mov rsi, %1
        xor rdx, rdx
%%plist_len:
        cmp byte [rsi + rdx], 0
        je %%plist_print
        inc rdx
        jmp %%plist_len
%%plist_print:
        mov rdi, 1
        mov rax, 1
        syscall
        pop rax
        pop rdi
        pop rdx
        pop rsi
        %rotate 1
    %endrep
%endmacro

section .data
    s1  db "First", 10, 0
    s2  db "Second", 10, 0
    s3  db "Third", 10, 0

section .text
    global _start

_start:
    ; PUSHALL รับ registers หลายตัว
    PUSHALL rax, rbx, rcx, rdx

    ; ทำบางอย่าง
    mov rax, 42
    mov rbx, 100

    ; POPALL คืนค่า
    POPALL rax, rbx, rcx, rdx

    ; STORE_SUM
    STORE_SUM 1, 2, 3, 4, 5    ; rax = 1+2+3+4+5 = 15

    ; PRINT_LIST
    PRINT_LIST s1, s2, s3

    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## หัวข้อที่ 12: String Macros

NASM มี operators พิเศษสำหรับจัดการ string ใน macro

```nasm
%+          ; concatenation operator
%str        ; แปลง token เป็น string
%substr     ; substring
%strlen     ; ความยาว string
```

### ตัวอย่าง: String Macros

```nasm
; ไฟล์: string_macros.asm
; คอมไพล์: nasm -f elf64 string_macros.asm -o string_macros.o && ld string_macros.o -o string_macros

; ========================================
; String concatenation ด้วย %+
; ========================================

; สร้าง label จาก prefix + name
%define PREFIX_LABEL(name)  prefix_ %+ name

; ตัวอย่าง:
; PREFIX_LABEL(data) จะกลายเป็น: prefix_data

; ========================================
; ตรวจสอบ string length ด้วย %strlen
; ========================================
%define MY_STRING "Hello, World"
%strlen MY_STR_LEN MY_STRING
; MY_STR_LEN = 12

; ========================================
; Macro สำหรับ error messages
; ========================================

; COMPILE_ERROR msg — แสดง error ตอน compile
%macro COMPILE_ERROR 1
    %error "Compile Error: " %+ %1
%endmacro

; COMPILE_WARN msg — แสดง warning ตอน compile
%macro COMPILE_WARN 1
    %warning "Warning: " %+ %1
%endmacro

; ========================================
; String ใน Data Section
; ========================================

; DEFSTRING name, content — สร้าง string พร้อม length
%macro DEFSTRING 2
    %1:         db %2, 0        ; string + null terminator
    %1 %+ _len  equ $ - %1 - 1 ; length ไม่นับ null
%endmacro

; DEFSTRING_NL name, content — string พร้อม newline
%macro DEFSTRING_NL 2
    %1:         db %2, 10, 0    ; string + newline + null
    %1 %+ _len  equ $ - %1 - 1
%endmacro

; ========================================
; ใช้ DEFSTRING ใน data section
; ========================================
section .data

DEFSTRING greeting, "Hello, Assembly World!"
DEFSTRING_NL error_msg, "Error occurred"
DEFSTRING_NL info_msg, "Program started"
DEFSTRING_NL done_msg, "Program completed"

section .text
    global _start

%macro PRINT_DS 1
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel %1]
    mov rdx, %1 %+ _len
    syscall
%endmacro

_start:
    PRINT_DS info_msg
    PRINT_DS greeting       ; ไม่มี newline, ต่อเนื่อง
    PRINT_DS done_msg

    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## หัวข้อที่ 13: Macro สำหรับ Function Prologue/Epilogue

นี่คือการประยุกต์ใช้ macro ในการเขียน function ให้สวยงาม

### ตัวอย่าง: Complete Function Framework

```nasm
; ไฟล์: function_framework.asm
; คอมไพล์: nasm -f elf64 function_framework.asm -o function_framework.o
;           ld function_framework.o -o function_framework

; ================================================================
; FUNCTION FRAMEWORK MACROS
; System V AMD64 ABI (Linux/macOS)
; Parameter registers: rdi, rsi, rdx, rcx, r8, r9
; Return register: rax
; Callee-saved: rbx, r12-r15, rbp
; Caller-saved: rax, rcx, rdx, rsi, rdi, r8, r9, r10, r11
; ================================================================

; ----------------------------------------
; FUNC_DEF name, local_bytes
; กำหนด function พร้อม frame
; ----------------------------------------
%macro FUNC_DEF 2
global %1
%1:
    push rbp
    mov rbp, rsp
    %if %2 > 0
        sub rsp, %2
        and rsp, -16            ; align to 16 bytes
    %endif
%endmacro

; ----------------------------------------
; FUNC_DEF_SAVE name, local_bytes
; กำหนด function พร้อมบันทึก callee-saved registers
; ----------------------------------------
%macro FUNC_DEF_SAVE 2
global %1
%1:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    push r15
    %if %2 > 0
        sub rsp, %2
        and rsp, -16
    %endif
%endmacro

; ----------------------------------------
; FUNC_RET — return จาก function ปกติ
; ----------------------------------------
%macro FUNC_RET 0
    mov rsp, rbp
    pop rbp
    ret
%endmacro

; ----------------------------------------
; FUNC_RET_SAVE — return พร้อมคืน callee-saved registers
; ----------------------------------------
%macro FUNC_RET_SAVE 0
    mov rsp, rbp
    sub rsp, 40                 ; กลับไปตำแหน่งก่อน push r15
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    mov rsp, rbp
    pop rbp
    ret
%endmacro

; ----------------------------------------
; LOCAL_VAR offset — เข้าถึง local variable
; ----------------------------------------
%define LOCAL_VAR(offset)   [rbp - offset]

; ----------------------------------------
; ARG — เข้าถึง function argument บน stack
; (สำหรับ argument ที่ 7+ หลังจาก register args หมด)
; ----------------------------------------
%define STACK_ARG(n)    [rbp + 8 + (n)*8]     ; n=1 คือ argument แรกบน stack

; ----------------------------------------
; CALL_FUNC name, arg1, arg2, ...
; เรียกฟังก์ชันพร้อม arguments
; ----------------------------------------
%macro CALL_FUNC 1-7
    %if %0 >= 2
        mov rdi, %2
    %endif
    %if %0 >= 3
        mov rsi, %3
    %endif
    %if %0 >= 4
        mov rdx, %4
    %endif
    %if %0 >= 5
        mov rcx, %5
    %endif
    %if %0 >= 6
        mov r8, %6
    %endif
    %if %0 >= 7
        mov r9, %7
    %endif
    call %1
%endmacro

; ================================================================
; ตัวอย่างการใช้งาน
; ================================================================

section .data
    fmt_int     db "Value: %d", 10, 0
    fmt_str     db "String: %s", 10, 0

section .bss
    buf         resb 32

section .text

; ฟังก์ชัน: add_integers(a: i64, b: i64) -> i64
; rdi = a, rsi = b, return rax
FUNC_DEF add_integers, 0
    mov rax, rdi
    add rax, rsi
FUNC_RET

; ฟังก์ชัน: multiply_add(a: i64, b: i64, c: i64) -> i64
; rdi=a, rsi=b, rdx=c, return a*b + c
FUNC_DEF_SAVE multiply_add, 0
    mov r12, rdx            ; บันทึก c
    imul rdi, rsi           ; a * b
    add rdi, r12            ; a*b + c
    mov rax, rdi
FUNC_RET_SAVE

; ฟังก์ชัน: sum_array(arr: *i64, n: i64) -> i64
; rdi = pointer, rsi = count, return rax = sum
FUNC_DEF_SAVE sum_array, 16
    mov r12, rdi            ; บันทึก array pointer
    mov r13, rsi            ; บันทึก count
    xor rax, rax            ; sum = 0
    xor rcx, rcx            ; i = 0
.loop:
    cmp rcx, r13
    jge .done
    add rax, [r12 + rcx*8]  ; sum += arr[i]
    inc rcx
    jmp .loop
.done:
FUNC_RET_SAVE

global _start
_start:
    ; ทดสอบ add_integers(10, 32)
    CALL_FUNC add_integers, 10, 32  ; rax = 42

    ; ทดสอบ multiply_add(3, 4, 5)
    CALL_FUNC multiply_add, 3, 4, 5 ; rax = 17

    ; ทดสอบ sum_array
    ; สร้าง array บน stack
    sub rsp, 40
    mov qword [rsp+0],  1
    mov qword [rsp+8],  2
    mov qword [rsp+16], 3
    mov qword [rsp+24], 4
    mov qword [rsp+32], 5
    mov rdi, rsp            ; pointer
    mov rsi, 5              ; count
    call sum_array          ; rax = 15
    add rsp, 40

    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## หัวข้อที่ 14: Debug Macros

Debug macros ช่วยให้เราสามารถพิมพ์ข้อมูลเพื่อ debug ได้ง่ายๆ

### ตัวอย่าง: Comprehensive Debug Macros

```nasm
; ไฟล์: debug_macros.asm
; คอมไพล์ด้วย debug: nasm -f elf64 -DDEBUG debug_macros.asm -o debug.o && ld debug.o -o debug
; คอมไพล์ release: nasm -f elf64 debug_macros.asm -o debug.o && ld debug.o -o debug

; ================================================================
; DEBUG MACRO LIBRARY
; ================================================================

%ifdef DEBUG

; ----------------------------------------
; DBG_PRINT_REG reg — พิมพ์ค่า register เป็น hex
; ----------------------------------------
%macro DBG_PRINT_REG 1
    ; บันทึก registers ที่ใช้
    push rax
    push rbx
    push rcx
    push rdx
    push rsi
    push rdi

    ; สร้าง string "REG=0xXXXXXXXXXXXXXXXX\n"
    sub rsp, 32             ; local buffer

    ; เขียน register name
    mov rdi, rsp
    ; ไม่สามารถทำ %1 เป็น string โดยตรงได้ใน runtime
    ; แต่สามารถพิมพ์ค่าเป็น hex ได้

    ; ค่าของ register อยู่ใน %1 ณ ตอนนี้
    ; แต่เราต้อง backup ก่อน
    mov rbx, %1             ; บันทึกค่า

    ; แปลงเป็น hex string
    mov rax, rbx
    mov rcx, 16             ; 16 hex digits
    lea rdi, [rsp+16]       ; ชี้ไปยัง buffer
    add rdi, rcx            ; ชี้ไปยังท้าย buffer
    dec rdi
%%dbg_hex_loop:
    mov rdx, rax
    and rdx, 0xF            ; ดึง 4 bits ล่าง
    cmp rdx, 9
    jle %%dbg_digit
    add rdx, 'A' - 10       ; A-F
    jmp %%dbg_store
%%dbg_digit:
    add rdx, '0'            ; 0-9
%%dbg_store:
    mov [rdi], dl
    dec rdi
    shr rax, 4
    dec rcx
    jnz %%dbg_hex_loop

    ; พิมพ์ค่า hex
    lea rsi, [rsp+16]
    mov rdx, 16
    mov rdi, 2              ; stderr
    mov rax, 1
    syscall

    ; newline
    mov byte [rsp], 10
    mov rsi, rsp
    mov rdx, 1
    mov rdi, 2
    mov rax, 1
    syscall

    add rsp, 32

    pop rdi
    pop rsi
    pop rdx
    pop rcx
    pop rbx
    pop rax
%endmacro

; ----------------------------------------
; DBG_CHECKPOINT n — พิมพ์ "CHECKPOINT n" ไปยัง stderr
; ----------------------------------------
%macro DBG_CHECKPOINT 1
    section .data
%%chk_msg:  db "CHECKPOINT ", '0' + %1, 10
%%chk_len   equ $ - %%chk_msg
    section .text
    push rax
    push rdi
    push rsi
    push rdx
    mov rax, 1
    mov rdi, 2
    lea rsi, [rel %%chk_msg]
    mov rdx, %%chk_len
    syscall
    pop rdx
    pop rsi
    pop rdi
    pop rax
%endmacro

; ----------------------------------------
; DBG_ASSERT cond, msg — assert condition
; ----------------------------------------
%macro DBG_ASSERT 2
    test %1, %1
    jnz %%assert_ok
    ; assertion failed!
    push rax
    push rdi
    push rsi
    push rdx
    ; พิมพ์ error
    section .data
%%assert_msg:   db "ASSERTION FAILED: ", %2, 10
%%assert_len    equ $ - %%assert_msg
    section .text
    mov rax, 1
    mov rdi, 2
    lea rsi, [rel %%assert_msg]
    mov rdx, %%assert_len
    syscall
    pop rdx
    pop rsi
    pop rdi
    pop rax
    ; abort
    mov rax, 60
    mov rdi, 1
    syscall
%%assert_ok:
%endmacro

%else
    ; Release mode: macros เป็น no-op
    %macro DBG_PRINT_REG 1
    %endmacro

    %macro DBG_CHECKPOINT 1
    %endmacro

    %macro DBG_ASSERT 2
    %endmacro
%endif

; ================================================================
; Main Program
; ================================================================
section .text
    global _start

_start:
    DBG_CHECKPOINT 1        ; ใน debug mode จะพิมพ์ "CHECKPOINT 1"

    mov rax, 0xDEADBEEF
    DBG_PRINT_REG rax       ; ใน debug mode จะแสดงค่า

    DBG_CHECKPOINT 2

    ; ทดสอบ assert
    mov rax, 1
    DBG_ASSERT rax, "rax should be non-zero"  ; ผ่าน

    DBG_CHECKPOINT 3

    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## หัวข้อที่ 15: Assert Macros

Assert macros ช่วยตรวจสอบข้อสมมติฐานของโปรแกรม

### ตัวอย่าง: Runtime Assertion Framework

```nasm
; ไฟล์: assert_macros.asm
; คอมไพล์: nasm -f elf64 assert_macros.asm -o assert_macros.o && ld assert_macros.o -o assert_macros

; ================================================================
; RUNTIME ASSERTION FRAMEWORK
; ================================================================

; ----------------------------------------
; ASSERT_EQ a, b, msg — assert ว่า a == b
; ----------------------------------------
%macro ASSERT_EQ 3
    cmp %1, %2
    je %%assert_eq_ok
    ; ล้มเหลว: พิมพ์ข้อความ
    section .data
%%ae_msg:   db "ASSERT_EQ FAILED: ", %3, 10
%%ae_len    equ $ - %%ae_msg
    section .text
    push rax
    push rdi
    push rsi
    push rdx
    mov rax, 1
    mov rdi, 2              ; stderr
    lea rsi, [rel %%ae_msg]
    mov rdx, %%ae_len
    syscall
    pop rdx
    pop rsi
    pop rdi
    pop rax
    mov rax, 60
    mov rdi, 1
    syscall
%%assert_eq_ok:
%endmacro

; ----------------------------------------
; ASSERT_NE a, b, msg — assert ว่า a != b
; ----------------------------------------
%macro ASSERT_NE 3
    cmp %1, %2
    jne %%assert_ne_ok
    section .data
%%ane_msg:  db "ASSERT_NE FAILED: ", %3, 10
%%ane_len   equ $ - %%ane_msg
    section .text
    push rax
    push rdi
    push rsi
    push rdx
    mov rax, 1
    mov rdi, 2
    lea rsi, [rel %%ane_msg]
    mov rdx, %%ane_len
    syscall
    pop rdx
    pop rsi
    pop rdi
    pop rax
    mov rax, 60
    mov rdi, 1
    syscall
%%assert_ne_ok:
%endmacro

; ----------------------------------------
; ASSERT_GT a, b, msg — assert ว่า a > b (signed)
; ----------------------------------------
%macro ASSERT_GT 3
    cmp %1, %2
    jg %%assert_gt_ok
    section .data
%%agt_msg:  db "ASSERT_GT FAILED: ", %3, 10
%%agt_len   equ $ - %%agt_msg
    section .text
    push rax
    push rdi
    push rsi
    push rdx
    mov rax, 1
    mov rdi, 2
    lea rsi, [rel %%agt_msg]
    mov rdx, %%agt_len
    syscall
    pop rdx
    pop rsi
    pop rdi
    pop rax
    mov rax, 60
    mov rdi, 1
    syscall
%%assert_gt_ok:
%endmacro

; ----------------------------------------
; ASSERT_NOT_NULL reg, msg — assert ว่า pointer ไม่ใช่ null
; ----------------------------------------
%macro ASSERT_NOT_NULL 2
    test %1, %1
    jnz %%assert_nn_ok
    section .data
%%ann_msg:  db "NULL POINTER: ", %2, 10
%%ann_len   equ $ - %%ann_msg
    section .text
    push rax
    push rdi
    push rsi
    push rdx
    mov rax, 1
    mov rdi, 2
    lea rsi, [rel %%ann_msg]
    mov rdx, %%ann_len
    syscall
    pop rdx
    pop rsi
    pop rdi
    pop rax
    mov rax, 60
    mov rdi, 1
    syscall
%%assert_nn_ok:
%endmacro

; ----------------------------------------
; ASSERT_RANGE val, min, max, msg
; assert ว่า min <= val <= max
; ----------------------------------------
%macro ASSERT_RANGE 4
    cmp %1, %2          ; val >= min?
    jl %%assert_range_fail
    cmp %1, %3          ; val <= max?
    jle %%assert_range_ok
%%assert_range_fail:
    section .data
%%ar_msg:   db "RANGE CHECK FAILED: ", %4, 10
%%ar_len    equ $ - %%ar_msg
    section .text
    push rax
    push rdi
    push rsi
    push rdx
    mov rax, 1
    mov rdi, 2
    lea rsi, [rel %%ar_msg]
    mov rdx, %%ar_len
    syscall
    pop rdx
    pop rsi
    pop rdi
    pop rax
    mov rax, 60
    mov rdi, 1
    syscall
%%assert_range_ok:
%endmacro

section .data
    ptr1    dq 0x5000       ; non-null pointer

section .text
    global _start

_start:
    ; ทดสอบ ASSERT_EQ
    mov rax, 42
    mov rbx, 42
    ASSERT_EQ rax, rbx, "rax should equal rbx"     ; ผ่าน

    ; ทดสอบ ASSERT_NE
    mov rax, 1
    mov rbx, 2
    ASSERT_NE rax, rbx, "rax should not equal rbx" ; ผ่าน

    ; ทดสอบ ASSERT_GT
    mov rax, 10
    mov rbx, 5
    ASSERT_GT rax, rbx, "rax should be greater"    ; ผ่าน

    ; ทดสอบ ASSERT_NOT_NULL
    mov rax, [ptr1]
    ASSERT_NOT_NULL rax, "ptr1 should not be null"  ; ผ่าน

    ; ทดสอบ ASSERT_RANGE
    mov rax, 50
    ASSERT_RANGE rax, 0, 100, "rax in range [0,100]" ; ผ่าน

    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## หัวข้อที่ 16: Practical — Printf Wrapper Macro

นี่คือตัวอย่างที่ใช้งานได้จริงในการ wrap `printf` จาก C library

### ตัวอย่าง: Printf Wrapper

```nasm
; ไฟล์: printf_wrapper.asm
; คอมไพล์: nasm -f elf64 printf_wrapper.asm -o printf_wrapper.o
;           gcc printf_wrapper.o -o printf_wrapper -no-pie
; หรือ:     nasm -f elf64 printf_wrapper.asm -o printf_wrapper.o
;           ld printf_wrapper.o -o printf_wrapper -lc --dynamic-linker /lib64/ld-linux-x86-64.so.2

; ================================================================
; PRINTF WRAPPER MACRO LIBRARY
; ================================================================

extern printf
extern sprintf
extern fprintf
extern exit
extern puts

; ----------------------------------------
; PRINTF fmt, arg1, arg2, ...
; เรียก printf ด้วย format string และ arguments
; รองรับ 0-6 arguments หลัง format
; ----------------------------------------
%macro PRINTF 1-7
    ; ตั้งค่า arguments ตาม System V ABI
    ; rdi = format, rsi = arg1, rdx = arg2, rcx = arg3, r8 = arg4, r9 = arg5

    ; บันทึก caller-saved registers ที่สำคัญ
    push rbp
    mov rbp, rsp
    and rsp, -16            ; align stack

    ; ตั้งค่า arguments
    lea rdi, [rel %1]       ; format string

    %if %0 >= 2
        mov rsi, %2
    %endif
    %if %0 >= 3
        mov rdx, %3
    %endif
    %if %0 >= 4
        mov rcx, %4
    %endif
    %if %0 >= 5
        mov r8, %5
    %endif
    %if %0 >= 6
        mov r9, %6
    %endif
    %if %0 >= 7
        ; argument ที่ 7+ ต้อง push ลง stack
        push %7
    %endif

    xor eax, eax            ; rax = 0 (ไม่มี SSE args)
    call printf

    %if %0 >= 7
        add rsp, 8          ; ล้าง argument ที่ push
    %endif

    mov rsp, rbp
    pop rbp
%endmacro

; ----------------------------------------
; PRINTLN str — พิมพ์ string + newline
; ----------------------------------------
%macro PRINTLN 1
    push rbp
    mov rbp, rsp
    and rsp, -16
    lea rdi, [rel %1]
    call puts
    mov rsp, rbp
    pop rbp
%endmacro

; ----------------------------------------
; PRINT_INT n — พิมพ์ integer
; ----------------------------------------
%macro PRINT_INT 1
    section .data
%%pint_fmt: db "%ld", 10, 0
    section .text
    push rbp
    mov rbp, rsp
    and rsp, -16
    lea rdi, [rel %%pint_fmt]
    mov rsi, %1
    xor eax, eax
    call printf
    mov rsp, rbp
    pop rbp
%endmacro

; ----------------------------------------
; PRINT_HEX n — พิมพ์ hex
; ----------------------------------------
%macro PRINT_HEX 1
    section .data
%%phex_fmt: db "0x%016lX", 10, 0
    section .text
    push rbp
    mov rbp, rsp
    and rsp, -16
    lea rdi, [rel %%phex_fmt]
    mov rsi, %1
    xor eax, eax
    call printf
    mov rsp, rbp
    pop rbp
%endmacro

; ----------------------------------------
; PRINT_FLOAT — พิมพ์ double precision float
; ----------------------------------------
%macro PRINT_FLOAT 1
    section .data
%%pflt_fmt: db "%f", 10, 0
    section .text
    push rbp
    mov rbp, rsp
    and rsp, -16
    lea rdi, [rel %%pflt_fmt]
    movsd xmm0, %1          ; float argument ใช้ xmm registers
    mov eax, 1              ; 1 SSE argument
    call printf
    mov rsp, rbp
    pop rbp
%endmacro

; ================================================================
; ตัวอย่างการใช้งาน
; ================================================================

section .data
    ; Format strings
    fmt_hello       db "Hello, %s! You are %d years old.", 10, 0
    fmt_calc        db "Result: %d + %d = %d", 10, 0
    fmt_hex_demo    db "Address: 0x%016lX", 10, 0
    fmt_sep         db "----------------------------", 10, 0

    ; Strings
    str_world       db "World", 0
    str_nasm        db "NASM Programmer", 0

    ; Float value
    pi              dq 3.14159265358979

    ; Integer arrays
    nums            dq 10, 20, 30, 40, 50
    nums_count      equ 5

section .text
    global main

; ================================================================
; main function
; ================================================================
main:
    push rbp
    mov rbp, rsp
    sub rsp, 32
    and rsp, -16

    ; พิมพ์ separator
    PRINTLN fmt_sep

    ; พิมพ์ greeting
    PRINTF fmt_hello, str_world, 25

    ; พิมพ์ calculation
    mov rdi, 15
    mov rsi, 27
    lea rdx, [rdi + rsi]    ; = 42
    PRINTF fmt_calc, rdi, rsi, rdx

    ; พิมพ์ hex address
    lea rax, [rel nums]
    PRINTF fmt_hex_demo, rax

    ; พิมพ์ integers ด้วย PRINT_INT
    mov rcx, 0
.print_loop:
    cmp rcx, nums_count
    jge .print_done
    push rcx
    mov rax, [nums + rcx*8]
    PRINT_INT rax
    pop rcx
    inc rcx
    jmp .print_loop
.print_done:

    ; พิมพ์ float
    PRINT_FLOAT [rel pi]

    ; พิมพ์ hex
    mov rax, 0xDEADBEEFCAFE
    PRINT_HEX rax

    PRINTLN fmt_sep

    xor eax, eax
    mov rsp, rbp
    pop rbp
    ret
```

---

## สรุปตาราง Macro Directives

| Directive | คำอธิบาย | ตัวอย่าง |
|-----------|----------|---------|
| `%define` | Simple text substitution | `%define SIZE 1024` |
| `%undef` | ยกเลิก definition | `%undef SIZE` |
| `%assign` | Numeric assignment | `%assign N 0` |
| `%macro/%endmacro` | Multi-line macro | `%macro FOO 2 ... %endmacro` |
| `%%label` | Local label ใน macro | `%%loop:` |
| `%0` | จำนวน arguments | `%if %0 == 2` |
| `%1 ... %9` | Argument n | `mov rax, %1` |
| `%rotate n` | เลื่อน arguments | `%rotate 1` |
| `%rep/%endrep` | ทำซ้ำ | `%rep 4 nop %endrep` |
| `%exitrep` | ออกจาก %rep loop | `%exitrep` |
| `%if/%elif/%else/%endif` | Conditional | `%if N > 0` |
| `%ifdef/%ifndef` | Existence check | `%ifdef DEBUG` |
| `%error` | Compile-time error | `%error "Bad!"` |
| `%warning` | Compile-time warning | `%warning "Hmm"` |
| `%message` | Compile-time message | `%message "OK"` |
| `%+` | String concatenation | `prefix_ %+ name` |
| `%strlen` | String length | `%strlen LEN "hello"` |

---

## ตัวอย่างโปรแกรมสมบูรณ์: Library ของ NASM Macros

```nasm
; ไฟล์: complete_macro_library.asm
; คอมไพล์: nasm -f elf64 complete_macro_library.asm -o complete_macro_library.o
;           ld complete_macro_library.o -o complete_macro_library

; ================================================================
; COMPLETE NASM MACRO LIBRARY
; รวม macros ที่ใช้บ่อยทั้งหมด
; ================================================================

; ----------------------------------------------------------------
; SECTION 1: System Call Numbers (Linux x86-64)
; ----------------------------------------------------------------
%define SYS_READ        0
%define SYS_WRITE       1
%define SYS_OPEN        2
%define SYS_CLOSE       3
%define SYS_STAT        4
%define SYS_FSTAT       5
%define SYS_MMAP        9
%define SYS_MUNMAP      11
%define SYS_BRK         12
%define SYS_EXIT        60
%define SYS_FORK        57
%define SYS_EXECVE      59
%define SYS_WAIT4       61
%define SYS_KILL        62
%define SYS_GETPID      39
%define SYS_GETPPID     110
%define SYS_SOCKET      41
%define SYS_CONNECT     42
%define SYS_ACCEPT      43
%define SYS_SEND        44
%define SYS_RECV        45
%define SYS_BIND        49
%define SYS_LISTEN      50
%define SYS_FSYNC       74
%define SYS_RENAME      82
%define SYS_MKDIR       83
%define SYS_RMDIR       84
%define SYS_UNLINK      87
%define SYS_GETCWD      79

; Standard file descriptors
%define FD_STDIN        0
%define FD_STDOUT       1
%define FD_STDERR       2

; Common exit codes
%define EXIT_SUCCESS    0
%define EXIT_FAILURE    1

; ----------------------------------------------------------------
; SECTION 2: Utility Macros
; ----------------------------------------------------------------

; SYSCALL1 num, arg1
%macro SYSCALL1 2
    mov rax, %1
    mov rdi, %2
    syscall
%endmacro

; SYSCALL2 num, arg1, arg2
%macro SYSCALL2 3
    mov rax, %1
    mov rdi, %2
    mov rsi, %3
    syscall
%endmacro

; SYSCALL3 num, arg1, arg2, arg3
%macro SYSCALL3 4
    mov rax, %1
    mov rdi, %2
    mov rsi, %3
    mov rdx, %4
    syscall
%endmacro

; ----------------------------------------------------------------
; SECTION 3: I/O Macros
; ----------------------------------------------------------------

; WRITE fd, buf, len
%macro WRITE 3
    SYSCALL3 SYS_WRITE, %1, %2, %3
%endmacro

; WRITELN buf, len — เขียนไปยัง stdout
%macro WRITELN 2
    WRITE FD_STDOUT, %1, %2
%endmacro

; READ fd, buf, maxlen
%macro READ 3
    SYSCALL3 SYS_READ, %1, %2, %3
%endmacro

; EXIT code
%macro EXIT 1
    SYSCALL1 SYS_EXIT, %1
%endmacro

; ----------------------------------------------------------------
; SECTION 4: Math Macros
; ----------------------------------------------------------------

; SWAP a, b — สลับค่า (ใช้ xor trick)
%macro SWAP 2
    xor %1, %2
    xor %2, %1
    xor %1, %2
%endmacro

; CLAMP val, lo, hi — จำกัดค่าให้อยู่ใน [lo, hi]
%macro CLAMP 3
    cmp %1, %2
    jge %%clamp_lo_ok
    mov %1, %2
%%clamp_lo_ok:
    cmp %1, %3
    jle %%clamp_hi_ok
    mov %1, %3
%%clamp_hi_ok:
%endmacro

; IS_POWER_OF_2 reg, out_reg — ตรวจว่าเป็น power of 2
; out_reg = 1 ถ้าใช่, 0 ถ้าไม่ใช่
%macro IS_POWER_OF_2 2
    mov %2, %1
    dec %2
    test %1, %2
    setz %2b                ; set byte to 1 if ZF=1
    movzx %2, %2b
%endmacro

; ROUND_UP_POW2 reg — หา power of 2 ที่ใกล้ที่สุดและ >= reg
%macro ROUND_UP_POW2 1
    dec %1
    mov rcx, %1
    shr rcx, 1
    or %1, rcx
    mov rcx, %1
    shr rcx, 2
    or %1, rcx
    mov rcx, %1
    shr rcx, 4
    or %1, rcx
    mov rcx, %1
    shr rcx, 8
    or %1, rcx
    mov rcx, %1
    shr rcx, 16
    or %1, rcx
    mov rcx, %1
    shr rcx, 32
    or %1, rcx
    inc %1
%endmacro

; ----------------------------------------------------------------
; SECTION 5: String Macros
; ----------------------------------------------------------------

; STRLEN src_reg, len_reg — หาความยาว null-terminated string
%macro STRLEN 2
    xor %2, %2
%%strlen_loop:
    cmp byte [%1 + %2], 0
    je %%strlen_done
    inc %2
    jmp %%strlen_loop
%%strlen_done:
%endmacro

; STRCMP str1_reg, str2_reg — เปรียบเทียบ strings
; ZF=1 ถ้าเท่ากัน
%macro STRCMP 2
    push rsi
    push rdi
    push rcx
    push rax
    mov rsi, %1
    mov rdi, %2
%%strcmp_loop:
    lodsb                   ; al = [rsi++]
    scasb                   ; compare al with [rdi++]
    jne %%strcmp_ne
    test al, al             ; null terminator?
    jnz %%strcmp_loop
    ; strings are equal (ZF already set by test al,al when al=0 and they're equal)
    jmp %%strcmp_done
%%strcmp_ne:
    ; strings not equal (ZF=0 from scasb)
%%strcmp_done:
    pop rax
    pop rcx
    pop rdi
    pop rsi
%endmacro

; MEMCPY dst, src, len — copy memory
%macro MEMCPY 3
    push rdi
    push rsi
    push rcx
    mov rdi, %1
    mov rsi, %2
    mov rcx, %3
    rep movsb
    pop rcx
    pop rsi
    pop rdi
%endmacro

; MEMSET dst, val, len — fill memory
%macro MEMSET 3
    push rdi
    push rcx
    push rax
    mov rdi, %1
    mov al, %2
    mov rcx, %3
    rep stosb
    pop rax
    pop rcx
    pop rdi
%endmacro

; ----------------------------------------------------------------
; SECTION 6: Stack Management
; ----------------------------------------------------------------

; ALIGN_STACK — จัดตำแหน่ง stack ให้ align 16 bytes
%macro ALIGN_STACK 0
    push rbp
    mov rbp, rsp
    and rsp, -16
%endmacro

%macro RESTORE_STACK 0
    mov rsp, rbp
    pop rbp
%endmacro

; ----------------------------------------------------------------
; SECTION 7: Test Data
; ----------------------------------------------------------------

section .data
    msg_start   db "=== NASM Macro Library Demo ===", 10
    msg_start_l equ $ - msg_start

    msg_swap    db "After SWAP: "
    msg_swap_l  equ $ - msg_swap

    msg_end     db "=== Demo Complete ===", 10
    msg_end_l   equ $ - msg_end

    buf_num     times 32 db 0   ; buffer สำหรับแปลงตัวเลข

section .bss
    input_buf   resb 256

; ----------------------------------------------------------------
; SECTION 8: Utility Functions
; ----------------------------------------------------------------

section .text
    global _start

; ----------------------------------------------------------------
; itoa: แปลง integer เป็น ASCII string
; rdi = number, rsi = buffer, return rax = length
; ----------------------------------------------------------------
itoa:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13

    mov r12, rsi            ; บันทึก buffer pointer
    mov r13, rdi            ; บันทึก number
    xor rbx, rbx            ; digit count = 0

    ; handle negative
    test r13, r13
    jns .positive
    neg r13
    mov byte [r12], '-'
    inc r12
    inc rbx

.positive:
    ; แปลงเป็น digits (ลำดับย้อนกลับ)
    lea rsi, [r12 + 20]     ; end of temp buffer
    mov rcx, 0              ; digit index

.extract_digits:
    xor edx, edx
    mov rax, r13
    mov r8, 10
    div r8                  ; rdx = r13 % 10, rax = r13 / 10
    add dl, '0'
    dec rsi
    mov [rsi], dl
    inc rcx
    mov r13, rax
    test r13, r13
    jnz .extract_digits

    ; copy digits to buffer (ลำดับถูกต้อง)
    mov rdi, r12            ; destination
.copy_digits:
    lodsb                   ; al = [rsi++]
    stosb                   ; [rdi++] = al
    dec rcx
    jnz .copy_digits

    mov byte [rdi], 0       ; null terminator

    ; คำนวณ length
    sub rdi, r12
    add rdi, rbx            ; รวม '-' sign ถ้ามี
    mov rax, rdi

    pop r13
    pop r12
    pop rbx
    mov rsp, rbp
    pop rbp
    ret

; ----------------------------------------------------------------
; _start: Main Program
; ----------------------------------------------------------------
_start:
    ; แสดง header
    WRITELN msg_start, msg_start_l

    ; ทดสอบ SWAP macro
    mov rax, 111
    mov rbx, 222
    SWAP rax, rbx           ; rax = 222, rbx = 111

    ; แปลง rax เป็น string และพิมพ์
    WRITELN msg_swap, msg_swap_l
    mov rdi, rax
    lea rsi, [rel buf_num]
    call itoa
    mov rdx, rax
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel buf_num]
    syscall
    ; พิมพ์ newline
    push 10
    mov rax, 1
    mov rdi, 1
    mov rsi, rsp
    mov rdx, 1
    syscall
    pop rax

    ; ทดสอบ CLAMP
    mov rax, 150
    CLAMP rax, 0, 100       ; rax = 100 (clamp ไว้ที่ max)

    mov rax, -5
    CLAMP rax, 0, 100       ; rax = 0 (clamp ไว้ที่ min)

    ; ทดสอบ IS_POWER_OF_2
    mov rax, 64
    IS_POWER_OF_2 rax, rbx  ; rbx = 1 (64 = 2^6)

    mov rax, 63
    IS_POWER_OF_2 rax, rbx  ; rbx = 0 (63 ไม่ใช่ power of 2)

    ; ทดสอบ ROUND_UP_POW2
    mov rax, 100
    ROUND_UP_POW2 rax       ; rax = 128

    ; แสดง footer
    WRITELN msg_end, msg_end_l

    EXIT EXIT_SUCCESS
```

---

## แบบฝึกหัด (Exercises)

### ระดับเริ่มต้น

**แบบฝึกหัดที่ 1:** สร้าง macro `PRINT_CHAR ch` ที่พิมพ์ character 1 ตัวไปยัง stdout โดยใช้ syscall
```nasm
; ตัวอย่างการใช้:
PRINT_CHAR 'A'      ; พิมพ์ 'A'
PRINT_CHAR 10       ; พิมพ์ newline
```

**แบบฝึกหัดที่ 2:** สร้าง macro `MULTIPLY_BY_N reg, n` ที่คูณ register ด้วย n โดยใช้ shift สำหรับ power-of-2 และ imul สำหรับ กรณีอื่น ใช้ `%if` ในการตัดสินใจ

**แบบฝึกหัดที่ 3:** สร้างชุด macros สำหรับ array operations:
- `ARRAY_GET arr, idx, elem_size, dest` — อ่านค่า element
- `ARRAY_SET arr, idx, elem_size, val` — เขียนค่า element

### ระดับกลาง

**แบบฝึกหัดที่ 4:** สร้าง macro `FOR_LOOP counter_reg, start, end, step` ที่ทำงานเหมือน for loop:
```nasm
; ตัวอย่าง: for (rcx = 0; rcx < 10; rcx++)
FOR_LOOP rcx, 0, 10, 1
    ; เนื้อหา loop
ENDFOR
```

**แบบฝึกหัดที่ 5:** สร้าง Stack-based String Buffer โดยใช้ macro:
- `STR_PUSH_CHAR reg` — push character ลง string buffer
- `STR_PUSH_STR ptr, len` — push string ลง buffer
- `STR_FLUSH fd` — write buffer ไปยัง fd และล้าง buffer

**แบบฝึกหัดที่ 6:** ใช้ `%rep` และ `%assign` สร้าง sine lookup table สำหรับ angle 0-359 degrees (ใช้ scaled integer 1000x)

### ระดับสูง

**แบบฝึกหัดที่ 7:** สร้าง `%macro SWITCH value` / `%macro CASE const` / `%macro DEFAULT` / `%macro ENDSWITCH` ที่จำลอง switch-case statement

**แบบฝึกหัดที่ 8:** สร้าง Variadic Printf Macro ที่สามารถรับ format string และ arguments ได้สูงสุด 6 ตัว โดยใช้ `%rotate` เพื่อจัดการ arguments

**แบบฝึกหัดที่ 9:** สร้าง Mini Unit Test Framework โดยใช้ macros:
- `TEST_SUITE name` — เริ่ม test suite
- `TEST_CASE name` — เริ่ม test case
- `EXPECT_EQ actual, expected` — assert equality
- `EXPECT_TRUE condition` — assert true
- `TEST_REPORT` — แสดงสรุปผล

### ระดับผู้เชี่ยวชาญ

**แบบฝึกหัดที่ 10:** สร้าง Type-safe Register Wrapper โดยใช้ macros:
- สร้าง "typed" registers เช่น `INT_REG`, `PTR_REG`, `FLOAT_REG`
- ตรวจสอบ type ใน compile time ด้วย `%if`
- สร้าง operations ที่ type-safe เช่น `INT_ADD`, `PTR_ADD`

---

## เฉลยแบบฝึกหัด (Selected Solutions)

### เฉลยแบบฝึกหัดที่ 1: PRINT_CHAR

```nasm
; เฉลย: PRINT_CHAR macro
%macro PRINT_CHAR 1
    ; บันทึก registers
    push rax
    push rdi
    push rsi
    push rdx

    ; วาง character บน stack
    push qword 0            ; padding
    mov byte [rsp], %1      ; เก็บ character

    ; syscall write
    mov rax, 1              ; sys_write
    mov rdi, 1              ; stdout
    mov rsi, rsp            ; pointer ไปยัง character
    mov rdx, 1              ; 1 byte
    syscall

    add rsp, 8              ; ล้าง stack

    ; คืน registers
    pop rdx
    pop rsi
    pop rdi
    pop rax
%endmacro
```

### เฉลยแบบฝึกหัดที่ 4: FOR_LOOP

```nasm
; เฉลย: FOR_LOOP และ ENDFOR macros
%assign _for_depth 0

%macro FOR_LOOP 4
    %assign _for_depth _for_depth + 1
    mov %1, %2                      ; counter = start
%%for_start_ %+ _for_depth %+ :
    cmp %1, %3                      ; counter < end?
    jge %%for_end_ %+ _for_depth    ; ถ้าไม่ใช่ ออกจาก loop
    ; เนื้อหา loop จะตามมา
    ; %4 = step (ต้องใช้ใน ENDFOR)
    %push for_ctx
    %define %$step %4
    %define %$counter %1
    %define %$depth _for_depth
%endmacro

%macro ENDFOR 0
    add %$counter, %$step           ; counter += step
    jmp %%for_start_ %+ %$depth
%%for_end_ %+ %$depth %+ :
    %pop for_ctx
    %assign _for_depth _for_depth - 1
%endmacro
```

---

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:

1. **`%define`** — Simple text substitution สำหรับ constants, aliases, และ function-like macros
2. **`%assign`** — Numeric assignment ที่คำนวณค่าทันที เหมาะสำหรับ enums และ calculations
3. **Single-line macros with arguments** — Macros บรรทัดเดียวที่รับ parameters
4. **`%macro/%endmacro`** — Multi-line macros ที่มีความสามารถสูงสุด
5. **`%%label`** — Local labels ที่ป้องกัน label conflicts ใน macro
6. **Default arguments** — Macros ที่มี optional parameters
7. **Overloaded macros** — Macros ชื่อเดียวกันแต่รับ arguments ต่างจำนวน
8. **Conditional compilation** — `%if/%elif/%else/%endif` สำหรับ platform-specific code
9. **`%ifdef/%ifndef`** — ตรวจสอบว่า macro ถูก define แล้วหรือยัง
10. **`%rep/%endrep`** — ทำซ้ำโค้ดหรือข้อมูล
11. **`%rotate`** — จัดการ argument lists
12. **String macros** — `%+`, `%strlen`, `%substr`
13. **Function macros** — Prologue/epilogue สำหรับ clean function code
14. **Debug macros** — เครื่องมือ debug ที่หายไปใน release build
15. **Assert macros** — ตรวจสอบข้อสมมติฐานของโปรแกรม
16. **Printf wrapper** — การ wrap C library functions ด้วย macros

### คำสั่ง Compile ที่ใช้บ่อย

```bash
# Basic compile
nasm -f elf64 file.asm -o file.o && ld file.o -o file

# With debug symbols
nasm -f elf64 -g -F dwarf file.asm -o file.o && ld file.o -o file

# With defines (feature flags)
nasm -f elf64 -DDEBUG -DDEBUG_LEVEL=2 file.asm -o file.o && ld file.o -o file

# Link with C library (สำหรับ printf)
nasm -f elf64 file.asm -o file.o
gcc file.o -o file -no-pie

# Generate preprocessed output (ดู macro expansion)
nasm -f elf64 -e file.asm     # only preprocess
nasm -f elf64 -l file.lst file.asm -o file.o   # generate listing
```

### ขั้นตอนถัดไป

ใน Part 028 เราจะเรียนรู้เรื่อง **NASM Includes และ Modular Programming** ซึ่งรวมถึง:
- การใช้ `%include` เพื่อแยก code เป็นไฟล์
- การสร้าง header files (.inc) สำหรับ NASM
- Modular design patterns
- การสร้าง static libraries
- การเชื่อมต่อกับ C libraries

---

*Part 027 — NASM Macros อย่างละเอียด | Assembly Programming Course*
*ระดับ: Intermediate → Advanced | บรรทัดทั้งหมด: ~1050+ บรรทัด*
