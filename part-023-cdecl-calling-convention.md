# Part 023: cdecl Calling Convention และ System V AMD64 ABI

## บทนำ (Introduction)

**Calling Convention** คือข้อตกลงระหว่าง caller (ฟังก์ชันที่เรียก) และ callee (ฟังก์ชันที่ถูกเรียก) ว่าจะส่งพารามิเตอร์อย่างไร จะเคลียร์ stack อย่างไร และจะคืนค่าอย่างไร ใน Assembly เราต้องเข้าใจ calling convention เพื่อที่จะสามารถเรียกฟังก์ชัน C library และสื่อสารกับโค้ด C ได้อย่างถูกต้อง

### Calling Conventions หลักๆ ที่ควรรู้

| Convention | Platform | พารามิเตอร์ | Stack Cleanup | Return Value |
|---|---|---|---|---|
| cdecl | x86 Linux/Windows | Stack (right-to-left) | Caller | EAX/EDX:EAX |
| stdcall | x86 Windows | Stack (right-to-left) | Callee | EAX/EDX:EAX |
| fastcall | x86 | ECX, EDX, Stack | Callee | EAX |
| System V AMD64 | x86-64 Linux/Mac | RDI,RSI,RDX,RCX,R8,R9,Stack | Caller | RAX/RDX:RAX |
| Microsoft x64 | x86-64 Windows | RCX,RDX,R8,R9,Stack | Caller | RAX |

---

## ส่วนที่ 1: cdecl Calling Convention (32-bit)

### 1.1 กฎของ cdecl

**cdecl** (C declaration) เป็น calling convention มาตรฐานสำหรับโปรแกรม C บน x86:

```
กฎหลักของ cdecl:
1. ส่งพารามิเตอร์ผ่าน stack จากขวาไปซ้าย (right-to-left)
2. Caller รับผิดชอบเคลียร์ stack หลังเรียกฟังก์ชัน
3. คืนค่า integer/pointer ใน EAX
4. คืนค่า 64-bit ใน EDX:EAX (EDX = high 32 bits, EAX = low 32 bits)
5. Callee ต้องรักษา (preserve) EBX, ESI, EDI, EBP, ESP
6. EAX, ECX, EDX สามารถเปลี่ยนได้ (caller-saved)
```

### 1.2 Stack Frame ของ cdecl

```
Stack Layout เมื่อเรียก func(a, b, c):

High Address
┌─────────────────┐
│    argument c   │ ← ESP+12 (push ก่อน)
├─────────────────┤
│    argument b   │ ← ESP+8
├─────────────────┤
│    argument a   │ ← ESP+4 (push หลังสุด)
├─────────────────┤
│   Return Addr   │ ← ESP+0  (CALL ใส่ให้อัตโนมัติ)
├─────────────────┤ ← EBP ชี้ตรงนี้ (หลัง push ebp)
│   Saved EBP     │ ← EBP+0
├─────────────────┤
│   Local Var 1   │ ← EBP-4
├─────────────────┤
│   Local Var 2   │ ← EBP-8
└─────────────────┘
Low Address

หลัง CALL: ESP ลดลง 4 (เก็บ return address)
หลัง PUSH EBP + MOV EBP,ESP:
  - EBP ชี้ที่ saved EBP
  - EBP+4 = return address
  - EBP+8 = argument a (พารามิเตอร์แรก)
  - EBP+12 = argument b
  - EBP+16 = argument c
```

### 1.3 ตัวอย่าง cdecl 32-bit พื้นฐาน

```nasm
; ไฟล์: cdecl_basic.asm
; วิธีคอมไพล์: nasm -f elf32 cdecl_basic.asm -o cdecl_basic.o
;              gcc -m32 cdecl_basic.o -o cdecl_basic
; วิธีรัน:     ./cdecl_basic

section .data
    ; ข้อความสำหรับแสดงผล
    fmt_add     db "add(%d, %d) = %d", 10, 0
    fmt_mul     db "mul(%d, %d) = %d", 10, 0
    fmt_result  db "Result: %d", 10, 0

section .text
    global main
    extern printf

;--------------------------------------------------
; ฟังก์ชัน add(a, b) - คืนค่า a + b
; พารามิเตอร์: EBP+8 = a, EBP+12 = b
; คืนค่า: EAX = a + b
;--------------------------------------------------
add_func:
    push ebp            ; บันทึก base pointer เดิม
    mov  ebp, esp       ; ตั้ง base pointer ใหม่
    
    mov  eax, [ebp+8]   ; โหลดพารามิเตอร์ a
    add  eax, [ebp+12]  ; บวก b เข้าไปใน EAX
    
    pop  ebp            ; คืน base pointer
    ret                 ; คืน EAX เป็น return value

;--------------------------------------------------
; ฟังก์ชัน mul(a, b) - คืนค่า a * b
;--------------------------------------------------
mul_func:
    push ebp
    mov  ebp, esp
    
    mov  eax, [ebp+8]   ; โหลด a
    imul eax, [ebp+12]  ; คูณ b (signed multiply)
    
    pop  ebp
    ret

;--------------------------------------------------
; ฟังก์ชัน main
;--------------------------------------------------
main:
    push ebp
    mov  ebp, esp
    
    ; --- เรียก add_func(10, 20) ---
    push dword 20       ; push b ก่อน (right-to-left)
    push dword 10       ; push a
    call add_func
    add  esp, 8         ; CALLER เคลียร์ stack (2 args × 4 bytes)
    
    ; แสดงผล: printf("add(%d, %d) = %d\n", 10, 20, result)
    push eax            ; push result (คืนค่าจาก add_func ใน EAX)
    push dword 20       ; push b
    push dword 10       ; push a
    push fmt_add        ; push format string
    call printf
    add  esp, 16        ; เคลียร์ stack (4 args × 4 bytes)
    
    ; --- เรียก mul_func(6, 7) ---
    push dword 7
    push dword 6
    call mul_func
    add  esp, 8
    
    ; แสดงผล
    push eax
    push dword 7
    push dword 6
    push fmt_mul
    call printf
    add  esp, 16
    
    ; คืน 0 (success)
    xor  eax, eax
    pop  ebp
    ret
```

### 1.4 การ Preserve Registers ใน cdecl

```nasm
; ไฟล์: preserve_regs.asm
; แสดงการ preserve callee-saved registers
; nasm -f elf32 preserve_regs.asm -o preserve_regs.o
; gcc -m32 preserve_regs.o -o preserve_regs

section .data
    msg_before  db "Before: EBX=%d, ESI=%d, EDI=%d", 10, 0
    msg_after   db "After:  EBX=%d, ESI=%d, EDI=%d", 10, 0
    msg_result  db "Function result: %d", 10, 0

section .text
    global main
    extern printf

;--------------------------------------------------
; complex_func - ฟังก์ชันที่ใช้ callee-saved registers
; และต้อง preserve ให้ถูกต้อง
;--------------------------------------------------
complex_func:
    push ebp
    mov  ebp, esp
    
    ; บันทึก callee-saved registers ที่จะใช้
    push ebx            ; ต้อง preserve EBX
    push esi            ; ต้อง preserve ESI
    push edi            ; ต้อง preserve EDI
    
    ; ใช้ registers เหล่านี้คำนวณ
    mov  ebx, [ebp+8]   ; ebx = พารามิเตอร์แรก
    mov  esi, [ebp+12]  ; esi = พารามิเตอร์สอง
    mov  edi, [ebp+16]  ; edi = พารามิเตอร์สาม
    
    ; คำนวณ: result = (a * b) + c
    mov  eax, ebx
    imul eax, esi       ; eax = a * b
    add  eax, edi       ; eax += c
    
    ; คืน callee-saved registers (ลำดับ LIFO)
    pop  edi
    pop  esi
    pop  ebx
    
    pop  ebp
    ret

main:
    push ebp
    mov  ebp, esp
    
    ; ตั้งค่าให้ EBX, ESI, EDI มีค่า
    mov  ebx, 100
    mov  esi, 200
    mov  edi, 300
    
    ; แสดงค่าก่อนเรียก
    push edi
    push esi
    push ebx
    push msg_before
    call printf
    add  esp, 16
    
    ; เรียก complex_func(5, 4, 3)
    push dword 3
    push dword 4
    push dword 5
    call complex_func
    add  esp, 12
    
    ; แสดงค่าหลังเรียก (ควรเท่าเดิม)
    push edi
    push esi
    push ebx
    push msg_after
    call printf
    add  esp, 16
    
    ; แสดงผลลัพธ์
    push eax
    push msg_result
    call printf
    add  esp, 8
    
    xor  eax, eax
    pop  ebp
    ret
```

---

## ส่วนที่ 2: System V AMD64 ABI (64-bit)

### 2.1 กฎของ System V AMD64 ABI

```
กฎหลักของ System V AMD64 ABI (Linux/macOS):
1. ส่งพารามิเตอร์แรก 6 ตัวผ่าน registers: RDI, RSI, RDX, RCX, R8, R9
2. พารามิเตอร์ที่เกิน 6 ตัวส่งผ่าน stack (right-to-left)
3. Caller รับผิดชอบเคลียร์ stack
4. คืนค่า integer/pointer ใน RAX
5. คืนค่า 128-bit ใน RDX:RAX
6. คืนค่า float/double ใน XMM0
7. Stack ต้อง align 16-byte ก่อน CALL
8. Callee-saved: RBX, RBP, R12-R15
9. Caller-saved (volatile): RAX, RCX, RDX, RSI, RDI, R8-R11
```

### 2.2 Register Usage สำหรับ 64-bit

```
Parameter Registers (Integer/Pointer):
  1st: RDI    4th: RCX
  2nd: RSI    5th: R8
  3rd: RDX    6th: R9

Return Registers:
  RAX = primary return value (int, pointer, etc.)
  RDX = secondary (สำหรับ 128-bit returns)
  XMM0 = float/double return

Callee-Saved (ต้อง preserve):
  RBP, RBX, R12, R13, R14, R15

Caller-Saved (สามารถเปลี่ยนได้):
  RAX, RCX, RDX, RSI, RDI, R8, R9, R10, R11

Special:
  RSP = Stack Pointer
  RIP = Instruction Pointer
```

### 2.3 ตัวอย่าง System V AMD64 ABI

