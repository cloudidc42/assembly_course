# Part 025: Recursive Procedures (โพรซีเดอร์แบบเรียกซ้ำ)

## หลักสูตร Assembly Programming x86/ARM
### ระดับ: Intermediate to Advanced

---

## สารบัญ (Table of Contents)

1. [แนวคิดการ Recursion ใน Assembly](#recursion-concept)
2. [การใช้ Stack ใน Recursion](#stack-usage)
3. [Fibonacci แบบ Recursive](#fibonacci-recursive)
4. [Factorial แบบ Recursive](#factorial-recursive)
5. [Tower of Hanoi](#tower-of-hanoi)
6. [Binary Search แบบ Recursive](#binary-search-recursive)
7. [Merge Sort แบบ Recursive](#merge-sort-recursive)
8. [Tree Traversal](#tree-traversal)
9. [Backtracking Algorithm](#backtracking)
10. [Memoization ใน Assembly](#memoization)
11. [Tail Recursion Optimization](#tail-recursion)
12. [Stack Depth Limits](#stack-limits)
13. [การแปลง Recursion เป็น Iteration](#recursion-to-iteration)
14. [Trampolining Technique](#trampolining)
15. [การเปรียบเทียบ Performance](#performance-comparison)
16. [แบบฝึกหัด](#exercises)

---

## 1. แนวคิดการ Recursion ใน Assembly {#recursion-concept}

### Recursion คืออะไร?

**Recursion** คือเทคนิคที่ฟังก์ชันเรียกตัวเองซ้ำ (self-calling) เพื่อแก้ปัญหาที่สามารถแบ่งเป็นปัญหาย่อยที่มีโครงสร้างเดียวกัน

ใน Assembly การทำ Recursion ต้องเข้าใจ:
- **Stack frame** การจัดการ return address
- **Register preservation** การเก็บค่า register
- **Base case** เงื่อนไขหยุดการ recursive
- **Stack overflow** ขีดจำกัดของ stack

### โครงสร้าง Stack Frame สำหรับ Recursive Function

```
Stack ก่อน recursive call:
┌─────────────────┐ ← RSP (ก่อน call)
│   Local vars    │
│   Saved regs    │
│   Return addr   │
│   Parameters    │
└─────────────────┘

Stack หลัง recursive call:
┌─────────────────┐ ← RSP (ใหม่)
│   Local vars    │ ← Frame ใหม่
│   Saved regs    │
│   Return addr   │
│─────────────────│
│   Local vars    │ ← Frame เดิม
│   Saved regs    │
│   Return addr   │
│   Parameters    │
└─────────────────┘
```

### ตัวอย่างพื้นฐาน: Simple Recursive Count

```nasm
; ไฟล์: recursive_count.asm
; คำอธิบาย: ตัวอย่าง recursion พื้นฐาน - นับถอยหลัง
; คอมไพล์: nasm -f elf64 recursive_count.asm -o recursive_count.o
; ลิงก์:   ld recursive_count.o -o recursive_count

section .data
    fmt_count   db "Count: %d", 10, 0
    msg_done    db "Done!", 10, 0

section .text
    global main
    extern printf

; ฟังก์ชัน countdown(n) - นับจาก n ลงมาถึง 0
; พารามิเตอร์: rdi = n (ตัวเลขที่จะนับ)
countdown:
    push rbp                    ; เก็บ base pointer เดิม
    mov rbp, rsp                ; ตั้งค่า stack frame ใหม่
    sub rsp, 32                 ; จองพื้นที่ local variables
    
    ; เก็บ parameter ไว้ใน local variable
    mov [rbp-8], rdi            ; เก็บ n ลงใน stack
    
    ; Base case: ถ้า n <= 0 ให้หยุด
    cmp rdi, 0
    jle .done                   ; กระโดดไปยัง done ถ้า n <= 0
    
    ; แสดงค่า n ปัจจุบัน
    push rdi                    ; เก็บ n ไว้ก่อน printf เปลี่ยน registers
    lea rdi, [rel fmt_count]    ; format string
    mov rsi, [rbp-8]            ; n = ค่าปัจจุบัน
    xor eax, eax                ; clear rax (no floating point args)
    call printf
    pop rdi                     ; คืนค่า n กลับมา
    
    ; Recursive call: countdown(n-1)
    mov rdi, [rbp-8]            ; โหลด n กลับมา
    dec rdi                     ; n - 1
    call countdown              ; เรียกตัวเองซ้ำ
    
    jmp .return

.done:
    ; แสดงข้อความเสร็จสิ้น
    lea rdi, [rel msg_done]
    xor eax, eax
    call printf

.return:
    mov rsp, rbp                ; คืนค่า stack pointer
    pop rbp                     ; คืนค่า base pointer
    ret

main:
    push rbp
    mov rbp, rsp
    
    mov rdi, 5                  ; เริ่มนับจาก 5
    call countdown
    
    xor eax, eax                ; return 0
    pop rbp
    ret
```

**คำสั่งคอมไพล์และรัน:**
```bash
nasm -f elf64 recursive_count.asm -o recursive_count.o
gcc recursive_count.o -o recursive_count -no-pie
./recursive_count
```

---

## 2. การใช้ Stack ใน Recursion {#stack-usage}

### Stack Frame Layout สำหรับ Recursive Functions

เมื่อฟังก์ชัน recursive ถูกเรียก แต่ละ call จะสร้าง **stack frame** ใหม่บน stack:

```
Memory (สูง → ต่ำ):
┌─────────────────────────────────┐ ← Stack bottom
│  main() frame                   │
│  [rbp - 8] = saved rbx          │
│  [rbp - 16] = local var         │
│  Return address to caller       │
│─────────────────────────────────│
│  func(5) frame                  │
│  [rbp - 8] = n = 5              │
│  Return address                 │
│─────────────────────────────────│
│  func(4) frame                  │
│  [rbp - 8] = n = 4              │
│  Return address                 │
│─────────────────────────────────│
│  func(3) frame                  │
│  [rbp - 8] = n = 3              │
│  Return address                 │
└─────────────────────────────────┘ ← RSP (stack top)
```

### ตัวอย่าง: การตรวจสอบ Stack Depth

```nasm
; ไฟล์: stack_depth.asm
; คำอธิบาย: แสดงการเปลี่ยนแปลงของ stack pointer ใน recursion
; คอมไพล์: nasm -f elf64 stack_depth.asm -o stack_depth.o
; ลิงก์:   gcc stack_depth.o -o stack_depth -no-pie

section .data
    fmt_rsp     db "Depth %d: RSP = 0x%lx (frame size: %ld bytes)", 10, 0
    initial_rsp dq 0                ; เก็บ RSP เริ่มต้น

section .text
    global main
    extern printf

; show_stack_depth(depth) - แสดง stack pointer ที่แต่ละระดับ
; rdi = depth level
show_stack_depth:
    push rbp
    mov rbp, rsp
    sub rsp, 32
    
    mov [rbp-8], rdi            ; เก็บ depth
    
    ; คำนวณ frame size
    mov rax, [rel initial_rsp]
    sub rax, rsp                ; frame size = initial - current
    
    ; แสดงข้อมูล
    lea rdi, [rel fmt_rsp]
    mov rsi, [rbp-8]            ; depth
    mov rdx, rsp                ; current RSP
    mov rcx, rax                ; frame size
    xor eax, eax
    call printf
    
    ; Base case: หยุดที่ depth 5
    cmp qword [rbp-8], 5
    jge .done
    
    ; Recursive: ไประดับถัดไป
    mov rdi, [rbp-8]
    inc rdi
    call show_stack_depth
    
.done:
    mov rsp, rbp
    pop rbp
    ret

main:
    push rbp
    mov rbp, rsp
    
    ; บันทึก RSP เริ่มต้น
    mov [rel initial_rsp], rsp
    
    mov rdi, 0
    call show_stack_depth
    
    xor eax, eax
    pop rbp
    ret
```

---

## 3. Fibonacci แบบ Recursive {#fibonacci-recursive}

### อัลกอริทึม Fibonacci

```
fib(0) = 0
fib(1) = 1
fib(n) = fib(n-1) + fib(n-2)  สำหรับ n > 1
```

### Implementation ใน NASM x86-64

```nasm
; ไฟล์: fibonacci_recursive.asm
; คำอธิบาย: คำนวณ Fibonacci แบบ recursive พร้อมนับจำนวนครั้งที่เรียก
; คอมไพล์: nasm -f elf64 fibonacci_recursive.asm -o fibonacci_recursive.o
; ลิงก์:   gcc fibonacci_recursive.o -o fibonacci_recursive -no-pie

section .data
    fmt_result  db "fib(%d) = %lld (calls: %lld)", 10, 0
    fmt_header  db "=== Fibonacci Recursive Demo ===", 10, 0
    call_count  dq 0                ; นับจำนวน recursive calls

section .text
    global main
    extern printf

; fibonacci(n) - คำนวณ Fibonacci number ที่ n
; พารามิเตอร์: rdi = n
; คืนค่า: rax = fib(n)
fibonacci:
    push rbp
    mov rbp, rsp
    push rbx                        ; เก็บ rbx (callee-saved)
    sub rsp, 16                     ; จองพื้นที่
    
    ; เพิ่ม call counter
    inc qword [rel call_count]
    
    ; เก็บ n
    mov rbx, rdi                    ; rbx = n
    
    ; Base case 1: fib(0) = 0
    test rdi, rdi
    jz .return_zero
    
    ; Base case 2: fib(1) = 1
    cmp rdi, 1
    je .return_one
    
    ; Recursive case: fib(n-1) + fib(n-2)
    
    ; คำนวณ fib(n-1)
    lea rdi, [rbx-1]                ; n-1
    call fibonacci
    mov [rbp-16], rax               ; เก็บผล fib(n-1)
    
    ; คำนวณ fib(n-2)
    lea rdi, [rbx-2]                ; n-2
    call fibonacci
    
    ; รวมผล
    add rax, [rbp-16]               ; fib(n) = fib(n-1) + fib(n-2)
    jmp .done

.return_zero:
    xor eax, eax                    ; return 0
    jmp .done

.return_one:
    mov eax, 1                      ; return 1

.done:
    add rsp, 16
    pop rbx
    pop rbp
    ret

main:
    push rbp
    mov rbp, rsp
    push rbx
    sub rsp, 16
    
    ; แสดง header
    lea rdi, [rel fmt_header]
    xor eax, eax
    call printf
    
    ; คำนวณ fib(0) ถึง fib(15)
    xor rbx, rbx                    ; rbx = i = 0

.loop:
    cmp rbx, 16
    jge .end_loop
    
    ; รีเซ็ต call counter
    mov qword [rel call_count], 0
    
    ; คำนวณ fibonacci(i)
    mov rdi, rbx
    call fibonacci
    
    ; แสดงผล
    mov [rbp-8], rax                ; เก็บผลลัพธ์
    lea rdi, [rel fmt_result]
    mov rsi, rbx                    ; n
    mov rdx, [rbp-8]                ; fib(n)
    mov rcx, [rel call_count]       ; จำนวน calls
    xor eax, eax
    call printf
    
    inc rbx
    jmp .loop

.end_loop:
    xor eax, eax
    add rsp, 16
    pop rbx
    pop rbp
    ret
```

**ผลลัพธ์ที่คาดหวัง:**
```
=== Fibonacci Recursive Demo ===
fib(0) = 0 (calls: 1)
fib(1) = 1 (calls: 1)
fib(2) = 1 (calls: 3)
fib(5) = 5 (calls: 15)
fib(10) = 55 (calls: 177)
fib(15) = 610 (calls: 1973)
```

---

## 4. Factorial แบบ Recursive {#factorial-recursive}

### อัลกอริทึม Factorial

```
0! = 1
n! = n × (n-1)!  สำหรับ n > 0
```

### Implementation พร้อม Error Checking

```nasm
; ไฟล์: factorial_recursive.asm
; คำอธิบาย: คำนวณ factorial แบบ recursive พร้อมตรวจสอบ overflow
; คอมไพล์: nasm -f elf64 factorial_recursive.asm -o factorial_recursive.o
; ลิงก์:   gcc factorial_recursive.o -o factorial_recursive -no-pie

section .data
    fmt_result      db "%d! = %llu", 10, 0
    fmt_overflow    db "Overflow at %d!", 10, 0
    fmt_neg         db "Error: negative input %d", 10, 0
    fmt_header      db "=== Factorial Calculator ===", 10, 0
    
    ; ค่า factorial สูงสุดที่ 64-bit จะเก็บได้คือ 20!
    MAX_FACTORIAL   equ 20

section .text
    global main
    extern printf

; factorial(n) - คำนวณ n!
; พารามิเตอร์: rdi = n (signed integer)
; คืนค่า: rax = n! หรือ -1 ถ้า error
; clobbers: rcx, rdx
factorial:
    push rbp
    mov rbp, rsp
    push rbx
    sub rsp, 8                      ; alignment
    
    ; ตรวจสอบ input < 0
    test rdi, rdi
    js .error_negative
    
    ; ตรวจสอบ overflow
    cmp rdi, MAX_FACTORIAL
    jg .error_overflow
    
    ; เก็บ n
    mov rbx, rdi
    
    ; Base case: 0! = 1
    test rdi, rdi
    jz .base_case
    
    ; Recursive: n * (n-1)!
    lea rdi, [rbx-1]                ; n-1
    call factorial
    
    ; ตรวจสอบ error จาก recursive call
    cmp rax, -1
    je .propagate_error
    
    ; คูณ n * (n-1)!
    imul rax, rbx                   ; rax = n * factorial(n-1)
    jmp .done

.base_case:
    mov rax, 1                      ; 0! = 1
    jmp .done

.error_negative:
    mov rax, -1                     ; คืนค่า -1 เป็น error code
    jmp .done

.error_overflow:
    mov rax, -1
    jmp .done

.propagate_error:
    mov rax, -1

.done:
    add rsp, 8
    pop rbx
    pop rbp
    ret

main:
    push rbp
    mov rbp, rsp
    push rbx
    sub rsp, 8
    
    ; แสดง header
    lea rdi, [rel fmt_header]
    xor eax, eax
    call printf
    
    ; คำนวณ factorial 0 ถึง 21
    xor rbx, rbx

.loop:
    cmp rbx, 22
    jge .end_loop
    
    mov rdi, rbx
    call factorial
    
    ; ตรวจสอบ error
    cmp rax, -1
    je .show_overflow
    
    ; แสดงผลปกติ
    lea rdi, [rel fmt_result]
    mov rsi, rbx
    mov rdx, rax
    xor eax, eax
    call printf
    jmp .next

.show_overflow:
    lea rdi, [rel fmt_overflow]
    mov rsi, rbx
    xor eax, eax
    call printf

.next:
    inc rbx
    jmp .loop

.end_loop:
    xor eax, eax
    add rsp, 8
    pop rbx
    pop rbp
    ret
```

---

## 5. Tower of Hanoi {#tower-of-hanoi}

### ปัญหา Tower of Hanoi

ย้าย n จาน จาก peg A ไป peg C โดยใช้ peg B เป็นตัวช่วย โดยจาน ที่ใหญ่กว่าต้องอยู่ล่างเสมอ

```
hanoi(n, from, to, via):
    if n == 1:
        move disk from "from" to "to"
    else:
        hanoi(n-1, from, via, to)    ; ย้าย n-1 จาน ไป via
        move disk n from "from" to "to"
        hanoi(n-1, via, to, from)    ; ย้าย n-1 จาน จาก via ไป to
```

### Implementation ใน NASM

```nasm
; ไฟล์: tower_of_hanoi.asm
; คำอธิบาย: แก้ปัญหา Tower of Hanoi แบบ recursive
; คอมไพล์: nasm -f elf64 tower_of_hanoi.asm -o tower_of_hanoi.o
; ลิงก์:   gcc tower_of_hanoi.o -o tower_of_hanoi -no-pie

section .data
    ; ชื่อ pegs
    peg_A       db "A", 0
    peg_B       db "B", 0
    peg_C       db "C", 0
    
    ; Format strings
    fmt_move    db "Move disk %d from %s to %s", 10, 0
    fmt_header  db "=== Tower of Hanoi (n=%d disks) ===", 10, 0
    fmt_count   db "Total moves: %lld", 10, 0
    
    move_count  dq 0                ; นับจำนวนการเคลื่อนที่ทั้งหมด

section .text
    global main
    extern printf

; hanoi(n, from, to, via)
; พารามิเตอร์:
;   rdi = n (จำนวนจาน)
;   rsi = from (pointer to peg name)
;   rdx = to (pointer to peg name)
;   rcx = via (pointer to peg name)
hanoi:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    push r15
    sub rsp, 8                      ; alignment
    
    ; เก็บพารามิเตอร์
    mov rbx, rdi                    ; n
    mov r12, rsi                    ; from
    mov r13, rdx                    ; to
    mov r14, rcx                    ; via
    
    ; Base case: n == 1
    cmp rbx, 1
    jne .recursive_case
    
    ; แสดงการเคลื่อนที่จาน
    inc qword [rel move_count]
    lea rdi, [rel fmt_move]
    mov rsi, rbx                    ; disk number
    mov rdx, r12                    ; from
    mov rcx, r13                    ; to
    xor eax, eax
    call printf
    jmp .done

.recursive_case:
    ; hanoi(n-1, from, via, to)
    lea rdi, [rbx-1]                ; n-1
    mov rsi, r12                    ; from
    mov rdx, r14                    ; via (เปลี่ยน to กับ via)
    mov rcx, r13                    ; to
    call hanoi
    
    ; แสดงการเคลื่อนจาน n
    inc qword [rel move_count]
    lea rdi, [rel fmt_move]
    mov rsi, rbx                    ; disk n
    mov rdx, r12                    ; from
    mov rcx, r13                    ; to
    xor eax, eax
    call printf
    
    ; hanoi(n-1, via, to, from)
    lea rdi, [rbx-1]                ; n-1
    mov rsi, r14                    ; via
    mov rdx, r13                    ; to
    mov rcx, r12                    ; from
    call hanoi

.done:
    add rsp, 8
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

main:
    push rbp
    mov rbp, rsp
    sub rsp, 16
    
    ; แสดง header
    lea rdi, [rel fmt_header]
    mov rsi, 3                      ; n = 3 จาน
    xor eax, eax
    call printf
    
    ; รีเซ็ต counter
    mov qword [rel move_count], 0
    
    ; เรียก hanoi(3, A, C, B)
    mov rdi, 3
    lea rsi, [rel peg_A]
    lea rdx, [rel peg_C]
    lea rcx, [rel peg_B]
    call hanoi
    
    ; แสดงจำนวนการเคลื่อนที่ทั้งหมด
    lea rdi, [rel fmt_count]
    mov rsi, [rel move_count]
    xor eax, eax
    call printf
    
    xor eax, eax
    add rsp, 16
    pop rbp
    ret
```

**ผลลัพธ์ที่คาดหวัง (3 จาน = 7 moves):**
```
=== Tower of Hanoi (n=3 disks) ===
Move disk 1 from A to C
Move disk 2 from A to B
Move disk 1 from C to B
Move disk 3 from A to C
Move disk 1 from B to A
Move disk 2 from B to C
Move disk 1 from A to C
Total moves: 7
```

---

## 6. Binary Search แบบ Recursive {#binary-search-recursive}

### อัลกอริทึม Binary Search

```
binary_search(arr, low, high, target):
    if low > high: return -1 (not found)
    mid = (low + high) / 2
    if arr[mid] == target: return mid
    if arr[mid] < target:
        return binary_search(arr, mid+1, high, target)
    else:
        return binary_search(arr, low, mid-1, target)
```

### Implementation ใน NASM

```nasm
; ไฟล์: binary_search_recursive.asm
; คำอธิบาย: Binary search แบบ recursive บน array ที่ sort แล้ว
; คอมไพล์: nasm -f elf64 binary_search_recursive.asm -o binary_search_recursive.o
; ลิงก์:   gcc binary_search_recursive.o -o binary_search_recursive -no-pie

section .data
    ; Array ที่ sort แล้ว
    sorted_array    dq 2, 5, 8, 12, 16, 23, 38, 56, 72, 91
    array_size      equ 10
    
    fmt_found       db "Found %lld at index %lld", 10, 0
    fmt_not_found   db "%lld not found in array", 10, 0
    fmt_search      db "Searching for: %lld", 10, 0
    fmt_header      db "=== Recursive Binary Search ===", 10, 0
    fmt_array       db "Array: "
    fmt_element     db "%lld "
    fmt_newline     db 10, 0

section .text
    global main
    extern printf

; binary_search(arr, low, high, target)
; พารามิเตอร์:
;   rdi = arr (pointer to array)
;   rsi = low (index ต่ำสุด)
;   rdx = high (index สูงสุด)
;   rcx = target (ค่าที่ต้องการหา)
; คืนค่า: rax = index ที่พบ หรือ -1 ถ้าไม่พบ
binary_search:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    push r15
    sub rsp, 8
    
    ; เก็บพารามิเตอร์
    mov r12, rdi                    ; arr
    mov r13, rsi                    ; low
    mov r14, rdx                    ; high
    mov r15, rcx                    ; target
    
    ; Base case: low > high (ไม่พบ)
    cmp r13, r14
    jg .not_found
    
    ; คำนวณ mid = (low + high) / 2
    mov rbx, r13
    add rbx, r14                    ; rbx = low + high
    sar rbx, 1                      ; rbx = (low + high) / 2 (arithmetic shift right)
    
    ; โหลด arr[mid]
    mov rax, [r12 + rbx*8]          ; rax = arr[mid]
    
    ; ตรวจสอบ arr[mid] == target
    cmp rax, r15
    je .found_at_mid
    
    ; ถ้า arr[mid] < target ค้นหาทางขวา
    jl .search_right
    
    ; ถ้า arr[mid] > target ค้นหาทางซ้าย
    ; binary_search(arr, low, mid-1, target)
    mov rdi, r12                    ; arr
    mov rsi, r13                    ; low
    lea rdx, [rbx-1]                ; mid - 1
    mov rcx, r15                    ; target
    call binary_search
    jmp .done

.search_right:
    ; binary_search(arr, mid+1, high, target)
    mov rdi, r12                    ; arr
    lea rsi, [rbx+1]                ; mid + 1
    mov rdx, r14                    ; high
    mov rcx, r15                    ; target
    call binary_search
    jmp .done

.found_at_mid:
    mov rax, rbx                    ; คืนค่า index ที่พบ
    jmp .done

.not_found:
    mov rax, -1                     ; คืนค่า -1 (ไม่พบ)

.done:
    add rsp, 8
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

; แสดง array
print_array:
    push rbp
    mov rbp, rsp
    push rbx
    sub rsp, 8
    
    lea rdi, [rel fmt_array]
    xor eax, eax
    call printf
    
    xor rbx, rbx
.loop:
    cmp rbx, array_size
    jge .done
    
    lea rdi, [rel fmt_element]
    mov rsi, [rel sorted_array + rbx*8]
    xor eax, eax
    call printf
    
    inc rbx
    jmp .loop

.done:
    lea rdi, [rel fmt_newline]
    xor eax, eax
    call printf
    
    add rsp, 8
    pop rbx
    pop rbp
    ret

main:
    push rbp
    mov rbp, rsp
    push rbx
    sub rsp, 8
    
    ; แสดง header
    lea rdi, [rel fmt_header]
    xor eax, eax
    call printf
    
    ; แสดง array
    call print_array
    
    ; ค่าที่จะค้นหา
    mov rbx, 0

.test_loop:
    ; ค้นหาค่าหลายๆ ค่า: 23, 99, 2, 91, 50
    lea r15, [rel test_values]
    cmp rbx, 5
    jge .end_test
    
    mov r14, [r15 + rbx*8]          ; target = test_values[i]
    
    ; แสดงค่าที่จะค้นหา
    lea rdi, [rel fmt_search]
    mov rsi, r14
    xor eax, eax
    call printf
    
    ; เรียก binary search
    lea rdi, [rel sorted_array]
    xor rsi, rsi                    ; low = 0
    mov rdx, array_size - 1         ; high = size - 1
    mov rcx, r14                    ; target
    call binary_search
    
    ; แสดงผล
    cmp rax, -1
    je .not_found_msg
    
    lea rdi, [rel fmt_found]
    mov rsi, r14
    mov rdx, rax
    xor eax, eax
    call printf
    jmp .next_test

.not_found_msg:
    lea rdi, [rel fmt_not_found]
    mov rsi, r14
    xor eax, eax
    call printf

.next_test:
    inc rbx
    jmp .test_loop

.end_test:
    xor eax, eax
    add rsp, 8
    pop rbx
    pop rbp
    ret

section .data
    test_values dq 23, 99, 2, 91, 50   ; ค่าที่ต้องการทดสอบ
```

---

## 7. Merge Sort แบบ Recursive {#merge-sort-recursive}

### อัลกอริทึม Merge Sort

```
merge_sort(arr, left, right):
    if left >= right: return
    mid = (left + right) / 2
    merge_sort(arr, left, mid)
    merge_sort(arr, mid+1, right)
    merge(arr, left, mid, right)
```

### Implementation ใน NASM

```nasm
; ไฟล์: merge_sort.asm
; คำอธิบาย: Merge Sort แบบ recursive ใน x86-64 assembly
; คอมไพล์: nasm -f elf64 merge_sort.asm -o merge_sort.o
; ลิงก์:   gcc merge_sort.o -o merge_sort -no-pie

section .data
    fmt_before  db "Before sort: "
    fmt_after   db "After sort:  "
    fmt_elem    db "%lld "
    fmt_nl      db 10, 0
    fmt_header  db "=== Merge Sort Demo ===", 10, 0

section .bss
    temp_arr    resq 1000           ; temporary array สำหรับ merge

section .text
    global main
    extern printf, memcpy

; merge(arr, left, mid, right)
; รวม 2 subarray ที่ sort แล้ว
; rdi = arr, rsi = left, rdx = mid, rcx = right
merge:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    push r15
    sub rsp, 24
    
    ; เก็บพารามิเตอร์
    mov r12, rdi                    ; arr
    mov r13, rsi                    ; left
    mov r14, rdx                    ; mid
    mov r15, rcx                    ; right
    
    ; คัดลอก left subarray ไปยัง temp
    ; temp[0..n-1] = arr[left..right]
    mov rdi, rel temp_arr           ; dest
    lea rsi, [r12 + r13*8]          ; src = arr + left*8
    mov rdx, r15
    sub rdx, r13
    inc rdx                         ; count = right - left + 1
    imul rdx, 8                     ; size in bytes
    call memcpy
    
    ; i = 0 (temp index ฝั่งซ้าย)
    ; j = mid - left + 1 (temp index ฝั่งขวา)
    ; k = left (arr index)
    xor rbx, rbx                    ; i = 0
    mov [rbp-8], rbx                ; เก็บ i
    
    mov rcx, r14
    sub rcx, r13
    inc rcx                         ; j = mid - left + 1
    mov [rbp-16], rcx               ; เก็บ j
    
    mov [rbp-24], r13               ; k = left
    
    ; count_left = mid - left + 1
    ; count_right = right - mid
    
.merge_loop:
    ; ตรวจสอบว่า i ยัง valid หรือไม่
    ; i < count_left AND j <= right - left
    mov rax, [rbp-8]                ; i
    mov rcx, r14
    sub rcx, r13
    inc rcx                         ; count_left = mid - left + 1
    cmp rax, rcx
    jge .copy_remaining_right
    
    mov rcx, [rbp-16]               ; j
    mov rdx, r15
    sub rdx, r13                    ; count = right - left
    cmp rcx, rdx
    jg .copy_remaining_left
    
    ; เปรียบเทียบ temp[i] และ temp[j]
    mov rax, [rbp-8]
    mov rcx, [rbp-16]
    
    mov rsi, [rel temp_arr + rax*8] ; temp[i]
    mov rdx, [rel temp_arr + rcx*8] ; temp[j]
    
    cmp rsi, rdx
    jg .take_right
    
    ; arr[k] = temp[i]; i++
    mov rax, [rbp-8]
    mov rdx, [rel temp_arr + rax*8]
    mov rax, [rbp-24]               ; k
    mov [r12 + rax*8], rdx
    inc qword [rbp-8]               ; i++
    inc qword [rbp-24]              ; k++
    jmp .merge_loop

.take_right:
    ; arr[k] = temp[j]; j++
    mov rcx, [rbp-16]
    mov rdx, [rel temp_arr + rcx*8]
    mov rax, [rbp-24]               ; k
    mov [r12 + rax*8], rdx
    inc qword [rbp-16]              ; j++
    inc qword [rbp-24]              ; k++
    jmp .merge_loop

.copy_remaining_left:
    ; คัดลอก temp[i..count_left-1] ที่เหลือ
    mov rax, [rbp-8]
    mov rcx, r14
    sub rcx, r13
    inc rcx
    cmp rax, rcx
    jge .done
    
    mov rdx, [rel temp_arr + rax*8]
    mov rax, [rbp-24]
    mov [r12 + rax*8], rdx
    inc qword [rbp-8]
    inc qword [rbp-24]
    jmp .copy_remaining_left

.copy_remaining_right:
    ; คัดลอก temp[j..count-1] ที่เหลือ
    mov rcx, [rbp-16]
    mov rdx, r15
    sub rdx, r13
    cmp rcx, rdx
    jg .done
    
    mov rsi, [rel temp_arr + rcx*8]
    mov rax, [rbp-24]
    mov [r12 + rax*8], rsi
    inc qword [rbp-16]
    inc qword [rbp-24]
    jmp .copy_remaining_right

.done:
    add rsp, 24
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

; merge_sort(arr, left, right)
; rdi = arr, rsi = left, rdx = right
merge_sort:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    sub rsp, 8
    
    mov r12, rdi                    ; arr
    mov r13, rsi                    ; left
    mov r14, rdx                    ; right
    
    ; Base case: left >= right
    cmp r13, r14
    jge .done
    
    ; คำนวณ mid
    mov rbx, r13
    add rbx, r14
    sar rbx, 1                      ; mid = (left + right) / 2
    
    ; merge_sort(arr, left, mid)
    mov rdi, r12
    mov rsi, r13
    mov rdx, rbx
    call merge_sort
    
    ; merge_sort(arr, mid+1, right)
    mov rdi, r12
    lea rsi, [rbx+1]
    mov rdx, r14
    call merge_sort
    
    ; merge(arr, left, mid, right)
    mov rdi, r12
    mov rsi, r13
    mov rdx, rbx
    mov rcx, r14
    call merge

.done:
    add rsp, 8
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

; print_array(arr, size)
print_array:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    sub rsp, 8
    
    mov r12, rdi                    ; arr
    mov rbx, rsi                    ; size

    xor r13, r13                    ; i = 0
.loop:
    cmp r13, rbx
    jge .done
    
    lea rdi, [rel fmt_elem]
    mov rsi, [r12 + r13*8]
    xor eax, eax
    call printf
    
    inc r13
    jmp .loop
    
.done:
    lea rdi, [rel fmt_nl]
    xor eax, eax
    call printf
    
    add rsp, 8
    pop r12
    pop rbx
    pop rbp
    ret

main:
    push rbp
    mov rbp, rsp
    sub rsp, 16
    
    lea rdi, [rel fmt_header]
    xor eax, eax
    call printf
    
    ; แสดง array ก่อน sort
    lea rdi, [rel fmt_before]
    xor eax, eax
    call printf
    
    lea rdi, [rel test_array]
    mov rsi, 8
    call print_array
    
    ; เรียก merge sort
    lea rdi, [rel test_array]
    xor rsi, rsi                    ; left = 0
    mov rdx, 7                      ; right = size - 1
    call merge_sort
    
    ; แสดง array หลัง sort
    lea rdi, [rel fmt_after]
    xor eax, eax
    call printf
    
    lea rdi, [rel test_array]
    mov rsi, 8
    call print_array
    
    xor eax, eax
    add rsp, 16
    pop rbp
    ret

section .data
    test_array  dq 64, 34, 25, 12, 22, 11, 90, 45
```

---

## 8. Tree Traversal {#tree-traversal}

### Binary Tree Structure

```
โครงสร้าง Node ใน memory:
┌──────────────────────────────┐
│  value    (8 bytes) offset 0 │
│  left ptr (8 bytes) offset 8 │
│  right ptr(8 bytes) offset 16│
└──────────────────────────────┘
```

### Implementation Tree Traversal

```nasm
; ไฟล์: tree_traversal.asm
; คำอธิบาย: Inorder, Preorder, Postorder tree traversal แบบ recursive
; คอมไพล์: nasm -f elf64 tree_traversal.asm -o tree_traversal.o
; ลิงก์:   gcc tree_traversal.o -o tree_traversal -no-pie

section .data
    fmt_value   db "%lld "
    fmt_nl      db 10, 0
    fmt_in      db "Inorder:   "
    fmt_pre     db "Preorder:  "
    fmt_post    db "Postorder: "
    fmt_header  db "=== Binary Tree Traversal ===", 10, 0
    
    ; คำอธิบาย offsets ของ tree node
    NODE_VALUE  equ 0
    NODE_LEFT   equ 8
    NODE_RIGHT  equ 16
    NODE_SIZE   equ 24

; สร้าง tree ไว้ใน .data section (static)
;
;        4
;       / \
;      2   6
;     / \ / \
;    1  3 5  7
;
    node1:  dq 1, 0, 0             ; value=1, left=NULL, right=NULL
    node2:  dq 2, node1, node3     ; value=2, left=node1, right=node3
    node3:  dq 3, 0, 0
    node4:  dq 4, node2, node6     ; root
    node5:  dq 5, 0, 0
    node6:  dq 6, node5, node7
    node7:  dq 7, 0, 0

section .text
    global main
    extern printf

; inorder_traversal(node)
; รูปแบบ: Left → Root → Right
; rdi = node pointer (NULL ถ้าไม่มี)
inorder_traversal:
    push rbp
    mov rbp, rsp
    push rbx
    sub rsp, 8
    
    ; Base case: NULL node
    test rdi, rdi
    jz .done
    
    mov rbx, rdi                    ; เก็บ node pointer
    
    ; ทำ Left subtree ก่อน
    mov rdi, [rbx + NODE_LEFT]
    call inorder_traversal
    
    ; แสดง current node value
    lea rdi, [rel fmt_value]
    mov rsi, [rbx + NODE_VALUE]
    xor eax, eax
    call printf
    
    ; ทำ Right subtree
    mov rdi, [rbx + NODE_RIGHT]
    call inorder_traversal

.done:
    add rsp, 8
    pop rbx
    pop rbp
    ret

; preorder_traversal(node)
; รูปแบบ: Root → Left → Right
preorder_traversal:
    push rbp
    mov rbp, rsp
    push rbx
    sub rsp, 8
    
    test rdi, rdi
    jz .done
    
    mov rbx, rdi
    
    ; แสดง current node ก่อน
    lea rdi, [rel fmt_value]
    mov rsi, [rbx + NODE_VALUE]
    xor eax, eax
    call printf
    
    ; Left subtree
    mov rdi, [rbx + NODE_LEFT]
    call preorder_traversal
    
    ; Right subtree
    mov rdi, [rbx + NODE_RIGHT]
    call preorder_traversal

.done:
    add rsp, 8
    pop rbx
    pop rbp
    ret

; postorder_traversal(node)
; รูปแบบ: Left → Right → Root
postorder_traversal:
    push rbp
    mov rbp, rsp
    push rbx
    sub rsp, 8
    
    test rdi, rdi
    jz .done
    
    mov rbx, rdi
    
    ; Left subtree ก่อน
    mov rdi, [rbx + NODE_LEFT]
    call postorder_traversal
    
    ; Right subtree
    mov rdi, [rbx + NODE_RIGHT]
    call postorder_traversal
    
    ; แสดง current node หลังสุด
    lea rdi, [rel fmt_value]
    mov rsi, [rbx + NODE_VALUE]
    xor eax, eax
    call printf

.done:
    add rsp, 8
    pop rbx
    pop rbp
    ret

main:
    push rbp
    mov rbp, rsp
    
    lea rdi, [rel fmt_header]
    xor eax, eax
    call printf
    
    ; Inorder traversal (ควรได้: 1 2 3 4 5 6 7)
    lea rdi, [rel fmt_in]
    xor eax, eax
    call printf
    
    lea rdi, [rel node4]            ; root
    call inorder_traversal
    
    lea rdi, [rel fmt_nl]
    xor eax, eax
    call printf
    
    ; Preorder traversal (ควรได้: 4 2 1 3 6 5 7)
    lea rdi, [rel fmt_pre]
    xor eax, eax
    call printf
    
    lea rdi, [rel node4]
    call preorder_traversal
    
    lea rdi, [rel fmt_nl]
    xor eax, eax
    call printf
    
    ; Postorder traversal (ควรได้: 1 3 2 5 7 6 4)
    lea rdi, [rel fmt_post]
    xor eax, eax
    call printf
    
    lea rdi, [rel node4]
    call postorder_traversal
    
    lea rdi, [rel fmt_nl]
    xor eax, eax
    call printf
    
    xor eax, eax
    pop rbp
    ret
```

---

## 9. Backtracking Algorithm {#backtracking}

### N-Queens Problem

วาง N ราชินีบนกระดาน N×N โดยที่ราชินีไม่โจมตีกัน

```nasm
; ไฟล์: n_queens.asm
; คำอธิบาย: แก้ปัญหา N-Queens ด้วย backtracking recursive
; คอมไพล์: nasm -f elf64 n_queens.asm -o n_queens.o
; ลิงก์:   gcc n_queens.o -o n_queens -no-pie

section .data
    N_SIZE      equ 8               ; ขนาดกระดาน 8×8
    fmt_queen   db "Q "
    fmt_empty   db ". "
    fmt_nl      db 10, 0
    fmt_sep     db "--------", 10, 0
    fmt_count   db "Solutions found: %d", 10, 0
    fmt_header  db "=== N-Queens (N=%d) ===", 10, 0
    
    solution_count  dd 0            ; จำนวน solutions ที่พบ

section .bss
    board       resb N_SIZE         ; board[i] = column ของ queen ในแถว i
    
section .text
    global main
    extern printf

; is_safe(board, row, col)
; ตรวจสอบว่าวาง queen ที่ (row, col) ปลอดภัยหรือไม่
; rdi = board, rsi = row, rdx = col
; คืนค่า: rax = 1 ถ้าปลอดภัย, 0 ถ้าไม่ปลอดภัย
is_safe:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    sub rsp, 8
    
    mov r12, rdi                    ; board
    mov r13, rsi                    ; row
    ; rdx = col ที่จะตรวจสอบ
    
    ; ตรวจสอบทุกแถวที่วางแล้ว (0 ถึง row-1)
    xor rbx, rbx                    ; i = 0

.check_loop:
    cmp rbx, r13
    jge .safe                       ; ตรวจสอบครบแล้ว
    
    ; โหลด column ของ queen ในแถว i
    movsx rax, byte [r12 + rbx]     ; col_i = board[i]
    
    ; ตรวจสอบ same column
    cmp rax, rdx
    je .not_safe
    
    ; ตรวจสอบ diagonal
    ; |col_i - col| == |i - row|
    mov rcx, rax
    sub rcx, rdx                    ; col_i - col
    
    mov r9, rbx
    sub r9, r13                     ; i - row
    
    ; หาค่า absolute value
    test rcx, rcx
    jns .pos_col_diff
    neg rcx
.pos_col_diff:
    
    test r9, r9
    jns .pos_row_diff
    neg r9
.pos_row_diff:
    
    cmp rcx, r9
    je .not_safe
    
    inc rbx
    jmp .check_loop

.safe:
    mov rax, 1
    jmp .done

.not_safe:
    xor eax, eax

.done:
    add rsp, 8
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

; print_board(board)
print_board:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    sub rsp, 8
    
    mov r12, rdi                    ; board
    
    xor rbx, rbx                   ; row = 0
.row_loop:
    cmp rbx, N_SIZE
    jge .done
    
    xor r13, r13                   ; col = 0
.col_loop:
    cmp r13, N_SIZE
    jge .end_row
    
    ; ตรวจสอบว่า queen อยู่ที่ (row, col) หรือไม่
    movsx rax, byte [r12 + rbx]
    cmp rax, r13
    jne .print_empty
    
    lea rdi, [rel fmt_queen]
    xor eax, eax
    call printf
    jmp .next_col

.print_empty:
    lea rdi, [rel fmt_empty]
    xor eax, eax
    call printf

.next_col:
    inc r13
    jmp .col_loop

.end_row:
    lea rdi, [rel fmt_nl]
    xor eax, eax
    call printf
    
    inc rbx
    jmp .row_loop

.done:
    lea rdi, [rel fmt_sep]
    xor eax, eax
    call printf
    
    add rsp, 8
    pop r12
    pop rbx
    pop rbp
    ret

; solve_queens(board, row)
; rdi = board, rsi = row
solve_queens:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    sub rsp, 8
    
    mov r12, rdi                    ; board
    mov r13, rsi                    ; row
    
    ; Base case: ทุกแถวถูกวางแล้ว
    cmp r13, N_SIZE
    jge .solution_found
    
    ; ลองวาง queen ในแต่ละ column ของแถวนี้
    xor rbx, rbx                   ; col = 0
.try_col:
    cmp rbx, N_SIZE
    jge .done
    
    ; ตรวจสอบว่าปลอดภัยหรือไม่
    mov rdi, r12
    mov rsi, r13
    mov rdx, rbx
    call is_safe
    
    test rax, rax
    jz .skip_col                    ; ไม่ปลอดภัย ข้ามไป
    
    ; วาง queen
    mov byte [r12 + r13], bl        ; board[row] = col
    
    ; Recursive: แก้แถวถัดไป
    mov rdi, r12
    lea rsi, [r13+1]
    call solve_queens
    
    ; Backtrack: ลบ queen ออก (ไม่จำเป็นต้องทำใน array แบบนี้)

.skip_col:
    inc rbx
    jmp .try_col

.solution_found:
    ; เพิ่ม counter
    inc dword [rel solution_count]
    
    ; แสดง solution แรกๆ (ถ้า count <= 3)
    mov eax, [rel solution_count]
    cmp eax, 4
    jge .done
    
    mov rdi, r12
    call print_board

.done:
    add rsp, 8
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

main:
    push rbp
    mov rbp, rsp
    
    lea rdi, [rel fmt_header]
    mov rsi, N_SIZE
    xor eax, eax
    call printf
    
    ; รีเซ็ต solution counter
    mov dword [rel solution_count], 0
    
    ; เริ่ม solve จากแถว 0
    lea rdi, [rel board]
    xor rsi, rsi
    call solve_queens
    
    ; แสดงจำนวน solutions ทั้งหมด
    lea rdi, [rel fmt_count]
    mov esi, [rel solution_count]
    xor eax, eax
    call printf
    
    xor eax, eax
    pop rbp
    ret
```

---

## 10. Memoization ใน Assembly {#memoization}

### Fibonacci พร้อม Memoization

```nasm
; ไฟล์: fibonacci_memo.asm
; คำอธิบาย: Fibonacci แบบ recursive พร้อม memoization
; ใช้ array เก็บผลลัพธ์ที่คำนวณแล้ว
; คอมไพล์: nasm -f elf64 fibonacci_memo.asm -o fibonacci_memo.o
; ลิงก์:   gcc fibonacci_memo.o -o fibonacci_memo -no-pie

section .data
    fmt_result  db "fib(%d) = %lld (calls: %lld)", 10, 0
    fmt_header  db "=== Fibonacci with Memoization ===", 10, 0
    MEMO_SIZE   equ 100             ; เก็บผลได้สูงสุด 100 ค่า
    call_count  dq 0
    
section .bss
    memo        resq MEMO_SIZE      ; array เก็บ memo (0 = ยังไม่ได้คำนวณ)
    memo_valid  resb MEMO_SIZE      ; flag บอกว่าค่านี้คำนวณแล้วหรือยัง

section .text
    global main
    extern printf, memset

; init_memo() - initialize memo array
init_memo:
    ; ตั้งค่า memo ทั้งหมดเป็น 0
    lea rdi, [rel memo]
    xor esi, esi
    mov edx, MEMO_SIZE * 8
    call memset
    
    ; ตั้งค่า valid flags เป็น 0 (ยังไม่ได้คำนวณ)
    lea rdi, [rel memo_valid]
    xor esi, esi
    mov edx, MEMO_SIZE
    call memset
    ret

; fib_memo(n) - fibonacci พร้อม memoization
; rdi = n
; คืนค่า: rax = fib(n)
fib_memo:
    push rbp
    mov rbp, rsp
    push rbx
    sub rsp, 8
    
    mov rbx, rdi                    ; n
    
    ; เพิ่ม call counter
    inc qword [rel call_count]
    
    ; Base cases
    test rbx, rbx
    jz .return_zero
    
    cmp rbx, 1
    je .return_one
    
    ; ตรวจสอบว่ามี memo แล้วหรือยัง
    movzx rax, byte [rel memo_valid + rbx]  ; โหลด valid flag
    test rax, rax
    jnz .return_memo                ; ถ้า valid != 0 คือมี memo แล้ว
    
    ; ยังไม่มี memo ต้องคำนวณ
    ; fib(n-1)
    lea rdi, [rbx-1]
    call fib_memo
    mov [rbp-16], rax               ; เก็บ fib(n-1)
    
    ; fib(n-2)
    lea rdi, [rbx-2]
    call fib_memo
    
    ; fib(n) = fib(n-1) + fib(n-2)
    add rax, [rbp-16]
    
    ; บันทึกลง memo
    mov [rel memo + rbx*8], rax     ; memo[n] = result
    mov byte [rel memo_valid + rbx], 1  ; valid[n] = 1
    
    jmp .done

.return_zero:
    xor eax, eax
    jmp .done

.return_one:
    mov eax, 1
    jmp .done

.return_memo:
    mov rax, [rel memo + rbx*8]     ; return memo[n]

.done:
    add rsp, 8
    pop rbx
    pop rbp
    ret

main:
    push rbp
    mov rbp, rsp
    push rbx
    sub rsp, 8
    
    ; Initialize
    call init_memo
    
    lea rdi, [rel fmt_header]
    xor eax, eax
    call printf
    
    ; คำนวณ fib แต่ละค่า โดย reset call_count ทุกครั้ง
    xor rbx, rbx

.loop:
    cmp rbx, 30                     ; คำนวณ fib(0) ถึง fib(29)
    jge .done
    
    ; Reset memo และ call_count
    call init_memo
    mov qword [rel call_count], 0
    
    mov rdi, rbx
    call fib_memo
    
    ; แสดงผล
    mov [rbp-16], rax
    lea rdi, [rel fmt_result]
    mov rsi, rbx
    mov rdx, [rbp-16]
    mov rcx, [rel call_count]
    xor eax, eax
    call printf
    
    inc rbx
    jmp .loop

.done:
    xor eax, eax
    add rsp, 8
    pop rbx
    pop rbp
    ret
```

---

## 11. Tail Recursion Optimization {#tail-recursion}

### Tail Recursion คืออะไร?

**Tail recursion** คือการเรียก recursive ที่เป็น **การกระทำสุดท้าย** ของฟังก์ชัน (ไม่มีการคำนวณหลัง recursive call)

Compiler สามารถ optimize เป็น loop ได้ ทำให้ไม่ต้องสร้าง stack frame ใหม่

```nasm
; ไฟล์: tail_recursion.asm
; คำอธิบาย: เปรียบเทียบ regular recursion vs tail recursion optimization
; คอมไพล์: nasm -f elf64 tail_recursion.asm -o tail_recursion.o
; ลิงก์:   gcc tail_recursion.o -o tail_recursion -no-pie

section .data
    fmt_result  db "Result: %lld", 10, 0
    fmt_header  db "=== Tail Recursion Optimization ===", 10, 0
    fmt_normal  db "Normal recursion:  "
    fmt_tail    db "Tail recursion:    "
    fmt_loop    db "Converted to loop: "

section .text
    global main
    extern printf

; factorial_normal(n) - factorial ปกติ (ไม่ใช่ tail recursive)
; รูปแบบ: n * factorial(n-1)
; การเรียกสุดท้ายคือ imul ไม่ใช่ recursive call
factorial_normal:
    push rbp
    mov rbp, rsp
    push rbx
    sub rsp, 8
    
    mov rbx, rdi
    
    ; Base case
    cmp rbx, 1
    jle .base
    
    ; n * factorial(n-1) -- ไม่ใช่ tail recursive
    lea rdi, [rbx-1]
    call factorial_normal
    imul rax, rbx                   ; คำนวณหลัง recursive call
    jmp .done

.base:
    mov rax, 1

.done:
    add rsp, 8
    pop rbx
    pop rbp
    ret

; factorial_tail_helper(n, accumulator)
; rdi = n, rsi = accumulator
; Tail recursive: ส่ง accumulator ไปสะสม
factorial_tail_helper:
    push rbp
    mov rbp, rsp
    sub rsp, 16
    
    ; Base case: n <= 1, return accumulator
    cmp rdi, 1
    jle .base
    
    ; factorial_tail_helper(n-1, n * acc)
    mov rax, rdi                    ; rax = n
    imul rax, rsi                   ; rax = n * acc
    
    ; ปรับ parameters สำหรับ next call
    dec rdi                         ; n-1
    mov rsi, rax                    ; acc = n * old_acc
    
    ; Tail call optimization: ใช้ jmp แทน call
    ; (compiler จะทำแบบนี้อัตโนมัติ)
    mov rsp, rbp
    pop rbp
    jmp factorial_tail_helper       ; TAIL CALL - ไม่สร้าง stack frame ใหม่

.base:
    mov rax, rsi                    ; return accumulator

    mov rsp, rbp
    pop rbp
    ret

; factorial_tail(n) - wrapper ที่เรียก tail recursive version
factorial_tail:
    push rbp
    mov rbp, rsp
    
    ; factorial_tail_helper(n, 1)
    mov rsi, 1                      ; accumulator เริ่มต้น = 1
    call factorial_tail_helper
    
    pop rbp
    ret

; factorial_loop(n) - เวอร์ชันที่แปลง recursion เป็น loop
; นี่คือ manual optimization ที่ compiler ทำให้
factorial_loop:
    push rbp
    mov rbp, rsp
    
    mov rax, 1                      ; result = 1
    
    cmp rdi, 1
    jle .done                       ; ถ้า n <= 1 return 1
    
.loop:
    imul rax, rdi                   ; result *= n
    dec rdi                         ; n--
    cmp rdi, 1
    jg .loop                        ; ทำซ้ำจนกว่า n <= 1

.done:
    pop rbp
    ret

main:
    push rbp
    mov rbp, rsp
    push rbx
    sub rsp, 8
    
    lea rdi, [rel fmt_header]
    xor eax, eax
    call printf
    
    ; ทดสอบ factorial(10) ด้วยสามวิธี
    mov rbx, 10                     ; n = 10
    
    ; Normal recursion
    lea rdi, [rel fmt_normal]
    xor eax, eax
    call printf
    
    mov rdi, rbx
    call factorial_normal
    
    lea rdi, [rel fmt_result]
    mov rsi, rax
    xor eax, eax
    call printf
    
    ; Tail recursion
    lea rdi, [rel fmt_tail]
    xor eax, eax
    call printf
    
    mov rdi, rbx
    call factorial_tail
    
    lea rdi, [rel fmt_result]
    mov rsi, rax
    xor eax, eax
    call printf
    
    ; Loop version
    lea rdi, [rel fmt_loop]
    xor eax, eax
    call printf
    
    mov rdi, rbx
    call factorial_loop
    
    lea rdi, [rel fmt_result]
    mov rsi, rax
    xor eax, eax
    call printf
    
    xor eax, eax
    add rsp, 8
    pop rbx
    pop rbp
    ret
```

---

## 12. Stack Depth Limits {#stack-limits}

### การตรวจสอบ Stack Overflow

```nasm
; ไฟล์: stack_guard.asm
; คำอธิบาย: ตรวจสอบ stack depth และป้องกัน stack overflow
; คอมไพล์: nasm -f elf64 stack_guard.asm -o stack_guard.o
; ลิงก์:   gcc stack_guard.o -o stack_guard -no-pie

section .data
    fmt_overflow    db "Stack overflow at depth %d! RSP=0x%lx", 10, 0
    fmt_depth       db "Max safe depth reached: %d", 10, 0
    fmt_result      db "Result computed at depth %d: %lld", 10, 0
    
    ; กำหนด stack limit ขนาดต่างๆ
    STACK_LIMIT     equ 8388608     ; 8MB default Linux stack
    SAFETY_MARGIN   equ 4096        ; เหลือไว้ 4KB เป็น safety margin
    initial_rsp     dq 0            ; เก็บ RSP เริ่มต้น

section .text
    global main
    extern printf

; check_stack_overflow() - ตรวจสอบว่า stack จะ overflow หรือไม่
; คืนค่า: rax = 1 ถ้าปลอดภัย, 0 ถ้าจะ overflow
check_stack_safe:
    ; คำนวณพื้นที่ที่ใช้ไปแล้ว
    mov rax, [rel initial_rsp]
    sub rax, rsp                    ; used = initial_rsp - current_rsp
    
    ; ตรวจสอบว่ายังเหลือพอ
    mov rcx, STACK_LIMIT
    sub rcx, SAFETY_MARGIN          ; safe limit
    cmp rax, rcx
    jge .overflow
    
    mov rax, 1                      ; ปลอดภัย
    ret

.overflow:
    xor eax, eax                    ; ไม่ปลอดภัย
    ret

; safe_recursive(n, depth)
; rdi = n, rsi = depth
safe_recursive:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    sub rsp, 8
    
    mov rbx, rdi
    mov r12, rsi
    
    ; ตรวจสอบ stack ก่อน recurse
    call check_stack_safe
    test rax, rax
    jz .stack_overflow
    
    ; Base case
    test rbx, rbx
    jz .base
    
    ; Recursive call
    lea rdi, [rbx-1]
    lea rsi, [r12+1]
    call safe_recursive
    add rax, 1                      ; สะสมผล
    jmp .done

.base:
    xor eax, eax
    jmp .done

.stack_overflow:
    ; แจ้งเตือน overflow
    lea rdi, [rel fmt_overflow]
    mov rsi, r12                    ; depth
    mov rdx, rsp                    ; current RSP
    xor eax, eax
    call printf
    
    mov rax, -1                     ; คืนค่า error

.done:
    add rsp, 8
    pop r12
    pop rbx
    pop rbp
    ret

main:
    push rbp
    mov rbp, rsp
    
    ; เก็บ RSP เริ่มต้น
    mov [rel initial_rsp], rsp
    
    ; ทดสอบ safe recursion
    mov rdi, 1000                   ; depth 1000
    xor rsi, rsi
    call safe_recursive
    
    cmp rax, -1
    je .overflow_detected
    
    lea rdi, [rel fmt_result]
    mov rsi, 1000
    mov rdx, rax
    xor eax, eax
    call printf
    jmp .done

.overflow_detected:
    lea rdi, [rel fmt_depth]
    mov rsi, 1000
    xor eax, eax
    call printf

.done:
    xor eax, eax
    pop rbp
    ret
```

---

## 13. การแปลง Recursion เป็น Iteration {#recursion-to-iteration}

### วิธีการแปลง Recursive เป็น Iterative ด้วย Explicit Stack

```nasm
; ไฟล์: recursion_to_iteration.asm
; คำอธิบาย: แปลง recursive DFS เป็น iterative โดยใช้ explicit stack
; คอมไพล์: nasm -f elf64 recursion_to_iteration.asm -o recursion_to_iteration.o
; ลิงก์:   gcc recursion_to_iteration.o -o recursion_to_iteration -no-pie

section .data
    fmt_result  db "Node %lld visited", 10, 0
    fmt_header  db "=== Recursion to Iteration ===", 10, 0
    fmt_recur   db "--- Recursive DFS ---", 10, 0
    fmt_iter    db "--- Iterative DFS ---", 10, 0
    
    ; Tree nodes (value, left_index, right_index)
    ; -1 = NULL
    tree_data:
    ; Index: value, left, right
    dq  1,  1,  2       ; Node 0: value=1, left=node1, right=node2
    dq  2,  3,  4       ; Node 1: value=2, left=node3, right=node4
    dq  3,  5,  6       ; Node 2: value=3, left=node5, right=node6
    dq  4, -1, -1       ; Node 3: value=4, leaf
    dq  5, -1, -1       ; Node 4: value=5, leaf
    dq  6, -1, -1       ; Node 5: value=6, leaf
    dq  7, -1, -1       ; Node 6: value=7, leaf
    
    NODE_VALUE  equ 0
    NODE_LEFT   equ 8
    NODE_RIGHT  equ 16
    NODE_STRIDE equ 24              ; ขนาด 1 node = 24 bytes

section .bss
    ; Stack สำหรับ iterative version
    explicit_stack  resq 100        ; stack ขนาด 100 elements
    stack_top       resq 1          ; stack pointer

section .text
    global main
    extern printf

; recursive_dfs(node_index)
; rdi = node index
recursive_dfs:
    push rbp
    mov rbp, rsp
    push rbx
    sub rsp, 8
    
    mov rbx, rdi
    
    ; Base case: -1 = NULL
    cmp rbx, -1
    je .done
    
    ; คำนวณ address ของ node
    imul rax, rbx, NODE_STRIDE
    lea r9, [rel tree_data + rax]
    
    ; แสดงค่าของ node
    lea rdi, [rel fmt_result]
    mov rsi, [r9 + NODE_VALUE]
    xor eax, eax
    call printf
    
    ; Recursive left
    imul rax, rbx, NODE_STRIDE
    lea r9, [rel tree_data + rax]
    mov rdi, [r9 + NODE_LEFT]
    call recursive_dfs
    
    ; Recursive right
    imul rax, rbx, NODE_STRIDE
    lea r9, [rel tree_data + rax]
    mov rdi, [r9 + NODE_RIGHT]
    call recursive_dfs

.done:
    add rsp, 8
    pop rbx
    pop rbp
    ret

; iterative_dfs(root_index)
; rdi = root node index
; ใช้ explicit stack แทน call stack
iterative_dfs:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    sub rsp, 8
    
    ; Initialize explicit stack
    mov qword [rel stack_top], 0    ; stack_top = 0 (stack ว่าง)
    
    ; Push root ลง stack
    cmp rdi, -1
    je .done
    
    lea r12, [rel explicit_stack]
    mov qword [r12], rdi            ; push root
    mov qword [rel stack_top], 1    ; top = 1

.loop:
    ; ตรวจสอบว่า stack ว่างหรือไม่
    mov rax, [rel stack_top]
    test rax, rax
    jz .done
    
    ; Pop จาก stack
    dec qword [rel stack_top]
    mov rbx, [rel stack_top]
    mov rbx, [r12 + rbx*8]         ; node_index = pop()
    
    ; ถ้าเป็น NULL ข้ามไป
    cmp rbx, -1
    je .loop
    
    ; คำนวณ address ของ node
    imul rax, rbx, NODE_STRIDE
    lea r9, [rel tree_data + rax]
    
    ; แสดงค่า node
    lea rdi, [rel fmt_result]
    mov rsi, [r9 + NODE_VALUE]
    xor eax, eax
    call printf
    
    ; Push right ก่อน left (เพราะ stack LIFO, left จะถูก process ก่อน)
    
    ; Push right child
    mov rcx, [rel stack_top]
    mov rdx, [r9 + NODE_RIGHT]
    mov [r12 + rcx*8], rdx
    inc qword [rel stack_top]
    
    ; Push left child
    mov rcx, [rel stack_top]
    mov rdx, [r9 + NODE_LEFT]
    mov [r12 + rcx*8], rdx
    inc qword [rel stack_top]
    
    jmp .loop

.done:
    add rsp, 8
    pop r12
    pop rbx
    pop rbp
    ret

main:
    push rbp
    mov rbp, rsp
    
    lea rdi, [rel fmt_header]
    xor eax, eax
    call printf
    
    ; Recursive DFS
    lea rdi, [rel fmt_recur]
    xor eax, eax
    call printf
    
    xor rdi, rdi                    ; root = index 0
    call recursive_dfs
    
    ; Iterative DFS
    lea rdi, [rel fmt_iter]
    xor eax, eax
    call printf
    
    xor rdi, rdi                    ; root = index 0
    call iterative_dfs
    
    xor eax, eax
    pop rbp
    ret
```

---

## 14. Trampolining Technique {#trampolining}

### Trampolining คืออะไร?

**Trampolining** เป็นเทคนิคที่แปลง recursive function ให้ทำงานแบบ **iterative** โดยไม่ต้องแก้ไข logic ภายใน ฟังก์ชันจะ return "thunk" (ฟังก์ชันที่จะถูกเรียกต่อ) แทนที่จะ call ตัวเองโดยตรง

```nasm
; ไฟล์: trampolining.asm
; คำอธิบาย: Trampolining technique สำหรับ tail recursion ที่ลึกมากๆ
; คอมไพล์: nasm -f elf64 trampolining.asm -o trampolining.o
; ลิงก์:   gcc trampolining.o -o trampolining -no-pie

section .data
    fmt_result  db "Sum 1..%lld = %lld", 10, 0
    fmt_header  db "=== Trampolining Demo ===", 10, 0
    
    ; Thunk structure
    ; [0] = function pointer (NULL ถ้าเสร็จแล้ว)
    ; [8] = argument 1 (n)
    ; [16] = argument 2 (accumulator)
    ; [24] = result (ใช้เมื่อ done)
    
    THUNK_FN    equ 0
    THUNK_N     equ 8
    THUNK_ACC   equ 16
    THUNK_RES   equ 24
    THUNK_SIZE  equ 32

section .bss
    thunk_buf   resb THUNK_SIZE     ; thunk buffer

section .text
    global main
    extern printf

; sum_step(thunk_ptr)
; ทำ 1 step ของ recursive sum
; rdi = pointer to thunk
; คืนค่า: pointer to thunk (ถ้ายังมีต่อ) หรือ NULL (ถ้าเสร็จแล้ว)
sum_step:
    push rbp
    mov rbp, rsp
    push rbx
    sub rsp, 8
    
    mov rbx, rdi                    ; thunk pointer
    
    ; โหลด n และ accumulator
    mov rax, [rbx + THUNK_N]        ; n
    mov rcx, [rbx + THUNK_ACC]      ; acc
    
    ; Base case: n == 0
    test rax, rax
    jz .base_case
    
    ; n > 0: สร้าง thunk ถัดไป
    ; ใช้ thunk_buf เดิม (ประหยัด memory)
    dec rax                         ; n - 1
    add rcx, rax                    ; acc + n (ใช้ n ที่ลดแล้ว + 1)
    inc rax                         ; กลับมาเป็น n เดิมก่อนบวก
    add rcx, 0                      ; กันสับสน: acc += n
    
    ; แก้ไข: acc = acc + n (n เดิม), n = n-1
    mov rax, [rbx + THUNK_N]        ; โหลด n ใหม่
    add rcx, rax                    ; acc += n
    dec rax                         ; n--
    
    ; อัพเดท thunk
    mov [rbx + THUNK_N], rax        ; n = n-1
    mov [rbx + THUNK_ACC], rcx      ; acc = acc + old_n
    
    ; คืน thunk pointer (ยังมีต่อ)
    mov rax, rbx
    jmp .done

.base_case:
    ; เสร็จแล้ว: เก็บผลลัพธ์
    mov [rbx + THUNK_RES], rcx      ; result = accumulator
    mov qword [rbx + THUNK_FN], 0   ; fn = NULL (signal ว่าเสร็จ)
    xor eax, eax                    ; คืน NULL

.done:
    add rsp, 8
    pop rbx
    pop rbp
    ret

; trampoline(initial_thunk)
; วน loop เรียก thunk จนกว่าจะเสร็จ
; rdi = initial thunk
; คืนค่า: rax = ผลลัพธ์สุดท้าย
trampoline:
    push rbp
    mov rbp, rsp
    push rbx
    sub rsp, 8
    
    mov rbx, rdi                    ; current thunk

.loop:
    test rbx, rbx
    jz .done                        ; ถ้า NULL เสร็จแล้ว
    
    ; เรียก sum_step(current_thunk)
    mov rdi, rbx
    call sum_step
    
    ; ถ้า NULL คือเสร็จแล้ว
    test rax, rax
    jz .get_result
    
    mov rbx, rax                    ; อัพเดท current thunk
    jmp .loop

.get_result:
    ; โหลดผลลัพธ์จาก thunk
    mov rax, [rbx + THUNK_RES]
    jmp .done_with_result

.done:
    xor eax, eax

.done_with_result:
    add rsp, 8
    pop rbx
    pop rbp
    ret

; sum_iterative(n)
; คำนวณ 1 + 2 + ... + n แบบ iterative (ตรงๆ)
sum_iterative:
    push rbp
    mov rbp, rsp
    
    xor rax, rax                    ; result = 0
    test rdi, rdi
    jz .done
    
.loop:
    add rax, rdi
    dec rdi
    jnz .loop

.done:
    pop rbp
    ret

main:
    push rbp
    mov rbp, rsp
    push rbx
    sub rsp, 8
    
    lea rdi, [rel fmt_header]
    xor eax, eax
    call printf
    
    ; ทดสอบ trampoline sum(100)
    mov rbx, 100                    ; n = 100
    
    ; Initialize thunk
    lea rax, [rel thunk_buf]
    lea rcx, [rel sum_step]
    mov [rax + THUNK_FN], rcx       ; fn = sum_step
    mov [rax + THUNK_N], rbx        ; n = 100
    mov qword [rax + THUNK_ACC], 0  ; acc = 0
    
    ; เรียก trampoline
    lea rdi, [rel thunk_buf]
    call trampoline
    
    ; แสดงผล
    lea rdi, [rel fmt_result]
    mov rsi, rbx
    mov rdx, rax
    xor eax, eax
    call printf
    
    xor eax, eax
    add rsp, 8
    pop rbx
    pop rbp
    ret
```

---

## 15. การเปรียบเทียบ Performance {#performance-comparison}

### Benchmark: Recursive vs Iterative

```nasm
; ไฟล์: performance_bench.asm
; คำอธิบาย: เปรียบเทียบ performance ระหว่าง recursive และ iterative
; คอมไพล์: nasm -f elf64 performance_bench.asm -o performance_bench.o
; ลิงก์:   gcc performance_bench.o -o performance_bench -no-pie

section .data
    fmt_header  db "=== Performance Benchmark ===", 10, 0
    fmt_result  db "Method: %-20s  Result: %llu  Cycles: ~%llu", 10, 0
    fmt_divider db "--------------------------------------------", 10, 0
    
    str_recur   db "Recursive"
    str_tail    db "Tail Recursive"
    str_iter    db "Iterative"
    str_memo    db "Memoized"
    
    N_TEST      equ 35              ; ทดสอบด้วย fib(35)
    
    ; Memoization cache
    memo_cache  times 50 dq -1     ; -1 = ยังไม่ได้คำนวณ

section .text
    global main
    extern printf

; rdtsc_start: อ่าน Time Stamp Counter (นับ CPU cycles)
; คืนค่า: rax = TSC value
rdtsc_read:
    mfence                          ; Memory fence เพื่อความแม่นยำ
    rdtsc                           ; อ่าน TSC: EDX:EAX
    shl rdx, 32
    or rax, rdx                     ; รวมเป็น 64-bit value
    ret

; fib_recursive(n) - Fibonacci recursive ปกติ
fib_recursive:
    push rbp
    mov rbp, rsp
    push rbx
    sub rsp, 8
    
    mov rbx, rdi
    
    cmp rbx, 1
    jle .base
    
    lea rdi, [rbx-1]
    call fib_recursive
    mov [rbp-16], rax
    
    lea rdi, [rbx-2]
    call fib_recursive
    add rax, [rbp-16]
    jmp .done

.base:
    mov rax, rbx

.done:
    add rsp, 8
    pop rbx
    pop rbp
    ret

; fib_tail(n, acc1, acc2) - Fibonacci tail recursive
; rdi=n, rsi=acc1=fib(n-1), rdx=acc2=fib(n)
fib_tail:
    test rdi, rdi
    jz .done_zero
    cmp rdi, 1
    jz .done_one
    
    ; fib_tail(n-1, acc2, acc1+acc2)
    mov rcx, rsi
    add rcx, rdx                    ; acc1 + acc2
    mov rsi, rdx                    ; new acc1 = old acc2
    mov rdx, rcx                    ; new acc2 = acc1 + acc2
    dec rdi
    jmp fib_tail                    ; tail call (จะถูก optimize เป็น loop)

.done_zero:
    mov rax, rsi
    ret

.done_one:
    mov rax, rdx
    ret

; fib_tail_wrapper(n) - wrapper สำหรับ fib_tail
fib_tail_wrapper:
    push rbp
    mov rbp, rsp
    
    test rdi, rdi
    jz .zero
    cmp rdi, 1
    jz .one
    
    ; fib_tail(n, 0, 1)
    mov rsi, 0                      ; fib(0) = 0
    mov rdx, 1                      ; fib(1) = 1
    call fib_tail
    jmp .done

.zero:
    xor eax, eax
    jmp .done

.one:
    mov eax, 1

.done:
    pop rbp
    ret

; fib_iterative(n) - Fibonacci iterative
fib_iterative:
    push rbp
    mov rbp, rsp
    
    test rdi, rdi
    jz .zero
    cmp rdi, 1
    jz .one
    
    mov rcx, rdi                    ; counter = n
    xor rax, rax                    ; a = 0
    mov rdx, 1                      ; b = 1
    
.loop:
    cmp rcx, 1
    jle .done
    
    add rax, rdx                    ; temp = a + b
    xchg rax, rdx                   ; b = a + b, a = old_b
    xchg rax, rdx                   ; swap back: a = new, b = old
    ; ปรับ: a = b; b = a + b
    ; ใช้วิธีที่ถูกต้อง:
    sub rdx, rax
    xchg rax, rdx
    add rax, rdx
    dec rcx
    jmp .loop

.done:
    pop rbp
    ret

.zero:
    xor eax, eax
    pop rbp
    ret

.one:
    mov eax, 1
    pop rbp
    ret

; fib_iterative_correct(n) - เวอร์ชันที่แก้แล้ว
fib_iter_correct:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    sub rsp, 8
    
    test rdi, rdi
    jz .zero
    cmp rdi, 1
    jz .one
    
    xor rbx, rbx                    ; prev = 0 (fib(0))
    mov r12, 1                      ; curr = 1 (fib(1))
    mov rcx, 2                      ; i = 2
    
.loop:
    cmp rcx, rdi
    jg .done
    
    mov rax, rbx
    add rax, r12                    ; next = prev + curr
    mov rbx, r12                    ; prev = curr
    mov r12, rax                    ; curr = next
    inc rcx
    jmp .loop

.done:
    mov rax, r12
    jmp .return

.zero:
    xor eax, eax
    jmp .return

.one:
    mov eax, 1

.return:
    add rsp, 8
    pop r12
    pop rbx
    pop rbp
    ret

main:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    sub rsp, 16
    
    lea rdi, [rel fmt_header]
    xor eax, eax
    call printf
    
    lea rdi, [rel fmt_divider]
    xor eax, eax
    call printf
    
    ; === Test Recursive ===
    call rdtsc_read
    mov r12, rax                    ; start time
    
    mov rdi, N_TEST
    call fib_recursive
    mov r13, rax                    ; result
    
    call rdtsc_read
    sub rax, r12                    ; elapsed cycles
    mov r14, rax
    
    lea rdi, [rel fmt_result]
    lea rsi, [rel str_recur]
    mov rdx, r13
    mov rcx, r14
    xor eax, eax
    call printf
    
    ; === Test Tail Recursive ===
    call rdtsc_read
    mov r12, rax
    
    mov rdi, N_TEST
    call fib_tail_wrapper
    mov r13, rax
    
    call rdtsc_read
    sub rax, r12
    mov r14, rax
    
    lea rdi, [rel fmt_result]
    lea rsi, [rel str_tail]
    mov rdx, r13
    mov rcx, r14
    xor eax, eax
    call printf
    
    ; === Test Iterative ===
    call rdtsc_read
    mov r12, rax
    
    mov rdi, N_TEST
    call fib_iter_correct
    mov r13, rax
    
    call rdtsc_read
    sub rax, r12
    mov r14, rax
    
    lea rdi, [rel fmt_result]
    lea rsi, [rel str_iter]
    mov rdx, r13
    mov rcx, r14
    xor eax, eax
    call printf
    
    lea rdi, [rel fmt_divider]
    xor eax, eax
    call printf
    
    xor eax, eax
    add rsp, 16
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret
```

---

## สรุปแนวคิดสำคัญ

### 1. Stack Frame ใน Recursion
```
ทุก recursive call จะสร้าง stack frame ใหม่ที่ประกอบด้วย:
- Return address (8 bytes)
- Saved rbp (8 bytes)
- Saved registers (N * 8 bytes)
- Local variables
- Padding สำหรับ alignment
```

### 2. ความแตกต่างของ Recursion Types

| ประเภท | Stack Usage | Performance | ความอ่านง่าย |
|--------|-------------|-------------|--------------|
| Normal Recursive | O(n) | ช้า | ง่าย |
| Tail Recursive | O(1) หลัง optimize | เร็ว | ปานกลาง |
| Iterative | O(1) | เร็วที่สุด | ยาก |
| Memoized | O(n) cache | เร็วมาก (cache hit) | ปานกลาง |

### 3. เมื่อใดควรใช้ Recursion

- ปัญหาที่มีโครงสร้างแบบ tree หรือ graph
- Divide and conquer algorithms
- Backtracking problems
- เมื่อ code readability สำคัญกว่า performance

### 4. เมื่อใดควรหลีกเลี่ยง Recursion

- ปัญหาที่ต้องการ depth มากๆ (>100,000)
- Real-time systems ที่ latency สำคัญ
- Embedded systems ที่ stack จำกัด

---

## 16. แบบฝึกหัด {#exercises}

### ระดับพื้นฐาน

**แบบฝึกหัดที่ 1:** Power Function
```nasm
; เขียนฟังก์ชัน power(base, exp) ที่คำนวณ base^exp แบบ recursive
; power(2, 0) = 1
; power(2, n) = 2 * power(2, n-1)
; ทดสอบ: power(2, 10) = 1024

; เริ่มต้น...
section .data
    fmt_power   db "power(%d, %d) = %lld", 10, 0

section .text
    global main
    extern printf
    
; TODO: เขียน power function
power:
    ; ??? เขียนได้เลย
    ret

main:
    push rbp
    mov rbp, rsp
    
    ; ทดสอบ power(2, 10)
    mov rdi, 2
    mov rsi, 10
    call power
    
    ; แสดงผล
    lea rdi, [rel fmt_power]
    mov rsi, 2
    mov rdx, 10
    mov rcx, rax
    xor eax, eax
    call printf
    
    xor eax, eax
    pop rbp
    ret
```

**แบบฝึกหัดที่ 2:** Sum of Digits
```nasm
; เขียนฟังก์ชัน sum_digits(n) ที่หาผลรวมของหลักของตัวเลข
; sum_digits(123) = 1 + 2 + 3 = 6
; sum_digits(9999) = 9 + 9 + 9 + 9 = 36
; คำแนะนำ: ใช้ div instruction ในการแยกหลักสุดท้าย

; TODO: เขียน sum_digits function
sum_digits:
    ; hint: n % 10 = หลักสุดท้าย
    ;       n / 10 = ตัวเลขที่เหลือ
    ret
```

### ระดับกลาง

**แบบฝึกหัดที่ 3:** Palindrome Check
```nasm
; เขียนฟังก์ชัน is_palindrome(str, start, end)
; ตรวจสอบว่า string เป็น palindrome หรือไม่แบบ recursive
; is_palindrome("racecar", 0, 6) = 1 (true)
; is_palindrome("hello", 0, 4) = 0 (false)
;
; rdi = str pointer
; rsi = start index
; rdx = end index
; คืนค่า: rax = 1 ถ้าเป็น palindrome, 0 ถ้าไม่ใช่

is_palindrome:
    ; Base case: start >= end
    ; Check: str[start] == str[end]
    ; Recursive: is_palindrome(str, start+1, end-1)
    ret
```

**แบบฝึกหัดที่ 4:** GCD (Greatest Common Divisor)
```nasm
; เขียนฟังก์ชัน gcd(a, b) ด้วย Euclidean algorithm แบบ recursive
; gcd(48, 18) = gcd(18, 48%18) = gcd(18, 12) = gcd(12, 6) = gcd(6, 0) = 6
; gcd(a, 0) = a  (base case)
; gcd(a, b) = gcd(b, a % b)

gcd:
    ; rdi = a, rsi = b
    ; TODO: implement recursive GCD
    ret
```

### ระดับสูง

**แบบฝึกหัดที่ 5:** Quicksort
```nasm
; เขียน Quicksort แบบ recursive
; quicksort(arr, low, high)
; - เลือก pivot (ใช้ element สุดท้าย)
; - partition array รอบ pivot
; - quicksort ทั้งสองฝั่งแบบ recursive

section .data
    test_arr    dq 64, 34, 25, 12, 22, 11, 90

quicksort:
    ; rdi = arr, rsi = low, rdx = high
    ; TODO: implement quicksort
    ret

partition:
    ; rdi = arr, rsi = low, rdx = high
    ; คืนค่า: rax = partition index
    ; TODO: implement partition
    ret
```

**แบบฝึกหัดที่ 6:** Flood Fill Algorithm
```nasm
; เขียน Flood Fill แบบ recursive บน 2D array
; ใช้ในการวาดภาพ (paint bucket tool)
; flood_fill(grid, x, y, old_color, new_color)
; - ถ้า grid[x][y] != old_color return
; - ตั้ง grid[x][y] = new_color
; - เรียก flood_fill ทั้ง 4 ทิศทาง (up, down, left, right)

GRID_W  equ 10
GRID_H  equ 10

section .bss
    grid    resb GRID_W * GRID_H    ; grid ขนาด 10x10

flood_fill:
    ; rdi = grid, rsi = x, rdx = y, rcx = old_color, r8 = new_color
    ; TODO: implement flood fill
    ret
```

---

## สรุปคำสั่ง Compilation

```bash
# คอมไพล์ไฟล์ .asm เป็น object file
nasm -f elf64 <filename>.asm -o <filename>.o

# ลิงก์ด้วย gcc (เพื่อใช้ C library functions)
gcc <filename>.o -o <filename> -no-pie

# ลิงก์ด้วย ld (ถ้าไม่ใช้ C library)
ld <filename>.o -o <filename>

# Debug ด้วย gdb
gdb ./<filename>
(gdb) break main
(gdb) run
(gdb) info registers
(gdb) x/10xg $rsp    # ดู 10 values จาก stack (hex, 64-bit)

# ตรวจสอบ stack size ของ process
ulimit -s            # แสดง stack size limit (KB)
ulimit -s unlimited  # เพิ่ม stack size (สำหรับ recursion ลึกมาก)

# ดู assembly ที่ compiler สร้าง
gcc -O2 -S source.c -o output.s   # ดู assembly ที่ optimize แล้ว
objdump -d <executable>            # disassemble
```

---

## ข้อควรระวังสำคัญ

### 1. Stack Alignment
```nasm
; x86-64 ABI กำหนดว่า RSP ต้องหาร 16 ลงตัวก่อน call
; ถ้า push rbp แล้ว RSP จะลดลง 8 bytes
; ต้อง sub rsp, 8 อีก เพื่อให้ 16-byte aligned

push rbp
mov rbp, rsp
sub rsp, 8          ; alignment เพิ่มเติม (ถ้า frame size ไม่ใช่ 16 multiple)
```

### 2. Callee-saved Registers
```nasm
; Registers ที่ต้องเก็บและคืนค่า (callee-saved):
; rbx, rbp, r12, r13, r14, r15

; ถ้าใช้ rbx ใน recursive function:
push rbx            ; เก็บก่อน
; ... ใช้ rbx ...
pop rbx             ; คืนค่าก่อน ret
ret
```

### 3. การตรวจจับ Stack Overflow
```nasm
; วิธีตรวจสอบ stack ที่เหลือ:
mov rax, rsp
sub rax, 4096       ; เหลือน้อยกว่า 4KB?
js .overflow_risk   ; ถ้า rsp - 4096 < 0
```

### 4. Tail Call Optimization
```nasm
; การทำ Tail Call ด้วยมือ (Manual TCO):
; แทนที่จะใช้ call และ ret แยก:
;   call function
;   ret

; ใช้การปรับ RSP และ jump แทน:
;   mov rsp, rbp     ; ทำลาย frame ปัจจุบัน
;   pop rbp
;   jmp function     ; tail jump ไม่ใช่ call
```

---

## อ้างอิงและแหล่งข้อมูลเพิ่มเติม

- **Intel x86-64 Architecture Manual** - การใช้งาน stack และ calling convention
- **System V AMD64 ABI** - มาตรฐาน calling convention สำหรับ Linux 64-bit
- **NASM Documentation** - https://www.nasm.us/doc/
- **"Computer Organization and Architecture"** - William Stallings
- **"Introduction to Algorithms"** - CLRS (Cormen, Leiserson, Rivest, Stein)

---

*Part 025 - Recursive Procedures | Assembly Programming Course*
*สร้างโดย: Assembly Course Team | อัพเดทล่าสุด: 2024*