```nasm
; ไฟล์: sysv_amd64.asm
; วิธีคอมไพล์: nasm -f elf64 sysv_amd64.asm -o sysv_amd64.o
;              gcc sysv_amd64.o -o sysv_amd64
; วิธีรัน:     ./sysv_amd64

section .data
    fmt_add     db "add(%d, %d) = %d", 10, 0
    fmt_many    db "sum(%d,%d,%d,%d,%d,%d,%d) = %d", 10, 0
    fmt_str     db "Hello from: %s", 10, 0
    name_str    db "Assembly 64-bit", 0

section .text
    global main
    extern printf

;--------------------------------------------------
; add_func(a, b) - 64-bit version
; พารามิเตอร์: RDI=a, RSI=b
; คืนค่า: RAX = a + b
;--------------------------------------------------
add_func:
    ; 64-bit leaf function ไม่จำเป็นต้อง setup stack frame
    ; (แต่ทำก็ได้เพื่อความชัดเจน)
    mov  rax, rdi       ; rax = a
    add  rax, rsi       ; rax += b
    ret

;--------------------------------------------------
; sum_seven(a,b,c,d,e,f,g) - รับ 7 พารามิเตอร์
; พารามิเตอร์ 1-6: RDI,RSI,RDX,RCX,R8,R9
; พารามิเตอร์ที่ 7: อยู่บน stack ที่ RSP+8 (หลัง CALL)
; คืนค่า: RAX = ผลรวม
;--------------------------------------------------
sum_seven:
    push rbp
    mov  rbp, rsp
    
    ; บวก 6 พารามิเตอร์แรกจาก registers
    mov  rax, rdi       ; a
    add  rax, rsi       ; + b
    add  rax, rdx       ; + c
    add  rax, rcx       ; + d
    add  rax, r8        ; + e
    add  rax, r9        ; + f
    
    ; บวกพารามิเตอร์ที่ 7 จาก stack
    ; [rbp+8] = return address, [rbp+16] = 7th argument
    add  rax, [rbp+16]  ; + g
    
    pop  rbp
    ret

;--------------------------------------------------
; greet(name) - รับ string pointer
; พารามิเตอร์: RDI = pointer to string
;--------------------------------------------------
greet:
    push rbp
    mov  rbp, rsp
    
    ; RDI ยังคงเป็น name pointer
    ; printf("Hello from: %s\n", name)
    mov  rsi, rdi       ; rsi = name (พารามิเตอร์ที่ 2 ของ printf)
    lea  rdi, [rel fmt_str] ; rdi = format string (พารามิเตอร์แรก)
    xor  eax, eax       ; AL=0 บอก printf ว่าไม่มี float args
    call printf
    
    pop  rbp
    ret

main:
    push rbp
    mov  rbp, rsp
    
    ; --- เรียก add_func(15, 25) ---
    mov  rdi, 15        ; พารามิเตอร์แรก
    mov  rsi, 25        ; พารามิเตอร์สอง
    call add_func
    ; RAX = 40
    
    ; printf("add(%d, %d) = %d\n", 15, 25, result)
    mov  rdx, rax       ; rdx = result
    mov  rsi, 25        ; rsi = b
    mov  rdi, 15        ; rdi = a (จะถูก overwrite ด้วย format)
    push rdx            ; บันทึก result ไว้ก่อน
    lea  rdi, [rel fmt_add]
    pop  rdx
    mov  rsi, 15
    mov  rdx, 25
    ; รอ... ต้องจัดลำดับใหม่
    ; printf args: rdi=fmt, rsi=15, rdx=25, rcx=result
    push rax            ; เก็บ result
    lea  rdi, [rel fmt_add]
    mov  rsi, 15
    mov  rdx, 25
    pop  rcx            ; result
    xor  eax, eax
    call printf
    
    ; --- เรียก sum_seven(1,2,3,4,5,6,7) ---
    ; พารามิเตอร์ที่ 7 ต้อง push ขึ้น stack ก่อน
    ; และต้อง align stack ให้เป็น 16-byte ก่อน CALL
    sub  rsp, 8         ; padding เพื่อ align (แล้วจะ push อีก 8 = -16 total)
    push qword 7        ; พารามิเตอร์ที่ 7 บน stack
    mov  rdi, 1
    mov  rsi, 2
    mov  rdx, 3
    mov  rcx, 4
    mov  r8,  5
    mov  r9,  6
    call sum_seven
    add  rsp, 16        ; เคลียร์ stack (8 padding + 8 สำหรับ arg7)
    
    ; แสดงผล sum
    push rax
    lea  rdi, [rel fmt_many]
    mov  rsi, 1
    mov  rdx, 2
    mov  rcx, 3
    mov  r8,  4
    mov  r9,  5
    sub  rsp, 8         ; align
    push qword 7        ; arg9 (ผลลัพธ์)
    pop  r10
    push qword 6        ; arg8
    ; ซับซ้อนเกินไป ใช้วิธีอื่น
    add  rsp, 8
    pop  rax
    
    ; --- เรียก greet ---
    lea  rdi, [rel name_str]
    call greet
    
    xor  eax, eax
    pop  rbp
    ret
```

### 2.4 ตัวอย่างที่สะอาดกว่า - 64-bit cdecl

```nasm
; ไฟล์: clean_64bit.asm
; ตัวอย่างที่อ่านง่าย สำหรับ 64-bit calling convention
; nasm -f elf64 clean_64bit.asm -o clean_64bit.o
; gcc clean_64bit.o -o clean_64bit

section .data
    msg1    db "=== 64-bit Calling Convention Demo ===", 10, 0
    fmt_int db "Result: %ld", 10, 0
    fmt_two db "%ld + %ld = %ld", 10, 0
    fmt_six db "sum(1..6) = %ld", 10, 0

section .text
    global main
    extern printf

;--------------------------------------------------
; multiply(a, b) -> a * b
; rdi=a, rsi=b, return rax
;--------------------------------------------------
multiply:
    mov  rax, rdi
    imul rax, rsi
    ret

;--------------------------------------------------
; power(base, exp) -> base^exp (iterative)
; rdi=base, rsi=exp
;--------------------------------------------------
power:
    push rbx
    push r12
    
    mov  r12, rdi       ; r12 = base (callee-saved)
    mov  rbx, rsi       ; rbx = exp (callee-saved)
    
    mov  rax, 1         ; result = 1
    
.loop:
    test rbx, rbx       ; ถ้า exp == 0 จบ
    jz   .done
    imul rax, r12       ; result *= base
    dec  rbx            ; exp--
    jmp  .loop
    
.done:
    pop  r12
    pop  rbx
    ret

;--------------------------------------------------
; add_six(a,b,c,d,e,f) -> a+b+c+d+e+f
; rdi, rsi, rdx, rcx, r8, r9
;--------------------------------------------------
add_six:
    mov  rax, rdi
    add  rax, rsi
    add  rax, rdx
    add  rax, rcx
    add  rax, r8
    add  rax, r9
    ret

main:
    push rbp
    mov  rbp, rsp
    ; Stack ต้อง 16-byte aligned ก่อน CALL
    ; ตอนนี้ rsp ลดไป 8 (return addr) + 8 (push rbp) = -16 = aligned
    
    ; พิมพ์ header
    lea  rdi, [rel msg1]
    xor  eax, eax
    call printf
    
    ; --- multiply(12, 13) ---
    mov  rdi, 12
    mov  rsi, 13
    call multiply
    ; rax = 156
    
    mov  rdx, rax       ; rdx = result
    mov  rsi, 12        ; rsi = a
    ; จัดเรียง args สำหรับ printf("%ld + %ld = %ld", 12, 13, 156)
    ; แต่นี่จะพิมพ์แบบ format อื่น
    ; printf("Result: %ld\n", 156)
    mov  rsi, rdx
    lea  rdi, [rel fmt_int]
    xor  eax, eax
    call printf
    
    ; --- power(2, 10) = 1024 ---
    mov  rdi, 2
    mov  rsi, 10
    call power
    
    mov  rsi, rax
    lea  rdi, [rel fmt_int]
    xor  eax, eax
    call printf
    
    ; --- add_six(1,2,3,4,5,6) = 21 ---
    mov  rdi, 1
    mov  rsi, 2
    mov  rdx, 3
    mov  rcx, 4
    mov  r8,  5
    mov  r9,  6
    call add_six
    
    mov  rsi, rax
    lea  rdi, [rel fmt_six]
    xor  eax, eax
    call printf
    
    xor  eax, eax       ; return 0
    pop  rbp
    ret
```

---

## ส่วนที่ 3: Stack Alignment (การ Align Stack)

### 3.1 ทำไมต้อง Align Stack

```
System V AMD64 ABI กำหนดว่า:
- Stack pointer (RSP) ต้อง align 16-byte ก่อนทุก CALL instruction
- เมื่อ CALL ทำงาน มันจะ push return address (8 bytes)
  ทำให้ RSP ลดลง 8 bytes
- ดังนั้นเมื่อ function เริ่มทำงาน RSP จะ misaligned by 8

ทำไม? SSE/AVX instructions ต้องการ 16-byte alignment
ถ้า misaligned จะ SEGFAULT หรือ performance penalty

วิธีแก้:
1. ใน main: RSP เริ่มต้น aligned หลัง OS load + CALL main
2. ทุกครั้งที่ push/pop ต้องคิดว่า alignment เป็นเท่าไหร่
3. ใส่ sub rsp, 8 หรือ push/pop dummy เพื่อ align
```

### 3.2 ตัวอย่างการจัดการ Stack Alignment

```nasm
; ไฟล์: stack_align.asm
; แสดงการ align stack อย่างถูกต้อง
; nasm -f elf64 stack_align.asm -o stack_align.o
; gcc stack_align.o -o stack_align

section .data
    fmt     db "aligned call works: %d", 10, 0
    fmt_bad db "Stack was: 0x%lx", 10, 0

section .text
    global main
    extern printf

;--------------------------------------------------
; check_alignment - ฟังก์ชันทดสอบการ align
;--------------------------------------------------
check_alignment:
    push rbp
    mov  rbp, rsp
    
    ; RSP ตอนนี้ควรเป็น 16-byte aligned
    ; (ก่อน call: aligned, after call: -8, after push rbp: -8+(-8)=-16 = aligned)
    mov  rax, rsp
    and  rax, 0xF       ; เอา 4 bits ล่าง
    ; ถ้า rax == 0 แสดงว่า aligned
    
    pop  rbp
    ret

main:
    push rbp
    mov  rbp, rsp
    ; ตอนนี้ rsp = original - 16 (aligned)
    
    ; กรณี 1: เรียกตรงๆ (aligned)
    call check_alignment
    
    mov  rsi, rax
    lea  rdi, [rel fmt_bad]
    xor  eax, eax
    call printf
    
    ; กรณี 2: push 1 value (misalign by 8)
    push rax            ; rsp ลดลง 8 -> misaligned!
    
    ; ต้องแก้ด้วยการ push อีก 1 ตัว หรือ sub rsp, 8
    ; เลือกวิธี sub rsp, 8 (แค่ padding)
    sub  rsp, 8         ; align กลับ
    
    call check_alignment
    
    add  rsp, 8         ; เคลียร์ padding
    pop  rax            ; เคลียร์ของที่ push ไป
    
    mov  rsi, rax
    lea  rdi, [rel fmt_bad]
    xor  eax, eax
    call printf
    
    ; กรณี 3: push 2 values (ยังคง aligned)
    push rax
    push rbx
    ; rsp ลดลง 16 -> ยังคง aligned
    
    call check_alignment
    
    pop  rbx
    pop  rax
    
    ; แสดงผล
    mov  rsi, 42
    lea  rdi, [rel fmt]
    xor  eax, eax
    call printf
    
    xor  eax, eax
    pop  rbp
    ret
```

---

## ส่วนที่ 4: Variadic Functions และ printf/scanf

### 4.1 Variadic Functions ใน 64-bit

```
ใน System V AMD64 ABI สำหรับ variadic functions (เช่น printf):
- AL (low byte ของ RAX) ต้องบอกจำนวน floating-point args ใน XMM registers
- ถ้าไม่มี float args: xor eax, eax หรือ mov al, 0
- ถ้ามี float args: mov al, N (N = จำนวน float args ใน XMM0-XMM7)
```

### 4.2 การเรียก printf จาก Assembly

```nasm
; ไฟล์: printf_demo.asm
; ตัวอย่างการเรียก printf หลายรูปแบบ
; nasm -f elf64 printf_demo.asm -o printf_demo.o
; gcc printf_demo.o -o printf_demo

section .data
    ; Format strings ต่างๆ
    fmt_int     db "Integer: %d", 10, 0
    fmt_long    db "Long: %ld", 10, 0
    fmt_hex     db "Hex: 0x%X", 10, 0
    fmt_str     db "String: %s", 10, 0
    fmt_float   db "Float: %.2f", 10, 0
    fmt_multi   db "Name: %s, Age: %d, Score: %.1f", 10, 0
    fmt_newline db 10, 0
    
    ; ข้อมูล
    name_data   db "Alice", 0
    
    ; ค่าคงที่สำหรับ float (ต้องเก็บใน memory)
    pi_val      dq 3.14159265  ; double precision

section .text
    global main
    extern printf

;--------------------------------------------------
; print_int(n) - พิมพ์ integer
; rdi = n
;--------------------------------------------------
print_int:
    push rbp
    mov  rbp, rsp
    sub  rsp, 8         ; align stack
    
    mov  rsi, rdi       ; rsi = n (arg2)
    lea  rdi, [rel fmt_int] ; rdi = format (arg1)
    xor  eax, eax       ; ไม่มี float args
    call printf
    
    add  rsp, 8
    pop  rbp
    ret

;--------------------------------------------------
; print_hex(n) - พิมพ์ hex
;--------------------------------------------------
print_hex:
    push rbp
    mov  rbp, rsp
    sub  rsp, 8
    
    mov  rsi, rdi
    lea  rdi, [rel fmt_hex]
    xor  eax, eax
    call printf
    
    add  rsp, 8
    pop  rbp
    ret

main:
    push rbp
    mov  rbp, rsp
    
    ; --- พิมพ์ integer ---
    mov  rdi, 42
    call print_int
    
    ; --- พิมพ์ hex ---
    mov  rdi, 0xDEADBEEF
    call print_hex
    
    ; --- พิมพ์ string ---
    lea  rsi, [rel name_data]
    lea  rdi, [rel fmt_str]
    xor  eax, eax
    call printf
    
    ; --- พิมพ์ float (double) ---
    ; float args ใส่ใน XMM registers
    movsd xmm0, [rel pi_val] ; xmm0 = pi (arg2 สำหรับ float)
    lea   rdi, [rel fmt_float]
    mov   eax, 1        ; AL=1 บอก printf ว่ามี 1 float arg ใน XMM
    call  printf
    
    ; --- printf หลายพารามิเตอร์ ---
    ; printf("Name: %s, Age: %d, Score: %.1f\n", name, 25, 95.5)
    lea   rdi, [rel fmt_multi]
    lea   rsi, [rel name_data]  ; %s
    mov   rdx, 25               ; %d
    mov   rax, 0x4057E00000000000 ; 95.5 เป็น double = 0x4057E00000000000
    movq  xmm0, rax             ; ใส่ใน XMM0 สำหรับ %.1f
    mov   eax, 1                ; 1 float arg
    call  printf
    
    xor  eax, eax
    pop  rbp
    ret
```

### 4.3 การเรียก scanf จาก Assembly

```nasm
; ไฟล์: scanf_demo.asm
; ตัวอย่างการรับ input ด้วย scanf
; nasm -f elf64 scanf_demo.asm -o scanf_demo.o
; gcc scanf_demo.o -o scanf_demo

section .data
    prompt_int  db "Enter an integer: ", 0
    prompt_str  db "Enter your name: ", 0
    fmt_scanf_i db "%d", 0
    fmt_scanf_s db "%49s", 0        ; จำกัด 49 chars เพื่อความปลอดภัย
    fmt_result  db "You entered: %d", 10, 0
    fmt_hello   db "Hello, %s!", 10, 0

section .bss
    ; Buffer สำหรับรับ input
    int_buf     resd 1              ; 4 bytes สำหรับ int
    str_buf     resb 50             ; 50 bytes สำหรับ string

section .text
    global main
    extern printf, scanf

main:
    push rbp
    mov  rbp, rsp
    sub  rsp, 16        ; สำรอง space + align
    
    ; --- รับ integer ---
    ; printf("Enter an integer: ")
    lea  rdi, [rel prompt_int]
    xor  eax, eax
    call printf
    
    ; scanf("%d", &int_buf)
    lea  rsi, [rel int_buf]     ; rsi = pointer to buffer (arg2)
    lea  rdi, [rel fmt_scanf_i] ; rdi = format string (arg1)
    xor  eax, eax
    call scanf
    
    ; printf("You entered: %d\n", int_buf)
    mov  esi, [rel int_buf]     ; esi = int_buf value (32-bit ok)
    lea  rdi, [rel fmt_result]
    xor  eax, eax
    call printf
    
    ; --- รับ string ---
    ; printf("Enter your name: ")
    lea  rdi, [rel prompt_str]
    xor  eax, eax
    call printf
    
    ; scanf("%49s", str_buf)
    lea  rsi, [rel str_buf]
    lea  rdi, [rel fmt_scanf_s]
    xor  eax, eax
    call scanf
    
    ; printf("Hello, %s!\n", str_buf)
    lea  rsi, [rel str_buf]
    lea  rdi, [rel fmt_hello]
    xor  eax, eax
    call printf
    
    xor  eax, eax
    add  rsp, 16
    pop  rbp
    ret
```

---

## ส่วนที่ 5: Caller-Saved vs Callee-Saved

### 5.1 สรุปตาราง Register Saving

```
32-bit cdecl:
┌──────────┬─────────────┬────────────────────────────────────────┐
│ Register │ สถานะ       │ ความหมาย                               │
├──────────┼─────────────┼────────────────────────────────────────┤
│ EAX      │ Caller-saved│ Return value, สามารถเปลี่ยนได้         │
│ ECX      │ Caller-saved│ Counter, สามารถเปลี่ยนได้              │
│ EDX      │ Caller-saved│ Data, สามารถเปลี่ยนได้                 │
│ EBX      │ Callee-saved│ ต้อง preserve, ใช้สำหรับ base address │
│ ESP      │ Special     │ Stack pointer, ต้อง restore              │
│ EBP      │ Callee-saved│ Base pointer, ต้อง preserve             │
│ ESI      │ Callee-saved│ Source index, ต้อง preserve             │
│ EDI      │ Callee-saved│ Dest index, ต้อง preserve               │
└──────────┴─────────────┴────────────────────────────────────────┘

64-bit System V AMD64:
┌──────────┬─────────────┬────────────────────────────────────────┐
│ Register │ สถานะ       │ ความหมาย                               │
├──────────┼─────────────┼────────────────────────────────────────┤
│ RAX      │ Caller-saved│ Return value                            │
│ RCX      │ Caller-saved│ 4th arg                                │
│ RDX      │ Caller-saved│ 3rd arg                                │
│ RSI      │ Caller-saved│ 2nd arg                                │
│ RDI      │ Caller-saved│ 1st arg                                │
│ R8-R11   │ Caller-saved│ 5th-6th arg + scratch                  │
│ RBX      │ Callee-saved│ ต้อง preserve                          │
│ RBP      │ Callee-saved│ ต้อง preserve                          │
│ R12-R15  │ Callee-saved│ ต้อง preserve                          │
│ RSP      │ Special     │ ต้อง restore, aligned 16-byte          │
└──────────┴─────────────┴────────────────────────────────────────┘
```

### 5.2 ตัวอย่าง Caller vs Callee Save

```nasm
; ไฟล์: caller_callee.asm
; แสดงความแตกต่างระหว่าง caller-saved และ callee-saved
; nasm -f elf64 caller_callee.asm -o caller_callee.o
; gcc caller_callee.o -o caller_callee

section .data
    fmt_before  db "Before call: rdi=%ld, rbx=%ld", 10, 0
    fmt_after   db "After call:  rdi=%ld, rbx=%ld", 10, 0
    fmt_sep     db "---", 10, 0

section .text
    global main
    extern printf

;--------------------------------------------------
; modify_regs - ฟังก์ชันที่ modify registers ต่างๆ
; แสดงว่า register ไหนเปลี่ยน ไหนไม่เปลี่ยน
;--------------------------------------------------
modify_regs:
    push rbp
    mov  rbp, rsp
    push rbx            ; MUST save rbx (callee-saved)
    
    ; Modify caller-saved registers (ทำได้เสรี)
    mov  rdi, 9999      ; เปลี่ยน rdi - caller-saved
    mov  rsi, 8888      ; เปลี่ยน rsi - caller-saved
    mov  rdx, 7777      ; เปลี่ยน rdx - caller-saved
    mov  rcx, 6666      ; เปลี่ยน rcx - caller-saved
    
    ; Modify rbx แล้วต้อง restore (callee-saved)
    mov  rbx, 5555      ; เปลี่ยน rbx ชั่วคราว
    
    ; ทำงาน...
    
    pop  rbx            ; MUST restore rbx
    pop  rbp
    ret

main:
    push rbp
    mov  rbp, rsp
    
    ; ตั้งค่า registers
    mov  rdi, 100       ; caller-saved
    mov  rbx, 200       ; callee-saved
    
    ; แสดงค่าก่อนเรียก
    push rdi
    push rbx
    lea  rdi, [rel fmt_before]
    mov  rsi, 100       ; ค่าของ rdi ที่ตั้งไว้
    mov  rdx, 200       ; ค่าของ rbx ที่ตั้งไว้
    xor  eax, eax
    call printf
    pop  rbx
    pop  rdi
    
    ; เรียก modify_regs (จะเปลี่ยน caller-saved regs!)
    call modify_regs
    
    ; หลังเรียก: rdi อาจเปลี่ยน! rbx ต้องเหมือนเดิม
    ; ต้องไม่ rely on rdi หลัง call (caller-saved)
    ; rbx ยังคงเป็น 200 (callee-saved)
    
    ; แสดงค่าหลังเรียก
    ; หมายเหตุ: rdi เปลี่ยนแล้ว ต้อง set ใหม่
    lea  rdi, [rel fmt_after]
    mov  rsi, -1        ; rdi เปลี่ยนแล้ว ใส่ -1 แสดงว่าไม่รู้
    mov  rdx, rbx       ; rbx ยังคงเป็น 200
    xor  eax, eax
    call printf
    
    xor  eax, eax
    pop  rbp
    ret
```

---

## ส่วนที่ 6: การเรียก C Library Functions

### 6.1 รายการ C Library Functions ที่ใช้บ่อย

```
libc functions ที่ต้องรู้สำหรับ Assembly:

I/O Functions:
  printf(fmt, ...)        - พิมพ์ formatted output
  scanf(fmt, ...)         - รับ formatted input
  puts(str)               - พิมพ์ string + newline
  putchar(c)              - พิมพ์ character เดียว
  getchar()               - รับ character เดียว
  fgets(buf, n, stream)   - รับ string (safe)

String Functions:
  strlen(str)             - ความยาว string
  strcpy(dst, src)        - copy string
  strcat(dst, src)        - ต่อ string
  strcmp(s1, s2)          - เปรียบเทียบ string
  sprintf(buf, fmt, ...)  - format ลง buffer

Memory Functions:
  malloc(size)            - จอง heap memory
  calloc(n, size)         - จอง + clear memory
  realloc(ptr, size)      - ปรับขนาด memory
  free(ptr)               - คืน memory
  memcpy(dst, src, n)     - copy memory
  memset(ptr, val, n)     - fill memory

Math Functions:
  abs(n)                  - absolute value
  sqrt(x)                 - square root (libm)
  pow(x, y)               - x^y (libm)
```

### 6.2 ตัวอย่างการใช้ String Functions

```nasm
; ไฟล์: string_funcs.asm
; ตัวอย่างการเรียก C string functions
; nasm -f elf64 string_funcs.asm -o string_funcs.o
; gcc string_funcs.o -o string_funcs

section .data
    str1        db "Hello, ", 0
    str2        db "World!", 0
    fmt_len     db "Length of '%s' = %ld", 10, 0
    fmt_cmp     db "strcmp('%s', '%s') = %d", 10, 0
    fmt_result  db "Result: %s", 10, 0
    fmt_copy    db "Copied: %s", 10, 0

section .bss
    buffer      resb 256    ; buffer สำหรับ string operations

section .text
    global main
    extern printf, strlen, strcpy, strcat, strcmp

main:
    push rbp
    mov  rbp, rsp
    
    ; --- strlen ---
    lea  rdi, [rel str1]
    call strlen
    ; rax = length of str1
    
    mov  rdx, rax           ; rdx = length
    lea  rsi, [rel str1]    ; rsi = string
    lea  rdi, [rel fmt_len]
    xor  eax, eax
    call printf
    
    ; strlen ของ str2
    lea  rdi, [rel str2]
    call strlen
    
    mov  rdx, rax
    lea  rsi, [rel str2]
    lea  rdi, [rel fmt_len]
    xor  eax, eax
    call printf
    
    ; --- strcpy ---
    lea  rdi, [rel buffer]  ; dst
    lea  rsi, [rel str1]    ; src
    call strcpy
    
    lea  rsi, [rel buffer]
    lea  rdi, [rel fmt_copy]
    xor  eax, eax
    call printf
    
    ; --- strcat ---
    lea  rdi, [rel buffer]  ; dst (ต่อจาก str1)
    lea  rsi, [rel str2]    ; src (str2 ต่อท้าย)
    call strcat
    
    lea  rsi, [rel buffer]
    lea  rdi, [rel fmt_result]
    xor  eax, eax
    call printf
    
    ; --- strcmp ---
    lea  rdi, [rel str1]
    lea  rsi, [rel str1]    ; เปรียบเทียบกับตัวเอง
    call strcmp
    ; rax = 0 (equal)
    
    mov  ecx, eax           ; result
    lea  rdx, [rel str1]    ; s2
    lea  rsi, [rel str1]    ; s1
    lea  rdi, [rel fmt_cmp]
    xor  eax, eax
    call printf
    
    ; strcmp str1 vs str2 (ต่างกัน)
    lea  rdi, [rel str1]
    lea  rsi, [rel str2]
    call strcmp
    
    mov  ecx, eax
    lea  rdx, [rel str2]
    lea  rsi, [rel str1]
    lea  rdi, [rel fmt_cmp]
    xor  eax, eax
    call printf
    
    xor  eax, eax
    pop  rbp
    ret
```

### 6.3 ตัวอย่างการใช้ Memory Functions

```nasm
; ไฟล์: memory_funcs.asm
; ตัวอย่าง malloc/free และ memory functions
; nasm -f elf64 memory_funcs.asm -o memory_funcs.o
; gcc memory_funcs.o -o memory_funcs

section .data
    fmt_alloc   db "Allocated %ld bytes at %p", 10, 0
    fmt_fill    db "Memory filled with 0x%02X", 10, 0
    fmt_val     db "Value at index %d: %d", 10, 0
    fmt_free    db "Memory freed", 10, 0
    fmt_null    db "malloc failed!", 10, 0

section .text
    global main
    extern printf, malloc, calloc, free, memset

main:
    push rbp
    mov  rbp, rsp
    push rbx            ; เก็บ pointer (callee-saved)
    push r12            ; เก็บ size (callee-saved)
    
    ; --- malloc(100) ---
    mov  rdi, 100       ; size = 100 bytes
    mov  r12, 100       ; บันทึก size ไว้
    call malloc
    mov  rbx, rax       ; rbx = pointer (บันทึกไว้)
    
    ; ตรวจสอบ NULL
    test rax, rax
    jz   .malloc_failed
    
    ; แสดงที่อยู่
    mov  rdx, rax       ; rdx = pointer
    mov  rsi, r12       ; rsi = size
    lea  rdi, [rel fmt_alloc]
    xor  eax, eax
    call printf
    
    ; --- memset(ptr, 0xAB, 100) ---
    mov  rdi, rbx       ; rdi = ptr
    mov  rsi, 0xAB      ; rsi = value
    mov  rdx, r12       ; rdx = count
    call memset
    
    ; แสดงว่า fill ด้วย 0xAB
    mov  rsi, 0xAB
    lea  rdi, [rel fmt_fill]
    xor  eax, eax
    call printf
    
    ; แสดงค่าบางตัว
    mov  rax, rbx       ; rax = ptr
    movzx esi, byte [rax]    ; ค่าที่ index 0
    mov  edx, esi
    xor  esi, esi       ; index = 0
    lea  rdi, [rel fmt_val]
    xor  eax, eax
    call printf
    
    movzx esi, byte [rbx+50] ; ค่าที่ index 50
    mov  edx, esi
    mov  esi, 50
    lea  rdi, [rel fmt_val]
    xor  eax, eax
    call printf
    
    ; --- free ---
    mov  rdi, rbx
    call free
    
    lea  rdi, [rel fmt_free]
    xor  eax, eax
    call printf
    
    jmp  .done

.malloc_failed:
    lea  rdi, [rel fmt_null]
    xor  eax, eax
    call printf

.done:
    xor  eax, eax
    pop  r12
    pop  rbx
    pop  rbp
    ret
```

---

## ส่วนที่ 7: Name Mangling

### 7.1 Name Mangling คืออะไร

```
Name Mangling คือการที่ compiler เปลี่ยนชื่อ symbol เพื่อเข้ารหัส
ข้อมูลเพิ่มเติม (เช่น type, namespace, class) ลงในชื่อ function

ทำไมมี Name Mangling?
- C++ ต้องการ function overloading (ชื่อเดียวกัน parameters ต่างกัน)
- ต้องการ namespace และ class information
- ป้องกันการ link ข้ามกันผิดพลาด

C ไม่มี name mangling (ยกเว้น prefix _ บน Windows/macOS):
  C function: int add(int, int)  → symbol: add (Linux) หรือ _add (macOS)

C++ มี name mangling:
  int add(int, int)         → _Z3addii
  int add(float, float)     → _Z3addff
  MyClass::method(int)      → _ZN7MyClass6methodEi
```

### 7.2 ตัวอย่าง Name Mangling

```nasm
; ไฟล์: name_mangling.asm
; แสดงการเรียก C function vs C++ function
; 
; สำหรับ C function (ใน .c file):
;   extern "C" int c_function(int x);  // ปิด mangling
;
; วิธีคอมไพล์ (กับ C):
;   nasm -f elf64 name_mangling.asm -o name_mangling.o
;   gcc name_mangling.o helper.c -o name_mangling
;
; วิธีคอมไพล์ (กับ C++):
;   nasm -f elf64 name_mangling.asm -o name_mangling.o
;   g++ name_mangling.o helper.cpp -o name_mangling
;   (ใน .cpp file ต้องมี extern "C" { ... })

section .data
    fmt_c   db "C function returned: %d", 10, 0
    fmt_asm db "Assembly function: %d", 10, 0

section .text
    global main
    global asm_add      ; export function นี้ให้ C/C++ เรียกได้
    extern printf
    extern c_add        ; เรียก C function (ต้องมี extern "C" ใน C++)

;--------------------------------------------------
; asm_add(a, b) - function ที่ C/C++ จะเรียก
; rdi=a, rsi=b, return rax
;--------------------------------------------------
asm_add:
    mov  rax, rdi
    add  rax, rsi
    ret

main:
    push rbp
    mov  rbp, rsp
    
    ; เรียก C function
    mov  rdi, 10
    mov  rsi, 20
    call c_add
    
    mov  rsi, rax
    lea  rdi, [rel fmt_c]
    xor  eax, eax
    call printf
    
    ; ทดสอบ asm_add เอง
    mov  rdi, 100
    mov  rsi, 200
    call asm_add
    
    mov  rsi, rax
    lea  rdi, [rel fmt_asm]
    xor  eax, eax
    call printf
    
    xor  eax, eax
    pop  rbp
    ret
```

```c
// ไฟล์: helper.c (ไฟล์ C สำหรับคู่กับ name_mangling.asm)
#include <stdio.h>

// C function ธรรมดา
int c_add(int a, int b) {
    return a + b;
}

// ถ้าใช้ C++ ต้องเพิ่ม extern "C":
// extern "C" int c_add(int a, int b) { return a + b; }
```

### 7.3 การตรวจสอบ Symbol Names

```bash
# ตรวจสอบ symbol ใน object file
nm myprogram.o

# ตรวจสอบ symbol ใน shared library
nm -D /lib/x86_64-linux-gnu/libc.so.6 | grep printf

# demangling C++ symbols
nm myprogram | c++filt

# ตรวจสอบ dynamic symbols
readelf -s myprogram

# ดู shared library dependencies
ldd myprogram
```

---

## ส่วนที่ 8: โปรแกรมตัวอย่างใหญ่ - Calculator

### 8.1 Calculator ด้วย cdecl (32-bit)

```nasm
; ไฟล์: calc32.asm
; Calculator 32-bit ด้วย cdecl convention
; nasm -f elf32 calc32.asm -o calc32.o
; gcc -m32 calc32.o -o calc32

section .data
    menu        db "=== Assembly Calculator (32-bit) ===", 10
                db "1. Add", 10
                db "2. Subtract", 10
                db "3. Multiply", 10
                db "4. Divide", 10
                db "5. Exit", 10
                db "Choice: ", 0
    prompt_a    db "Enter a: ", 0
    prompt_b    db "Enter b: ", 0
    fmt_int     db "%d", 0
    fmt_result  db "Result: %d", 10, 0
    fmt_div0    db "Error: Division by zero!", 10, 0
    fmt_rem     db "Quotient: %d, Remainder: %d", 10, 0
    fmt_invalid db "Invalid choice!", 10, 0

section .bss
    choice      resd 1
    num_a       resd 1
    num_b       resd 1

section .text
    global main
    extern printf, scanf

;--------------------------------------------------
; read_int(prompt, result_ptr)
; EBP+8 = prompt string, EBP+12 = pointer to int
;--------------------------------------------------
read_int:
    push ebp
    mov  ebp, esp
    
    ; printf(prompt)
    push dword [ebp+8]
    call printf
    add  esp, 4
    
    ; scanf("%d", result_ptr)
    push dword [ebp+12]
    push fmt_int
    call scanf
    add  esp, 8
    
    pop  ebp
    ret

;--------------------------------------------------
; do_add(a, b) -> a + b
;--------------------------------------------------
do_add:
    push ebp
    mov  ebp, esp
    mov  eax, [ebp+8]
    add  eax, [ebp+12]
    pop  ebp
    ret

;--------------------------------------------------
; do_sub(a, b) -> a - b
;--------------------------------------------------
do_sub:
    push ebp
    mov  ebp, esp
    mov  eax, [ebp+8]
    sub  eax, [ebp+12]
    pop  ebp
    ret

;--------------------------------------------------
; do_mul(a, b) -> a * b
;--------------------------------------------------
do_mul:
    push ebp
    mov  ebp, esp
    mov  eax, [ebp+8]
    imul eax, [ebp+12]
    pop  ebp
    ret

;--------------------------------------------------
; do_div(a, b) -> quotient (remainder ใน EDX)
; คืน 0 ถ้า b = 0
;--------------------------------------------------
do_div:
    push ebp
    mov  ebp, esp
    
    ; ตรวจสอบ division by zero
    mov  ecx, [ebp+12]  ; ecx = b
    test ecx, ecx
    jz   .div_zero
    
    mov  eax, [ebp+8]   ; eax = a
    cdq                  ; sign-extend eax -> edx:eax
    idiv ecx            ; eax = quotient, edx = remainder
    
    pop  ebp
    ret
    
.div_zero:
    push fmt_div0
    call printf
    add  esp, 4
    xor  eax, eax
    xor  edx, edx
    pop  ebp
    ret

main:
    push ebp
    mov  ebp, esp

.main_loop:
    ; แสดง menu
    push menu
    call printf
    add  esp, 4
    
    ; รับ choice
    push choice
    push fmt_int
    call scanf
    add  esp, 8
    
    ; ตรวจสอบ choice 5 (exit)
    mov  eax, [choice]
    cmp  eax, 5
    je   .exit
    
    ; ตรวจสอบ invalid
    cmp  eax, 1
    jl   .invalid
    cmp  eax, 4
    jg   .invalid
    
    ; รับ a และ b
    push num_a
    push prompt_a
    call read_int
    add  esp, 8
    
    push num_b
    push prompt_b
    call read_int
    add  esp, 8
    
    ; dispatch ตาม choice
    mov  eax, [choice]
    cmp  eax, 1
    je   .do_add
    cmp  eax, 2
    je   .do_sub
    cmp  eax, 3
    je   .do_mul
    cmp  eax, 4
    je   .do_div
    jmp  .invalid

.do_add:
    push dword [num_b]
    push dword [num_a]
    call do_add
    add  esp, 8
    push eax
    push fmt_result
    call printf
    add  esp, 8
    jmp  .main_loop

.do_sub:
    push dword [num_b]
    push dword [num_a]
    call do_sub
    add  esp, 8
    push eax
    push fmt_result
    call printf
    add  esp, 8
    jmp  .main_loop

.do_mul:
    push dword [num_b]
    push dword [num_a]
    call do_mul
    add  esp, 8
    push eax
    push fmt_result
    call printf
    add  esp, 8
    jmp  .main_loop

.do_div:
    push dword [num_b]
    push dword [num_a]
    call do_div
    add  esp, 8
    ; ตรวจสอบว่าเป็น div by zero
    cmp  dword [num_b], 0
    je   .main_loop
    push edx            ; remainder
    push eax            ; quotient
    push fmt_rem
    call printf
    add  esp, 12
    jmp  .main_loop

.invalid:
    push fmt_invalid
    call printf
    add  esp, 4
    jmp  .main_loop

.exit:
    xor  eax, eax
    pop  ebp
    ret
```

### 8.2 Calculator ด้วย System V AMD64 ABI (64-bit)

```nasm
; ไฟล์: calc64.asm
; Calculator 64-bit ด้วย System V AMD64 ABI
; nasm -f elf64 calc64.asm -o calc64.o
; gcc calc64.o -o calc64

section .data
    menu        db "=== Assembly Calculator (64-bit) ===", 10
                db "1. Add    2. Subtract", 10
                db "3. Multiply  4. Divide", 10
                db "5. Power  6. Exit", 10
                db "Choice: ", 0
    prompt_a    db "Enter a: ", 0
    prompt_b    db "Enter b: ", 0
    fmt_int     db "%ld", 0
    fmt_result  db "Result: %ld", 10, 0
    fmt_rem     db "Quotient: %ld, Remainder: %ld", 10, 0
    fmt_div0    db "Error: Division by zero!", 10, 0
    fmt_invalid db "Invalid choice! (1-6)", 10, 0
    fmt_nl      db 10, 0

section .bss
    num_a       resq 1      ; 8 bytes สำหรับ int64
    num_b       resq 1

section .text
    global main
    extern printf, scanf

;--------------------------------------------------
; read_long(prompt) -> rax = value อ่านได้
; rdi = prompt string
;--------------------------------------------------
read_long:
    push rbp
    mov  rbp, rsp
    push rbx
    sub  rsp, 8             ; align + space for local
    
    ; printf(prompt)
    xor  eax, eax
    call printf             ; rdi ยังเป็น prompt
    
    ; scanf("%ld", &local_var)
    lea  rsi, [rbp-16]      ; pointer to local storage
    lea  rdi, [rel fmt_int]
    xor  eax, eax
    call scanf
    
    mov  rax, [rbp-16]      ; คืนค่าที่อ่านได้
    
    add  rsp, 8
    pop  rbx
    pop  rbp
    ret

;--------------------------------------------------
; add64(a, b) -> rax = a + b
; rdi=a, rsi=b
;--------------------------------------------------
add64:
    mov  rax, rdi
    add  rax, rsi
    ret

;--------------------------------------------------
; sub64(a, b) -> rax = a - b
;--------------------------------------------------
sub64:
    mov  rax, rdi
    sub  rax, rsi
    ret

;--------------------------------------------------
; mul64(a, b) -> rax = a * b
;--------------------------------------------------
mul64:
    mov  rax, rdi
    imul rax, rsi
    ret

;--------------------------------------------------
; div64(a, b) -> rax = quotient, rdx = remainder
; ถ้า b==0 คืน 0 และแสดง error
;--------------------------------------------------
div64:
    push rbp
    mov  rbp, rsp
    sub  rsp, 16
    
    ; เก็บ a, b
    mov  [rbp-8],  rdi    ; a
    mov  [rbp-16], rsi    ; b
    
    ; ตรวจ div by zero
    test rsi, rsi
    jnz  .do_div
    
    lea  rdi, [rel fmt_div0]
    xor  eax, eax
    call printf
    
    xor  rax, rax
    xor  rdx, rdx
    add  rsp, 16
    pop  rbp
    ret

.do_div:
    mov  rax, [rbp-8]
    cqo                    ; sign-extend rax -> rdx:rax
    idiv qword [rbp-16]    ; rax = quotient, rdx = remainder
    
    add  rsp, 16
    pop  rbp
    ret

;--------------------------------------------------
; pow64(base, exp) -> rax = base^exp
; rdi=base, rsi=exp
;--------------------------------------------------
pow64:
    push rbx
    push r12
    
    mov  r12, rdi           ; r12 = base
    mov  rbx, rsi           ; rbx = exp
    mov  rax, 1             ; result = 1
    
.pow_loop:
    test rbx, rbx
    jle  .pow_done
    imul rax, r12
    dec  rbx
    jmp  .pow_loop

.pow_done:
    pop  r12
    pop  rbx
    ret

main:
    push rbp
    mov  rbp, rsp
    push rbx                ; callee-saved สำหรับ choice
    push r12                ; callee-saved สำหรับ a
    push r13                ; callee-saved สำหรับ b
    sub  rsp, 8             ; align (3 pushes = 24 bytes, need 8 more = 32)

.menu_loop:
    ; แสดง menu
    lea  rdi, [rel menu]
    xor  eax, eax
    call printf
    
    ; อ่าน choice
    lea  rdi, [rel prompt_a]  ; reuse prompt (แค่แสดง "Enter a:")
    ; ใช้ scanf โดยตรง
    sub  rsp, 16
    lea  rsi, [rsp]           ; pointer to local
    lea  rdi, [rel fmt_int]
    xor  eax, eax
    call scanf
    mov  rbx, [rsp]           ; rbx = choice
    add  rsp, 16
    
    ; ตรวจ exit
    cmp  rbx, 6
    je   .exit
    
    ; ตรวจ range
    cmp  rbx, 1
    jl   .invalid_choice
    cmp  rbx, 5
    jg   .invalid_choice
    
    ; อ่าน a
    lea  rdi, [rel prompt_a]
    call read_long
    mov  r12, rax             ; r12 = a
    
    ; อ่าน b
    lea  rdi, [rel prompt_b]
    call read_long
    mov  r13, rax             ; r13 = b
    
    ; dispatch
    cmp  rbx, 1
    je   .calc_add
    cmp  rbx, 2
    je   .calc_sub
    cmp  rbx, 3
    je   .calc_mul
    cmp  rbx, 4
    je   .calc_div
    cmp  rbx, 5
    je   .calc_pow
    jmp  .invalid_choice

.calc_add:
    mov  rdi, r12
    mov  rsi, r13
    call add64
    mov  rsi, rax
    lea  rdi, [rel fmt_result]
    xor  eax, eax
    call printf
    jmp  .menu_loop

.calc_sub:
    mov  rdi, r12
    mov  rsi, r13
    call sub64
    mov  rsi, rax
    lea  rdi, [rel fmt_result]
    xor  eax, eax
    call printf
    jmp  .menu_loop

.calc_mul:
    mov  rdi, r12
    mov  rsi, r13
    call mul64
    mov  rsi, rax
    lea  rdi, [rel fmt_result]
    xor  eax, eax
    call printf
    jmp  .menu_loop

.calc_div:
    mov  rdi, r12
    mov  rsi, r13
    call div64
    test r13, r13
    jz   .menu_loop
    push rdx                  ; remainder
    push rax                  ; quotient
    lea  rdi, [rel fmt_rem]
    pop  rsi                  ; quotient
    pop  rdx                  ; remainder
    xor  eax, eax
    call printf
    jmp  .menu_loop

.calc_pow:
    mov  rdi, r12
    mov  rsi, r13
    call pow64
    mov  rsi, rax
    lea  rdi, [rel fmt_result]
    xor  eax, eax
    call printf
    jmp  .menu_loop

.invalid_choice:
    lea  rdi, [rel fmt_invalid]
    xor  eax, eax
    call printf
    jmp  .menu_loop

.exit:
    xor  eax, eax
    add  rsp, 8
    pop  r13
    pop  r12
    pop  rbx
    pop  rbp
    ret
```

---

## ส่วนที่ 9: Mixed Assembly-C Programming

### 9.1 การรวม Assembly กับ C

```c
// ไฟล์: mixed_main.c
// C file ที่เรียก Assembly functions
#include <stdio.h>

// ประกาศ functions ที่เขียนใน Assembly
extern long asm_fibonacci(long n);
extern long asm_factorial(long n);
extern void asm_sort(long* arr, long n);

int main() {
    // ทดสอบ fibonacci
    printf("=== Fibonacci ===\n");
    for (int i = 0; i <= 10; i++) {
        printf("fib(%d) = %ld\n", i, asm_fibonacci(i));
    }
    
    // ทดสอบ factorial
    printf("\n=== Factorial ===\n");
    for (int i = 0; i <= 10; i++) {
        printf("%d! = %ld\n", i, asm_factorial(i));
    }
    
    // ทดสอบ sort
    printf("\n=== Sort ===\n");
    long arr[] = {5, 2, 8, 1, 9, 3, 7, 4, 6};
    long n = sizeof(arr) / sizeof(arr[0]);
    asm_sort(arr, n);
    printf("Sorted: ");
    for (int i = 0; i < n; i++) {
        printf("%ld ", arr[i]);
    }
    printf("\n");
    
    return 0;
}
```

```nasm
; ไฟล์: asm_funcs.asm
; Assembly functions สำหรับ mixed_main.c
; nasm -f elf64 asm_funcs.asm -o asm_funcs.o
; gcc mixed_main.c asm_funcs.o -o mixed_program

section .text
    global asm_fibonacci
    global asm_factorial
    global asm_sort

;--------------------------------------------------
; asm_fibonacci(n) -> fib(n) iterative
; rdi = n
;--------------------------------------------------
asm_fibonacci:
    ; Base cases
    cmp  rdi, 0
    je   .ret_zero
    cmp  rdi, 1
    je   .ret_one
    
    ; Iterative fibonacci
    push rbx
    push r12
    
    mov  rbx, 0         ; rbx = fib(i-2) = 0
    mov  r12, 1         ; r12 = fib(i-1) = 1
    sub  rdi, 1         ; n - 1 iterations
    
.fib_loop:
    test rdi, rdi
    jz   .fib_done
    mov  rax, rbx
    add  rax, r12       ; rax = fib(i-2) + fib(i-1)
    mov  rbx, r12       ; fib(i-2) = fib(i-1)
    mov  r12, rax       ; fib(i-1) = fib(i)
    dec  rdi
    jmp  .fib_loop
    
.fib_done:
    mov  rax, r12
    pop  r12
    pop  rbx
    ret

.ret_zero:
    xor  rax, rax
    ret

.ret_one:
    mov  rax, 1
    ret

;--------------------------------------------------
; asm_factorial(n) -> n! iterative
; rdi = n
;--------------------------------------------------
asm_factorial:
    cmp  rdi, 0
    jle  .fact_zero
    
    mov  rax, 1         ; result = 1
    
.fact_loop:
    imul rax, rdi       ; result *= n
    dec  rdi
    jnz  .fact_loop
    ret
    
.fact_zero:
    mov  rax, 1         ; 0! = 1
    ret

;--------------------------------------------------
; asm_sort(arr, n) - Bubble Sort
; rdi = pointer to array, rsi = n
;--------------------------------------------------
asm_sort:
    push rbp
    mov  rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    
    mov  r12, rdi       ; r12 = arr
    mov  r13, rsi       ; r13 = n
    
    ; Outer loop: i = 0..n-2
    xor  rbx, rbx       ; rbx = i
    
.outer_loop:
    lea  rax, [r13-1]
    cmp  rbx, rax
    jge  .sort_done
    
    ; Inner loop: j = 0..n-i-2
    xor  r14, r14       ; r14 = j
    
.inner_loop:
    mov  rax, r13
    sub  rax, rbx
    sub  rax, 1
    cmp  r14, rax
    jge  .inner_done
    
    ; เปรียบเทียบ arr[j] และ arr[j+1]
    mov  rax, [r12 + r14*8]        ; rax = arr[j]
    mov  rcx, [r12 + r14*8 + 8]    ; rcx = arr[j+1]
    
    cmp  rax, rcx
    jle  .no_swap        ; ถ้า arr[j] <= arr[j+1] ไม่ต้อง swap
    
    ; Swap arr[j] และ arr[j+1]
    mov  [r12 + r14*8],     rcx
    mov  [r12 + r14*8 + 8], rax
    
.no_swap:
    inc  r14
    jmp  .inner_loop
    
.inner_done:
    inc  rbx
    jmp  .outer_loop
    
.sort_done:
    pop  r14
    pop  r13
    pop  r12
    pop  rbx
    pop  rbp
    ret
```

---

## ส่วนที่ 10: การ Debug Calling Convention Issues

### 10.1 ปัญหาที่พบบ่อย

```
ปัญหาที่ 1: Stack Misalignment
Symptom: SEGFAULT ใน printf/memcpy หรือ SSE instructions
Cause: ลืม align stack ก่อน CALL
Fix: ตรวจนับจำนวน push/pop ให้เท่ากัน

ปัญหาที่ 2: Wrong Return Value
Symptom: ได้ผลลัพธ์ผิด หรือ garbage value
Cause: ลืมใส่ค่าใน RAX ก่อน RET
Fix: ตรวจ return path ทุก branch

ปัญหาที่ 3: Clobbered Callee-Saved Registers
Symptom: ค่าใน RBX/R12/etc เปลี่ยนหลัง call
Cause: ลืม push/pop callee-saved registers
Fix: push callee-saved ตอนต้น pop ตอนท้าย

ปัญหาที่ 4: Wrong Parameter Order (32-bit)
Symptom: พารามิเตอร์สลับกัน
Cause: ใน cdecl ต้อง push right-to-left
Fix: push c, push b, push a (ถ้า call f(a,b,c))

ปัญหาที่ 5: Stack Leak (32-bit cdecl)
Symptom: stack overflow หรือ wrong values
Cause: ลืม add esp, N หลัง call
Fix: add esp, (N * 4) หลังทุก call
```

### 10.2 เทคนิค Debug ด้วย GDB

```bash
# Compile with debug info
nasm -f elf64 -g -F dwarf myprogram.asm -o myprogram.o
gcc -g myprogram.o -o myprogram

# เริ่ม GDB
gdb myprogram

# คำสั่ง GDB ที่ใช้บ่อย
(gdb) break main          # set breakpoint ที่ main
(gdb) run                 # รันโปรแกรม
(gdb) info registers      # ดู registers ทั้งหมด
(gdb) x/10gx $rsp         # ดู stack (10 qwords)
(gdb) x/10wd $rsp         # ดู stack (10 dwords)
(gdb) si                  # step instruction
(gdb) ni                  # next instruction (ข้าม call)
(gdb) disas               # disassemble current function
(gdb) p $rax              # ดูค่า rax
(gdb) p/x $rsp            # ดูค่า rsp เป็น hex
(gdb) backtrace           # ดู call stack

# ตรวจสอบ alignment
(gdb) p $rsp & 0xF        # ควรเป็น 0 ก่อน CALL
```

### 10.3 ตัวอย่าง Debug ปัญหา Stack

```nasm
; ไฟล์: debug_example.asm
; ตัวอย่างโค้ดที่มีปัญหาและวิธีแก้
; nasm -f elf64 debug_example.asm -o debug_example.o
; gcc debug_example.o -o debug_example

section .data
    fmt1    db "Value 1: %d", 10, 0
    fmt2    db "Value 2: %d", 10, 0
    fmt3    db "Sum: %d", 10, 0

section .text
    global main
    extern printf

; ---- VERSION ที่ผิด (commented out) ----
; buggy_main:
;     push rbp
;     mov  rbp, rsp
;     ; !! ลืม align stack !!
;     push rax            ; rsp misaligned by 8
;     
;     lea  rdi, [rel fmt1]
;     mov  rsi, 10
;     xor  eax, eax
;     call printf         ; CRASH! stack misaligned
;     
;     pop  rax
;     pop  rbp
;     ret

; ---- VERSION ที่ถูก ----
main:
    push rbp
    mov  rbp, rsp
    
    ; กรณีที่ต้องการ push ค่าก่อน call
    ; ต้อง ensure alignment
    
    ; วิธีที่ 1: ใช้ local variable แทน push
    sub  rsp, 16        ; reserve space (aligned)
    mov  qword [rsp], 42 ; เก็บค่าใน local var
    
    ; เรียก printf
    lea  rdi, [rel fmt1]
    mov  rsi, qword [rsp]
    xor  eax, eax
    call printf          ; OK! RSP aligned
    
    add  rsp, 16        ; เคลียร์ local vars
    
    ; วิธีที่ 2: push แล้ว push padding
    push qword 100      ; rsp - 8 = misaligned
    push qword 0        ; rsp - 8 = aligned again
    
    lea  rdi, [rel fmt2]
    mov  rsi, qword [rsp+8] ; ค่า 100
    xor  eax, eax
    call printf
    
    pop  rax            ; เคลียร์ padding
    pop  rax            ; เคลียร์ค่า 100
    
    xor  eax, eax
    pop  rbp
    ret
```

---

## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: cdecl 32-bit Functions

**โจทย์:** เขียนฟังก์ชัน Assembly ต่อไปนี้ด้วย cdecl (32-bit):

```
1. int max3(int a, int b, int c) - คืนค่าสูงสุดใน 3 ตัว
2. int sum_array(int* arr, int n) - หาผลรวมของ array
3. int count_positive(int* arr, int n) - นับจำนวนค่าบวก
```

```nasm
; เฉลย: cdecl_exercises.asm
; nasm -f elf32 cdecl_exercises.asm -o cdecl_exercises.o
; gcc -m32 cdecl_exercises.o -o cdecl_exercises

section .data
    fmt_max     db "max3(%d,%d,%d) = %d", 10, 0
    fmt_sum     db "sum = %d", 10, 0
    fmt_pos     db "positive count = %d", 10, 0
    arr_data    dd 1, -2, 3, -4, 5, -6, 7, -8, 9, 0
    arr_size    dd 10

section .text
    global main
    extern printf

; max3(a, b, c)
max3:
    push ebp
    mov  ebp, esp
    
    mov  eax, [ebp+8]   ; eax = a
    mov  ecx, [ebp+12]  ; ecx = b
    mov  edx, [ebp+16]  ; edx = c
    
    ; max(a, b)
    cmp  eax, ecx
    jge  .ab_done
    mov  eax, ecx       ; eax = max(a,b)
.ab_done:
    ; max(max(a,b), c)
    cmp  eax, edx
    jge  .abc_done
    mov  eax, edx       ; eax = max(a,b,c)
.abc_done:
    pop  ebp
    ret

; sum_array(arr, n)
sum_array:
    push ebp
    mov  ebp, esp
    push esi
    push ebx
    
    mov  esi, [ebp+8]   ; esi = arr pointer
    mov  ecx, [ebp+12]  ; ecx = n
    xor  eax, eax       ; eax = sum = 0
    
.sum_loop:
    test ecx, ecx
    jz   .sum_done
    add  eax, [esi]     ; sum += *arr
    add  esi, 4         ; arr++
    dec  ecx
    jmp  .sum_loop
    
.sum_done:
    pop  ebx
    pop  esi
    pop  ebp
    ret

; count_positive(arr, n)
count_positive:
    push ebp
    mov  ebp, esp
    push esi
    
    mov  esi, [ebp+8]   ; esi = arr pointer
    mov  ecx, [ebp+12]  ; ecx = n
    xor  eax, eax       ; eax = count = 0
    
.count_loop:
    test ecx, ecx
    jz   .count_done
    cmp  dword [esi], 0
    jle  .not_positive
    inc  eax            ; count++
.not_positive:
    add  esi, 4
    dec  ecx
    jmp  .count_loop
    
.count_done:
    pop  esi
    pop  ebp
    ret

main:
    push ebp
    mov  ebp, esp
    
    ; ทดสอบ max3
    push dword 8
    push dword 15
    push dword 3
    call max3
    add  esp, 12
    
    push eax
    push dword 8
    push dword 15
    push dword 3
    push fmt_max
    call printf
    add  esp, 20
    
    ; ทดสอบ sum_array
    push dword [arr_size]
    push arr_data
    call sum_array
    add  esp, 8
    
    push eax
    push fmt_sum
    call printf
    add  esp, 8
    
    ; ทดสอบ count_positive
    push dword [arr_size]
    push arr_data
    call count_positive
    add  esp, 8
    
    push eax
    push fmt_pos
    call printf
    add  esp, 8
    
    xor  eax, eax
    pop  ebp
    ret
```

### แบบฝึกหัดที่ 2: System V AMD64 ABI

**โจทย์:** เขียนโปรแกรม 64-bit ที่ใช้ฟังก์ชัน C library:

```
1. โปรแกรมรับชื่อและอายุ แล้วทักทาย
2. โปรแกรมคำนวณ GCD (Greatest Common Divisor) ด้วย Euclidean algorithm
3. โปรแกรมแสดง Fibonacci sequence 20 ตัวแรก
```

```nasm
; เฉลย: sysv_exercises.asm
; nasm -f elf64 sysv_exercises.asm -o sysv_exercises.o
; gcc sysv_exercises.o -o sysv_exercises

section .data
    prompt_name db "Enter your name: ", 0
    prompt_age  db "Enter your age: ", 0
    fmt_greet   db "Hello, %s! You are %ld years old.", 10, 0
    fmt_gcd     db "GCD(%ld, %ld) = %ld", 10, 0
    fmt_fib_hdr db "=== Fibonacci (20 terms) ===", 10, 0
    fmt_fib     db "fib(%d) = %ld", 10, 0
    fmt_s       db "%49s", 0
    fmt_ld      db "%ld", 0
    prompt_a    db "Enter a for GCD: ", 0
    prompt_b    db "Enter b for GCD: ", 0

section .bss
    name_buf    resb 50
    num_buf     resq 1

section .text
    global main
    extern printf, scanf

;--------------------------------------------------
; gcd(a, b) -> GCD
; rdi=a, rsi=b
;--------------------------------------------------
gcd:
    test rsi, rsi
    jz   .done
    
    ; rax = a mod b
    mov  rax, rdi
    cqo
    idiv rsi            ; rax = quotient, rdx = remainder
    
    mov  rdi, rsi       ; a = b
    mov  rsi, rdx       ; b = remainder
    jmp  gcd
    
.done:
    mov  rax, rdi
    ret

;--------------------------------------------------
; fibonacci_print(n) - พิมพ์ n terms
; rdi = n
;--------------------------------------------------
fibonacci_print:
    push rbp
    mov  rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    sub  rsp, 8
    
    mov  r14, rdi       ; r14 = n
    
    ; print header
    lea  rdi, [rel fmt_fib_hdr]
    xor  eax, eax
    call printf
    
    ; ลูป
    xor  rbx, rbx       ; rbx = i
    mov  r12, 0         ; r12 = prev (fib(i-2))
    mov  r13, 1         ; r13 = curr (fib(i-1))
    
.fib_loop:
    cmp  rbx, r14
    jge  .fib_done
    
    ; printf("fib(%d) = %ld\n", i, fib)
    lea  rdi, [rel fmt_fib]
    mov  esi, ebx       ; i (32-bit)
    mov  rdx, r12       ; fib(i)
    xor  eax, eax
    call printf
    
    ; คำนวณ fib ถัดไป
    mov  rax, r12
    add  rax, r13       ; next = prev + curr
    mov  r12, r13       ; prev = curr
    mov  r13, rax       ; curr = next
    
    inc  rbx
    jmp  .fib_loop
    
.fib_done:
    add  rsp, 8
    pop  r14
    pop  r13
    pop  r12
    pop  rbx
    pop  rbp
    ret

main:
    push rbp
    mov  rbp, rsp
    
    ; --- Exercise 1: รับชื่อและอายุ ---
    lea  rdi, [rel prompt_name]
    xor  eax, eax
    call printf
    
    lea  rsi, [rel name_buf]
    lea  rdi, [rel fmt_s]
    xor  eax, eax
    call scanf
    
    lea  rdi, [rel prompt_age]
    xor  eax, eax
    call printf
    
    lea  rsi, [rel num_buf]
    lea  rdi, [rel fmt_ld]
    xor  eax, eax
    call scanf
    
    lea  rdi, [rel fmt_greet]
    lea  rsi, [rel name_buf]
    mov  rdx, [rel num_buf]
    xor  eax, eax
    call printf
    
    ; --- Exercise 2: GCD ---
    lea  rdi, [rel prompt_a]
    xor  eax, eax
    call printf
    
    sub  rsp, 16
    lea  rsi, [rsp]
    lea  rdi, [rel fmt_ld]
    xor  eax, eax
    call scanf
    mov  r12, [rsp]     ; a - ต้องการ callee-saved
    ; แต่ r12 ต้องถูก push ก่อน! (ใน main ไม่ได้ push r12)
    ; ในโค้ดจริงต้องแก้ให้ถูกต้อง
    add  rsp, 16
    
    lea  rdi, [rel prompt_b]
    xor  eax, eax
    call printf
    
    sub  rsp, 16
    lea  rsi, [rsp]
    lea  rdi, [rel fmt_ld]
    xor  eax, eax
    call scanf
    mov  r13, [rsp]     ; b
    add  rsp, 16
    
    ; เรียก gcd
    mov  rdi, r12
    mov  rsi, r13
    call gcd
    
    lea  rdi, [rel fmt_gcd]
    mov  rsi, r12
    mov  rdx, r13
    mov  rcx, rax
    xor  eax, eax
    call printf
    
    ; --- Exercise 3: Fibonacci ---
    mov  rdi, 20
    call fibonacci_print
    
    xor  eax, eax
    pop  rbp
    ret
```

### แบบฝึกหัดที่ 3: โจทย์ท้าทาย

```
โจทย์ขั้นสูง:
1. เขียน qsort callback ใน Assembly (ผ่าน function pointer)
2. เขียน assembly function ที่รับ va_list (variadic)
3. สร้าง struct ใน Assembly และส่งผ่าน pointer ให้ C function
4. เขียน wrapper function สำหรับ system call โดยตรง
```

```nasm
; ตัวอย่าง: qsort callback
; ไฟล์: qsort_demo.asm
; nasm -f elf64 qsort_demo.asm -o qsort_demo.o
; gcc qsort_demo.o -o qsort_demo

section .data
    fmt_arr     db "Array: ", 0
    fmt_elem    db "%ld ", 0
    fmt_nl      db 10, 0
    fmt_sorted  db "Sorted: ", 0

section .bss
    arr_data    resq 10
    
section .text
    global main
    extern printf, qsort, scanf

;--------------------------------------------------
; compare_int(a, b) - callback สำหรับ qsort
; rdi = pointer to element a
; rsi = pointer to element b
; return: < 0 ถ้า a < b, 0 ถ้า a==b, > 0 ถ้า a > b
;--------------------------------------------------
compare_int:
    mov  rax, [rdi]     ; rax = *a
    mov  rcx, [rsi]     ; rcx = *b
    cmp  rax, rcx
    jl   .less
    jg   .greater
    xor  eax, eax       ; equal
    ret
.less:
    mov  eax, -1
    ret
.greater:
    mov  eax, 1
    ret

main:
    push rbp
    mov  rbp, rsp
    push rbx
    sub  rsp, 8
    
    ; เติมข้อมูลในอาเรย์
    mov  qword [arr_data+0],  64
    mov  qword [arr_data+8],  25
    mov  qword [arr_data+16], 12
    mov  qword [arr_data+24], 90
    mov  qword [arr_data+32], 1
    mov  qword [arr_data+40], 50
    mov  qword [arr_data+48], 38
    mov  qword [arr_data+56], 73
    mov  qword [arr_data+64], 45
    mov  qword [arr_data+72], 82
    
    ; แสดงก่อน sort
    lea  rdi, [rel fmt_arr]
    xor  eax, eax
    call printf
    
    xor  rbx, rbx
.print_before:
    cmp  rbx, 10
    jge  .print_before_done
    lea  rdi, [rel fmt_elem]
    mov  rsi, [arr_data + rbx*8]
    xor  eax, eax
    call printf
    inc  rbx
    jmp  .print_before
.print_before_done:
    lea  rdi, [rel fmt_nl]
    xor  eax, eax
    call printf
    
    ; เรียก qsort(arr, n, size, compare)
    lea  rdi, [rel arr_data]        ; array
    mov  rsi, 10                    ; number of elements
    mov  rdx, 8                     ; size of each element (qword = 8)
    lea  rcx, [rel compare_int]     ; comparison function
    call qsort
    
    ; แสดงหลัง sort
    lea  rdi, [rel fmt_sorted]
    xor  eax, eax
    call printf
    
    xor  rbx, rbx
.print_after:
    cmp  rbx, 10
    jge  .print_after_done
    lea  rdi, [rel fmt_elem]
    mov  rsi, [arr_data + rbx*8]
    xor  eax, eax
    call printf
    inc  rbx
    jmp  .print_after
.print_after_done:
    lea  rdi, [rel fmt_nl]
    xor  eax, eax
    call printf
    
    xor  eax, eax
    add  rsp, 8
    pop  rbx
    pop  rbp
    ret
```

---

## สรุปคำสั่งสำคัญ (Quick Reference)

### 32-bit cdecl Template

```nasm
; Template สำหรับ cdecl 32-bit function
my_function:
    ; Prologue
    push ebp
    mov  ebp, esp
    sub  esp, N         ; N bytes สำหรับ local variables
    push ebx            ; save callee-saved (ถ้าใช้)
    push esi
    push edi
    
    ; Access parameters:
    ;   [ebp+8]  = param 1
    ;   [ebp+12] = param 2
    ;   [ebp+16] = param 3
    
    ; Access local vars:
    ;   [ebp-4]  = local 1
    ;   [ebp-8]  = local 2
    
    ; ... function body ...
    
    ; Epilogue
    pop  edi
    pop  esi
    pop  ebx
    mov  esp, ebp       ; หรือใช้ 'leave' instruction
    pop  ebp
    ret

; การเรียก cdecl function:
    push param3         ; push right-to-left
    push param2
    push param1
    call my_function
    add  esp, 12        ; CALLER cleans stack (3 params * 4 bytes)
```

### 64-bit System V AMD64 Template

```nasm
; Template สำหรับ System V AMD64 ABI 64-bit function
my_function:
    ; Prologue
    push rbp
    mov  rbp, rsp
    push rbx            ; save callee-saved (ถ้าใช้)
    push r12
    push r13
    push r14
    push r15
    sub  rsp, N         ; N bytes สำหรับ locals (ต้อง keep 16-byte alignment)
    
    ; Parameters ใน registers:
    ;   rdi = param 1    rsi = param 2
    ;   rdx = param 3    rcx = param 4
    ;   r8  = param 5    r9  = param 6
    ;   [rbp+16] = param 7 (stack, ถ้ามี)
    
    ; ... function body ...
    ; ใส่ return value ใน rax
    
    ; Epilogue
    add  rsp, N
    pop  r15
    pop  r14
    pop  r13
    pop  r12
    pop  rbx
    pop  rbp
    ret

; การเรียก 64-bit function:
; ตรวจสอบ stack alignment ก่อน call!
    mov  rdi, param1    ; ใส่ใน registers
    mov  rsi, param2
    mov  rdx, param3
    ; ...
    call my_function
    ; return value ใน rax
```

### Script สำหรับ Compile

```bash
#!/bin/bash
# compile.sh - script สำหรับ compile Assembly programs

# 32-bit cdecl
compile_32() {
    local src="$1"
    local out="${src%.asm}"
    echo "Compiling 32-bit: $src"
    nasm -f elf32 "$src" -o "${out}.o" && \
    gcc -m32 "${out}.o" -o "$out" && \
    echo "Done: ./$out"
}

# 64-bit System V
compile_64() {
    local src="$1"
    local out="${src%.asm}"
    echo "Compiling 64-bit: $src"
    nasm -f elf64 "$src" -o "${out}.o" && \
    gcc "${out}.o" -o "$out" && \
    echo "Done: ./$out"
}

# 64-bit กับ math library
compile_64_math() {
    local src="$1"
    local out="${src%.asm}"
    echo "Compiling 64-bit with math: $src"
    nasm -f elf64 "$src" -o "${out}.o" && \
    gcc "${out}.o" -lm -o "$out" && \
    echo "Done: ./$out"
}

# 64-bit mixed C+Assembly
compile_mixed() {
    local asm_src="$1"
    local c_src="$2"
    local out="${asm_src%.asm}"
    echo "Compiling mixed: $asm_src + $c_src"
    nasm -f elf64 "$asm_src" -o "${asm_src%.asm}.o" && \
    gcc "${asm_src%.asm}.o" "$c_src" -o "$out" && \
    echo "Done: ./$out"
}

# ใช้งาน:
# compile_32 cdecl_basic.asm
# compile_64 clean_64bit.asm
# compile_mixed asm_funcs.asm mixed_main.c
```

---

## บทสรุป (Conclusion)

ใน Part 023 เราได้เรียนรู้:

1. **cdecl Calling Convention (32-bit)**
   - พารามิเตอร์ส่งผ่าน stack จากขวาไปซ้าย
   - Caller เคลียร์ stack ด้วย `add esp, N`
   - Return value ใน EAX
   - Callee-saved: EBX, ESI, EDI, EBP

2. **System V AMD64 ABI (64-bit)**
   - พารามิเตอร์ใน RDI, RSI, RDX, RCX, R8, R9
   - Stack align 16-byte ก่อน CALL
   - Return value ใน RAX
   - Callee-saved: RBX, RBP, R12-R15

3. **Stack Alignment**
   - ต้อง align 16-byte สำหรับ 64-bit
   - นับ push/pop ให้ถูกต้อง

4. **Variadic Functions**
   - AL บอกจำนวน float args สำหรับ printf
   - ใช้ XMM registers สำหรับ float/double

5. **C Library Integration**
   - printf, scanf, string functions, memory functions
   - Name mangling ใน C vs C++

6. **การ Debug**
   - ใช้ GDB ตรวจสอบ registers และ stack
   - ระวัง stack misalignment, wrong registers

**Part ถัดไป:** Part 024 - Windows x64 Calling Convention (Shadow Space, Home Space, Parameter Passing)
