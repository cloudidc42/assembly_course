# Part 021: Procedures และ Subroutines ใน Assembly

## ภาพรวม (Overview)

ใน Assembly การเรียกใช้ procedures และ subroutines เป็นหัวใจสำคัญของการเขียนโปรแกรมที่มีโครงสร้าง
บทนี้จะครอบคลุมทุกแง่มุมของ CALL/RET mechanics ตั้งแต่พื้นฐานจนถึงเทคนิคขั้นสูง
รวมถึงการป้องกันช่องโหว่ด้านความปลอดภัยที่เกิดจากการจัดการ return address

---

## 1. CALL Instruction พื้นฐาน

### 1.1 Near Call vs Far Call

```nasm
; ============================================================
; ไฟล์: call_basics.asm
; วัตถุประสงค์: แสดงความแตกต่างระหว่าง near call และ far call
; คอมไพล์: nasm -f elf64 call_basics.asm -o call_basics.o
;           ld call_basics.o -o call_basics
; ============================================================

section .data
    msg_near    db "Near call executed", 0x0A, 0
    msg_near_len equ $ - msg_near
    msg_far     db "Far call demonstration", 0x0A, 0
    msg_far_len equ $ - msg_far

section .text
    global _start

_start:
    ; --- Near Call (การเรียกแบบ near - ภายใน segment เดียวกัน) ---
    ; CALL แบบ near จะ push เฉพาะ EIP/RIP (return address) ลง stack
    ; และกระโดดไปยัง address ที่ระบุภายใน code segment เดียวกัน
    call near_procedure      ; push rip, jmp near_procedure

    ; --- Indirect Call (การเรียกผ่าน register หรือ memory) ---
    lea rax, [rel indirect_procedure]  ; โหลด address ของ procedure เข้า RAX
    call rax                           ; เรียกผ่าน register (indirect call)

    ; จบโปรแกรม
    mov rax, 60         ; syscall: exit
    xor rdi, rdi        ; exit code 0
    syscall

; --- Near Procedure ---
; Procedure นี้อยู่ใน segment เดียวกับ caller
near_procedure:
    ; แสดงข้อความ
    mov rax, 1              ; syscall: write
    mov rdi, 1              ; fd: stdout
    mov rsi, msg_near       ; pointer ไปยังข้อความ
    mov rdx, msg_near_len   ; ความยาวข้อความ
    syscall
    ret                     ; กลับไปยัง caller (near return)

; --- Indirect Procedure ---
indirect_procedure:
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_far
    mov rdx, msg_far_len
    syscall
    ret
```

### 1.2 Direct vs Indirect CALL

```nasm
; ============================================================
; ไฟล์: direct_indirect_call.asm
; วัตถุประสงค์: เปรียบเทียบ direct และ indirect call
; คอมไพล์: nasm -f elf64 direct_indirect_call.asm -o dic.o && ld dic.o -o dic
; ============================================================

section .data
    result_msg  db "Function called via: "
    call_type   db "DIRECT  ", 0x0A, 0  ; 8 ตัวอักษรเพื่อให้ overwrite ได้
    result_len  equ $ - result_msg

section .bss
    ; ไม่มีตัวแปรที่ไม่ได้กำหนดค่า

section .text
    global _start

; --- Macro สำหรับ print string ---
%macro print_str 2
    mov rax, 1
    mov rdi, 1
    mov rsi, %1
    mov rdx, %2
    syscall
%endmacro

_start:
    ; 1. Direct Call - Assembler รู้ address ตอน assemble time
    ;    ผลลัพธ์: opcode E8 xx xx xx xx (relative offset)
    call my_function        ; direct call

    ; 2. Indirect Call ผ่าน register
    lea rbx, [rel my_function]  ; โหลด address เข้า RBX
    call rbx                     ; indirect call ผ่าน register
                                 ; opcode: FF D3

    ; 3. Indirect Call ผ่าน memory (function pointer)
    lea rax, [rel func_ptr]      ; โหลด address ของ func_ptr
    call [rax]                   ; เรียกผ่าน memory
                                 ; opcode: FF 10

    ; Exit
    mov rax, 60
    xor rdi, rdi
    syscall

my_function:
    ; procedure body
    print_str result_msg, result_len
    ret

section .data
    func_ptr    dq my_function  ; function pointer ใน data section
```

---

## 2. Stack Mechanics เมื่อเรียก CALL

### 2.1 สิ่งที่เกิดขึ้นใน Stack

```
ก่อน CALL:
┌─────────────────┐
│  ...            │  ← higher address
│  local var      │
│  saved rbp      │
│  ...            │
└─────────────────┘  ← RSP ชี้อยู่ที่นี่

หลัง CALL (CPU push return address อัตโนมัติ):
┌─────────────────┐
│  ...            │
│  local var      │
│  saved rbp      │
│  ...            │
│  return address │  ← RSP ชี้อยู่ที่นี่ (RSP ลดลง 8 bytes บน 64-bit)
└─────────────────┘
```

```nasm
; ============================================================
; ไฟล์: stack_mechanics.asm
; วัตถุประสงค์: แสดง stack mechanics ของ CALL/RET
; คอมไพล์: nasm -f elf64 stack_mechanics.asm -o sm.o && ld sm.o -o sm
; ============================================================

section .data
    fmt_rsp     db "RSP before CALL: 0x"
    fmt_rsp_len equ $ - fmt_rsp
    newline     db 0x0A

section .text
    global _start

_start:
    ; บันทึก RSP ก่อนเรียก
    mov r12, rsp            ; เก็บ RSP ปัจจุบัน

    ; เรียก procedure ที่จะตรวจสอบ stack
    call examine_stack

    ; ตรวจสอบว่า RSP กลับมาเป็นค่าเดิม
    cmp rsp, r12
    je stack_balanced
    ; ถ้า RSP ไม่เท่ากัน แสดงว่า stack ไม่ balance
    ; (ในโปรแกรมจริงควร handle error)
    jmp exit_program

stack_balanced:
    ; Stack balance ถูกต้อง
    jmp exit_program

examine_stack:
    ; ณ จุดนี้ [rsp] คือ return address ที่ CALL push ให้
    ; RSP ถูก push ให้ลดลง 8 bytes
    
    ; อ่าน return address จาก stack
    mov rax, [rsp]          ; return address อยู่ที่ [rsp]
    
    ; แสดง return address (hex)
    call print_hex_rax

    ret                     ; pop return address จาก stack, กระโดดกลับ

; ============================================================
; print_hex_rax: แสดงค่า RAX เป็น hexadecimal
; Input: RAX = ค่าที่ต้องการแสดง
; Clobbers: RAX, RBX, RCX, RDX, RSI, RDI
; ============================================================
print_hex_rax:
    push rbp
    mov rbp, rsp
    sub rsp, 32             ; จอง space สำหรับ buffer

    ; สร้าง hex string
    lea rsi, [rbp - 16]     ; buffer สำหรับ hex string
    mov rcx, 16             ; 16 หลักสำหรับ 64-bit

    mov rbx, rax            ; เก็บค่าไว้ใน RBX
.loop:
    mov rdx, rbx
    and rdx, 0xF            ; เอา nibble ล่างสุด
    ; แปลง nibble เป็น ASCII
    cmp rdx, 9
    jle .digit
    add rdx, 'A' - 10
    jmp .store
.digit:
    add rdx, '0'
.store:
    mov [rsi + rcx - 1], dl  ; เก็บ character
    shr rbx, 4               ; เลื่อน nibble
    loop .loop

    ; print
    mov rax, 1
    mov rdi, 1
    ; rsi ยังคงชี้ที่ buffer
    mov rdx, 16
    syscall

    ; print newline
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel newline]
    mov rdx, 1
    syscall

    mov rsp, rbp
    pop rbp
    ret

exit_program:
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 3. Procedure Prologue และ Epilogue

### 3.1 Standard Prologue/Epilogue

```nasm
; ============================================================
; ไฟล์: prologue_epilogue.asm
; วัตถุประสงค์: แสดง standard procedure prologue/epilogue
; คอมไพล์: nasm -f elf64 prologue_epilogue.asm -o pe.o && ld pe.o -o pe
; ============================================================

; Stack frame หลังจาก prologue:
; ┌──────────────────┐  ← higher address
; │  (caller frame)  │
; │  return address  │  ← [rbp + 8]
; │  saved old rbp   │  ← [rbp + 0]  (rbp ชี้ที่นี่)
; │  local var 1     │  ← [rbp - 8]
; │  local var 2     │  ← [rbp - 16]
; │  ...             │
; └──────────────────┘  ← RSP (stack pointer)

section .data
    result_str  db "Result: ", 0
    result_len  equ $ - result_str
    newline     db 0x0A

section .text
    global _start

_start:
    ; เรียก function พร้อม arguments ตาม System V AMD64 ABI
    ; RDI = arg1, RSI = arg2, RDX = arg3, RCX = arg4, R8 = arg5, R9 = arg6
    mov rdi, 10         ; arg1: a = 10
    mov rsi, 20         ; arg2: b = 20
    call add_numbers    ; เรียก function

    ; RAX มีค่าผลลัพธ์ (30)
    ; แสดงผล
    push rax            ; เก็บผลลัพธ์ไว้ก่อน
    
    mov rax, 1
    mov rdi, 1
    mov rsi, result_str
    mov rdx, result_len
    syscall

    pop rax             ; คืนผลลัพธ์กลับ
    call print_decimal

    mov rax, 1
    mov rdi, 1
    lea rsi, [rel newline]
    mov rdx, 1
    syscall

    mov rax, 60
    xor rdi, rdi
    syscall

; ============================================================
; add_numbers: บวกสองจำนวน
; Input:  RDI = a, RSI = b
; Output: RAX = a + b
; Local variables:
;   [rbp - 8]  = local_a (สำเนา a)
;   [rbp - 16] = local_b (สำเนา b)
; ============================================================
add_numbers:
    ; === PROLOGUE ===
    push rbp            ; บันทึก base pointer เดิมไว้ใน stack
    mov rbp, rsp        ; ตั้งค่า base pointer ใหม่
    sub rsp, 16         ; จอง space สำหรับ local variables (2 x 8 bytes)
    
    ; บันทึก callee-saved registers ที่จะใช้
    ; (ในตัวอย่างนี้ไม่ได้ใช้ r12-r15, rbx, rbp เพิ่มเติม)
    
    ; === FUNCTION BODY ===
    ; เก็บ arguments เป็น local variables
    mov [rbp - 8], rdi   ; local_a = a
    mov [rbp - 16], rsi  ; local_b = b
    
    ; คำนวณผลรวม
    mov rax, [rbp - 8]   ; โหลด local_a
    add rax, [rbp - 16]  ; บวก local_b
    ; ผลลัพธ์อยู่ใน RAX
    
    ; === EPILOGUE ===
    ; คืน callee-saved registers (ถ้ามี)
    
    mov rsp, rbp        ; คืน stack pointer (deallocate local variables)
    pop rbp             ; คืน base pointer เดิม
    ret                 ; กลับไปยัง caller

; ============================================================
; print_decimal: แสดงตัวเลข decimal
; Input: RAX = ตัวเลขที่ต้องการแสดง
; ============================================================
print_decimal:
    push rbp
    mov rbp, rsp
    sub rsp, 32

    ; ตรวจสอบว่าเป็นลบหรือไม่
    test rax, rax
    jns .positive
    push rax
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel minus_sign]
    mov rdx, 1
    syscall
    pop rax
    neg rax

.positive:
    ; แปลงเป็น string
    lea rdi, [rbp - 20]  ; buffer
    mov byte [rbp - 20], 0
    mov rcx, 0           ; digit count

    ; ถ้าเป็น 0
    test rax, rax
    jnz .convert_loop
    mov byte [rbp - 21], '0'
    inc rcx
    jmp .print_digits

.convert_loop:
    test rax, rax
    jz .print_digits
    mov rdx, 0
    mov rbx, 10
    div rbx              ; RDX:RAX / RBX
    add dl, '0'
    dec rdi
    mov [rdi], dl
    inc rcx
    jmp .convert_loop

.print_digits:
    mov rax, 1
    mov rsi, rdi
    mov rdx, rcx
    mov rdi, 1
    syscall

    mov rsp, rbp
    pop rbp
    ret

section .data
    minus_sign  db "-"
```

### 3.2 การใช้ ENTER และ LEAVE

```nasm
; ============================================================
; ไฟล์: enter_leave.asm
; วัตถุประสงค์: แสดงการใช้ ENTER/LEAVE instructions
; คอมไพล์: nasm -f elf64 enter_leave.asm -o el.o && ld el.o -o el
; หมายเหตุ: ENTER/LEAVE เป็น x86 instructions ที่ทำสิ่งเดียวกับ
;           push rbp / mov rbp, rsp / sub rsp, N
;           แต่ช้ากว่าในโปรเซสเซอร์สมัยใหม่
; ============================================================

section .data
    hello   db "Hello from ENTER/LEAVE procedure", 0x0A, 0
    hello_len equ $ - hello

section .text
    global _start

_start:
    call demo_enter_leave
    mov rax, 60
    xor rdi, rdi
    syscall

demo_enter_leave:
    ; ENTER immediate16, immediate8
    ; immediate16 = จำนวน bytes สำหรับ local variables
    ; immediate8  = nesting level (0 สำหรับ non-nested)
    enter 16, 0     ; เทียบเท่ากับ: push rbp / mov rbp, rsp / sub rsp, 16
    
    ; ใช้ local variables
    mov qword [rbp - 8], 42    ; local variable 1
    mov qword [rbp - 16], 100  ; local variable 2
    
    ; แสดงข้อความ
    mov rax, 1
    mov rdi, 1
    mov rsi, hello
    mov rdx, hello_len
    syscall
    
    ; LEAVE เทียบเท่ากับ: mov rsp, rbp / pop rbp
    leave
    ret
```

---

## 4. Nested Calls และ Call Stack

```nasm
; ============================================================
; ไฟล์: nested_calls.asm
; วัตถุประสงค์: แสดง nested procedure calls และ stack ที่เกิดขึ้น
; คอมไพล์: nasm -f elf64 nested_calls.asm -o nc.o && ld nc.o -o nc
; ============================================================
;
; Call chain: _start -> level1 -> level2 -> level3
;
; Stack ขณะอยู่ใน level3:
; ┌────────────────────────────────┐ higher address
; │  _start's stack frame          │
; │  return addr (back to _start)  │
; ├────────────────────────────────┤
; │  level1's saved rbp            │
; │  level1's local vars           │
; │  return addr (back to level1)  │
; ├────────────────────────────────┤
; │  level2's saved rbp            │
; │  level2's local vars           │
; │  return addr (back to level2)  │
; ├────────────────────────────────┤
; │  level3's saved rbp            │  ← RBP ชี้ที่นี่
; │  level3's local vars           │
; └────────────────────────────────┘  ← RSP ชี้ที่นี่ (lowest address)

section .data
    msg1    db "Level 1: Entering", 0x0A, 0
    msg1l   equ $ - msg1
    msg2    db "Level 2: Entering", 0x0A, 0
    msg2l   equ $ - msg2
    msg3    db "Level 3: Deep call!", 0x0A, 0
    msg3l   equ $ - msg3
    msg3r   db "Level 3: Returning", 0x0A, 0
    msg3rl  equ $ - msg3r
    msg2r   db "Level 2: Returning", 0x0A, 0
    msg2rl  equ $ - msg2r
    msg1r   db "Level 1: Returning", 0x0A, 0
    msg1rl  equ $ - msg1r
    done    db "All calls completed!", 0x0A, 0
    donel   equ $ - done

section .text
    global _start

; Macro สำหรับ print
%macro say 2
    push rax
    push rdi
    push rsi
    push rdx
    mov rax, 1
    mov rdi, 1
    mov rsi, %1
    mov rdx, %2
    syscall
    pop rdx
    pop rsi
    pop rdi
    pop rax
%endmacro

_start:
    call level1

    say done, donel

    mov rax, 60
    xor rdi, rdi
    syscall

; ============================================================
level1:
    push rbp
    mov rbp, rsp
    sub rsp, 8              ; local: depth counter

    say msg1, msg1l

    mov qword [rbp - 8], 1  ; depth = 1
    call level2

    say msg1r, msg1rl

    mov rsp, rbp
    pop rbp
    ret

; ============================================================
level2:
    push rbp
    mov rbp, rsp
    sub rsp, 8

    say msg2, msg2l

    mov qword [rbp - 8], 2  ; depth = 2
    call level3

    say msg2r, msg2rl

    mov rsp, rbp
    pop rbp
    ret

; ============================================================
level3:
    push rbp
    mov rbp, rsp
    sub rsp, 8

    say msg3, msg3l

    mov qword [rbp - 8], 3  ; depth = 3
    ; ไม่มีการเรียก function เพิ่มเติม

    say msg3r, msg3rl

    mov rsp, rbp
    pop rbp
    ret
```

---

## 5. RET, RETF, RETN Instructions

### 5.1 ความแตกต่างระหว่าง RET/RETF/RETN

```nasm
; ============================================================
; ไฟล์: ret_variants.asm
; วัตถุประสงค์: แสดงความแตกต่างของ RET variants
; คอมไพล์: nasm -f elf64 ret_variants.asm -o rv.o && ld rv.o -o rv
; ============================================================
;
; RET   = Near return (pop RIP จาก stack)
;         ใช้สำหรับ procedure ใน segment เดียวกัน
;
; RETF  = Far return (pop RIP และ CS จาก stack)
;         ใช้สำหรับ far procedure calls (ต่าง segment)
;         ในโหมด 64-bit มักไม่ใช้
;
; RETN  = Near return (เหมือน RET ปกติ)
;         NASM ตีความว่าเป็น near return
;
; RET N = Near return + deallocate N bytes จาก stack
;         (stdcall convention ใช้แบบนี้)
;         เทียบเท่ากับ: pop RIP; add rsp, N
;
; ============================================================

section .data
    msg_clean   db "Stack-cleaning RET used", 0x0A, 0
    msg_clean_l equ $ - msg_clean

section .text
    global _start

_start:
    ; === RET N Example (callee stack cleanup) ===
    ; Push 2 arguments ลง stack (แบบ 32-bit cdecl จำลอง)
    push qword 20       ; arg2
    push qword 10       ; arg1
    call add_with_cleanup
    ; หลังจากนี้ RSP ควรจะ clean แล้ว (add_with_cleanup ทำความสะอาดเอง)
    ; RAX มีผลลัพธ์

    mov rax, 60
    xor rdi, rdi
    syscall

; ============================================================
; add_with_cleanup: บวกสองตัวเลขที่ส่งผ่าน stack
; Stack layout เมื่อเข้า function:
;   [rsp + 0]  = return address
;   [rsp + 8]  = arg1 (10)
;   [rsp + 16] = arg2 (20)
; ============================================================
add_with_cleanup:
    ; ไม่ต้องทำ prologue ถ้าไม่ใช้ local variables
    ; อ่าน arguments จาก stack
    mov rax, [rsp + 8]    ; arg1
    add rax, [rsp + 16]   ; รวม arg2

    ; RET 16 จะ:
    ; 1. pop return address จาก [rsp] เข้า RIP
    ; 2. add rsp, 16 (ลบ 2 arguments x 8 bytes)
    ret 16                ; return พร้อม cleanup 16 bytes

; ============================================================
; demonstrate_retn: แสดงว่า RETN เหมือน RET
; ============================================================
demonstrate_retn:
    push rbp
    mov rbp, rsp
    
    ; แสดงข้อความ
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_clean
    mov rdx, msg_clean_l
    syscall
    
    mov rsp, rbp
    pop rbp
    retn                  ; near return (เหมือน ret)
```

---

## 6. Stack Cleanup: Caller vs Callee

### 6.1 Calling Conventions

```nasm
; ============================================================
; ไฟล์: calling_conventions.asm
; วัตถุประสงค์: แสดง calling conventions ต่างๆ
; คอมไพล์: nasm -f elf64 calling_conventions.asm -o cc.o && ld cc.o -o cc
; ============================================================
;
; Calling Conventions หลักๆ:
;
; 1. System V AMD64 ABI (Linux 64-bit - ใช้บ่อยที่สุด)
;    - Arguments: RDI, RSI, RDX, RCX, R8, R9 (ถ้าเกิน → stack)
;    - Return: RAX (RDX สำหรับ 128-bit)
;    - Caller saves: RAX, RCX, RDX, RSI, RDI, R8-R11
;    - Callee saves: RBX, RBP, R12-R15
;    - Stack cleanup: CALLER
;
; 2. cdecl (32-bit C calling convention)
;    - Arguments: stack (right-to-left)
;    - Return: EAX
;    - Stack cleanup: CALLER (add esp, N หลัง call)
;
; 3. stdcall (Windows 32-bit API)
;    - Arguments: stack (right-to-left)
;    - Return: EAX
;    - Stack cleanup: CALLEE (ret N)
;
; 4. Microsoft x64 (Windows 64-bit)
;    - Arguments: RCX, RDX, R8, R9 (ถ้าเกิน → stack)
;    - Shadow space: 32 bytes ต้อง allocate ก่อน call
;    - Return: RAX
;    - Callee saves: RBX, RBP, RDI, RSI, R12-R15, XMM6-XMM15

section .data
    sum_msg     db "Sum: "
    sum_msg_l   equ $ - sum_msg
    newline     db 0x0A

section .text
    global _start

_start:
    ; === System V AMD64 ABI (caller cleanup - standard Linux) ===
    mov rdi, 5          ; arg1
    mov rsi, 10         ; arg2
    mov rdx, 15         ; arg3
    call sysv_add3      ; เรียก function
    ; RSP ไม่เปลี่ยนแปลง (registers ถูกใช้ ไม่ใช่ stack)
    ; RAX มีผลลัพธ์ (30)

    ; แสดงผล
    push rax
    mov rax, 1
    mov rdi, 1
    mov rsi, sum_msg
    mov rdx, sum_msg_l
    syscall
    pop rax
    call print_decimal_simple

    mov rax, 1
    mov rdi, 1
    lea rsi, [rel newline]
    mov rdx, 1
    syscall

    ; === Stack-based calling (จำลอง cdecl 64-bit) ===
    push qword 15       ; arg3
    push qword 10       ; arg2
    push qword 5        ; arg1
    call stack_add3     ; เรียก function
    add rsp, 24         ; CALLER cleanup (3 args x 8 bytes)

    ; Exit
    mov rax, 60
    xor rdi, rdi
    syscall

; ============================================================
; sysv_add3: บวกสามจำนวน (System V ABI)
; Input:  RDI=a, RSI=b, RDX=c
; Output: RAX = a + b + c
; ============================================================
sysv_add3:
    push rbp
    mov rbp, rsp
    ; ไม่ต้อง save/restore RDI, RSI, RDX (caller-saved)
    
    mov rax, rdi
    add rax, rsi
    add rax, rdx
    
    pop rbp     ; ไม่ต้อง mov rsp, rbp เพราะไม่ได้ sub rsp
    ret

; ============================================================
; stack_add3: บวกสามจำนวน (stack-based, callee saves nothing)
; Stack layout:
;   [rsp + 0]  = return address
;   [rsp + 8]  = arg1
;   [rsp + 16] = arg2
;   [rsp + 24] = arg3
; Output: RAX = a + b + c
; ============================================================
stack_add3:
    mov rax, [rsp + 8]   ; arg1
    add rax, [rsp + 16]  ; arg2
    add rax, [rsp + 24]  ; arg3
    ret                   ; CALLER จะทำ cleanup

; ============================================================
; print_decimal_simple: print RAX as decimal
; ============================================================
print_decimal_simple:
    push rbp
    mov rbp, rsp
    sub rsp, 32

    lea rdi, [rbp - 20]
    mov byte [rbp - 21], 0x0A   ; newline
    mov rcx, 0

    test rax, rax
    jnz .loop
    mov byte [rbp - 22], '0'
    mov rdi, rbp
    sub rdi, 22
    mov rcx, 1
    jmp .do_print

.loop:
    test rax, rax
    jz .do_print
    mov rdx, 0
    mov rbx, 10
    div rbx
    add dl, '0'
    dec rdi
    mov [rdi], dl
    inc rcx
    jmp .loop

.do_print:
    mov rax, 1
    mov rsi, rdi
    mov rdx, rcx
    mov rdi, 1
    syscall

    mov rsp, rbp
    pop rbp
    ret
```

---

## 7. Interrupt Routines vs Regular Procedures

### 7.1 ความแตกต่างสำคัญ

```nasm
; ============================================================
; ไฟล์: interrupt_vs_proc.asm
; วัตถุประสงค์: อธิบายความแตกต่าง ISR vs Procedure
; คอมไพล์: (conceptual - ต้องรันบน bare metal หรือ kernel)
; ============================================================
;
; Regular Procedure (CALL/RET):
; ┌─────────────────────────────────────────┐
; │ Caller เรียกโดยตั้งใจ (explicit call)  │
; │ Stack: return address เท่านั้น          │
; │ Return: RET (near) หรือ RETF (far)      │
; │ Flags: ไม่ถูกบันทึกอัตโนมัติ           │
; │ Context: caller กับ callee ใช้ CPU      │
; └─────────────────────────────────────────┘
;
; Interrupt Service Routine (ISR):
; ┌─────────────────────────────────────────┐
; │ ถูกเรียกโดย hardware/software/exception │
; │ Stack: CPU push FLAGS, CS, IP/RIP ให้   │
; │ Return: IRET/IRETD/IRETQ                │
; │ Flags: CPU บันทึก EFLAGS อัตโนมัติ     │
; │ IF bit ถูก clear อัตโนมัติ (no nesting) │
; └─────────────────────────────────────────┘

; ตัวอย่าง ISR structure (x86 32-bit, สำหรับ OS kernel)
; [BITS 32]
; section .text
;
; keyboard_isr:
;     ; CPU ได้ push: EFLAGS, CS, EIP ให้แล้ว
;     ; ต้อง save all registers ด้วยตัวเอง
;     pushad              ; push EAX, ECX, EDX, EBX, ESP, EBP, ESI, EDI
;     push ds
;     push es
;     push fs
;     push gs
;
;     ; ตั้งค่า data segment สำหรับ kernel
;     mov ax, 0x10        ; kernel data segment selector
;     mov ds, ax
;     mov es, ax
;
;     ; อ่านข้อมูลจาก keyboard port
;     in al, 0x60         ; อ่าน scancode จาก port 0x60
;
;     ; ส่ง EOI (End of Interrupt) ไปยัง PIC
;     mov al, 0x20
;     out 0x20, al        ; ส่ง EOI ไปยัง Master PIC
;
;     ; คืน registers
;     pop gs
;     pop fs
;     pop es
;     pop ds
;     popad               ; pop ทุก registers
;
;     iretd               ; Interrupt Return (pop EIP, CS, EFLAGS)
;                         ; ใน 64-bit ใช้ IRETQ (pop RIP, CS, RFLAGS, RSP, SS)

; ============================================================
; ตัวอย่าง software interrupt (INT instruction)
; ใน Linux ใช้ INT 0x80 สำหรับ 32-bit syscalls
; ============================================================

section .text
    global _start

_start:
    ; Linux syscall ผ่าน SYSCALL instruction (64-bit)
    mov rax, 1          ; syscall: write
    mov rdi, 1          ; fd: stdout
    lea rsi, [rel hello]
    mov rdx, hello_len
    syscall             ; invoke kernel (ผ่าน syscall instruction)

    ; Linux syscall ผ่าน INT 0x80 (32-bit mode)
    ; (เพื่อการศึกษา - ไม่แนะนำใน 64-bit mode)
    ; mov eax, 4          ; sys_write (32-bit number)
    ; mov ebx, 1          ; fd
    ; mov ecx, hello      ; buffer
    ; mov edx, hello_len  ; length
    ; int 0x80            ; trigger software interrupt

    mov rax, 60
    xor rdi, rdi
    syscall

section .data
    hello       db "Hello from syscall", 0x0A
    hello_len   equ $ - hello
```

---

## 8. Tail Call Optimization

### 8.1 ทำไม Tail Call Optimization ถึงสำคัญ

```nasm
; ============================================================
; ไฟล์: tail_call_opt.asm
; วัตถุประสงค์: แสดง tail call optimization ใน Assembly
; คอมไพล์: nasm -f elf64 tail_call_opt.asm -o tco.o && ld tco.o -o tco
; ============================================================
;
; Tail Call: เมื่อ call เป็นการกระทำสุดท้ายของ function
;
; ตัวอย่าง C:
;   int f(int n) { return g(n); }  ← tail call
;   int f(int n) { int x = g(n); return x + 1; }  ← NOT tail call
;
; โดยปกติ (ไม่มี TCO):
;   f: call g; ret       ← push/pop return address สองครั้ง
;
; ด้วย TCO:
;   f: jmp g             ← แทน call+ret ด้วย jmp
;      (ประหยัด stack frame!)
;
; TCO สำคัญมากสำหรับ recursive functions!
; ============================================================

section .data
    result_msg  db "Countdown result: ", 0
    result_len  equ $ - result_msg
    newline     db 0x0A

section .text
    global _start

_start:
    ; เรียก countdown แบบ non-TCO
    mov rdi, 5
    call countdown_no_tco
    ; แสดงผล
    push rax
    mov rax, 1
    mov rdi, 1
    mov rsi, result_msg
    mov rdx, result_len
    syscall
    pop rax
    call print_number_nl

    ; เรียก countdown แบบ TCO
    mov rdi, 5
    call countdown_tco
    push rax
    mov rax, 1
    mov rdi, 1
    mov rsi, result_msg
    mov rdx, result_len
    syscall
    pop rax
    call print_number_nl

    mov rax, 60
    xor rdi, rdi
    syscall

; ============================================================
; countdown_no_tco: นับถอยหลังแบบ recursive (ไม่มี TCO)
; Input: RDI = n
; Output: RAX = 0 (นับถอยหลังจนถึง 0)
; Stack usage: O(n) - อันตรายถ้า n ใหญ่มาก!
; ============================================================
countdown_no_tco:
    push rbp
    mov rbp, rsp

    ; base case: n == 0
    test rdi, rdi
    jz .base_case

    ; recursive case: countdown(n - 1)
    dec rdi
    call countdown_no_tco   ; ← เรียก recursive แต่ไม่ใช่ tail position จริงๆ
                             ; (จริงๆ แล้วมันเป็น tail call ในกรณีนี้
                             ; แต่เราจงใจไม่ optimize เพื่อแสดงให้เห็น)
    ; จะมี stack frame เพิ่มขึ้นเรื่อยๆ N ครั้ง

    pop rbp
    ret

.base_case:
    xor rax, rax    ; return 0
    pop rbp
    ret

; ============================================================
; countdown_tco: นับถอยหลังแบบ Tail Call Optimized
; Input: RDI = n
; Output: RAX = 0
; Stack usage: O(1) - ไม่ว่า n จะใหญ่แค่ไหน!
; ============================================================
countdown_tco:
    ; base case: n == 0
    test rdi, rdi
    jz .done

    ; recursive case: ใช้ JMP แทน CALL (TCO)
    dec rdi
    jmp countdown_tco   ; ← JMP แทน CALL+RET = ประหยัด stack!

.done:
    xor rax, rax    ; return 0
    ret

; ============================================================
; Tail Call ไปยัง function อื่น
; ============================================================
function_a:
    push rbp
    mov rbp, rsp
    ; ... ทำบางอย่าง ...
    mov rdi, rdi        ; ส่ง argument ต่อ

    ; แทน:
    ; call function_b
    ; pop rbp
    ; ret
    ;
    ; ใช้:
    pop rbp             ; คืน frame ก่อน
    jmp function_b      ; tail call optimization!

function_b:
    push rbp
    mov rbp, rsp
    ; ... ทำบางอย่าง ...
    xor rax, rax
    pop rbp
    ret

; print_number_nl: print RAX as decimal + newline
print_number_nl:
    push rbp
    mov rbp, rsp
    sub rsp, 48

    lea r9, [rbp - 30]      ; buffer end
    mov byte [r9], 0x0A     ; newline
    mov rcx, 1              ; length starts at 1 (for newline)

    test rax, rax
    jnz .pn_loop
    dec r9
    mov byte [r9], '0'
    inc rcx
    jmp .pn_print

.pn_loop:
    test rax, rax
    jz .pn_print
    mov rdx, 0
    mov rbx, 10
    div rbx
    add dl, '0'
    dec r9
    mov [r9], dl
    inc rcx
    jmp .pn_loop

.pn_print:
    mov rax, 1
    mov rdi, 1
    mov rsi, r9
    mov rdx, rcx
    syscall

    mov rsp, rbp
    pop rbp
    ret
```

---

## 9. Recursive Procedures

### 9.1 Recursive Factorial

```nasm
; ============================================================
; ไฟล์: recursive_factorial.asm
; วัตถุประสงค์: คำนวณ factorial แบบ recursive
; คอมไพล์: nasm -f elf64 recursive_factorial.asm -o fact.o && ld fact.o -o fact
; ============================================================
;
; factorial(n) = n * factorial(n-1)
; factorial(0) = 1
;
; Stack trace สำหรับ factorial(4):
; _start calls factorial(4)
;   factorial(4) calls factorial(3)
;     factorial(3) calls factorial(2)
;       factorial(2) calls factorial(1)
;         factorial(1) calls factorial(0)
;         factorial(0) returns 1
;       factorial(1) returns 1*1 = 1
;     factorial(2) returns 2*1 = 2
;   factorial(3) returns 3*2 = 6
; factorial(4) returns 4*6 = 24

section .data
    msg_fact    db "factorial(", 0
    msg_fact_l  equ $ - msg_fact
    msg_eq      db ") = ", 0
    msg_eq_l    equ $ - msg_eq
    newline     db 0x0A

section .text
    global _start

_start:
    ; คำนวณ factorial สำหรับ n = 0 ถึง 12
    mov r15, 0      ; loop counter

.loop:
    cmp r15, 13
    jge .done

    ; พิมพ์ "factorial("
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_fact
    mov rdx, msg_fact_l
    syscall

    ; พิมพ์ n
    mov rax, r15
    call print_uint

    ; พิมพ์ ") = "
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_eq
    mov rdx, msg_eq_l
    syscall

    ; คำนวณ factorial(n)
    mov rdi, r15
    call factorial

    ; พิมพ์ผลลัพธ์
    call print_uint

    ; พิมพ์ newline
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel newline]
    mov rdx, 1
    syscall

    inc r15
    jmp .loop

.done:
    mov rax, 60
    xor rdi, rdi
    syscall

; ============================================================
; factorial: คำนวณ n! แบบ recursive
; Input:  RDI = n (non-negative integer)
; Output: RAX = n!
; Registers saved: RBP, RBX (callee-saved)
; ============================================================
factorial:
    push rbp
    mov rbp, rsp
    push rbx            ; save rbx (callee-saved)

    mov rbx, rdi        ; เก็บ n ไว้ใน RBX (จะคงอยู่ข้าม recursive call)

    ; Base case: if n <= 1, return 1
    cmp rbx, 1
    jle .base_case

    ; Recursive case: return n * factorial(n-1)
    mov rdi, rbx
    dec rdi             ; RDI = n - 1
    call factorial      ; RAX = factorial(n-1)

    ; RAX = n * factorial(n-1)
    imul rax, rbx       ; RAX = n * factorial(n-1)
    jmp .done

.base_case:
    mov rax, 1          ; return 1

.done:
    pop rbx             ; restore rbx
    pop rbp
    ret

; ============================================================
; print_uint: พิมพ์ unsigned integer ใน RAX
; ============================================================
print_uint:
    push rbp
    mov rbp, rsp
    sub rsp, 32

    lea rdi, [rbp - 20]
    mov rcx, 0

    ; กรณี 0
    test rax, rax
    jnz .pu_loop
    dec rdi
    mov byte [rdi], '0'
    inc rcx
    jmp .pu_print

.pu_loop:
    test rax, rax
    jz .pu_print
    mov rdx, 0
    push rdi
    mov rdi, 10
    div rdi
    pop rdi
    add dl, '0'
    dec rdi
    mov [rdi], dl
    inc rcx
    jmp .pu_loop

.pu_print:
    push rdi
    mov rax, 1
    mov rsi, rdi
    mov rdx, rcx
    mov rdi, 1
    syscall
    pop rdi

    mov rsp, rbp
    pop rbp
    ret
```

### 9.2 Ackermann Function

```nasm
; ============================================================
; ไฟล์: ackermann.asm
; วัตถุประสงค์: คำนวณ Ackermann function (grows extremely fast)
; คอมไพล์: nasm -f elf64 ackermann.asm -o ack.o && ld ack.o -o ack
; ============================================================
;
; Ackermann function:
;   A(0, n) = n + 1
;   A(m, 0) = A(m-1, 1)       เมื่อ m > 0
;   A(m, n) = A(m-1, A(m, n-1)) เมื่อ m > 0, n > 0
;
; ค่า Ackermann เติบโตเร็วมาก:
;   A(0,0)=1, A(1,1)=3, A(2,2)=7, A(3,3)=61, A(4,1)=65533
;   อย่าลอง A(5,5) หรือ A(4,4)! (stack overflow / เวลานานมาก)
;
; WARNING: ใช้เฉพาะ m <= 3 และ n เล็กๆ เท่านั้น!

section .data
    msg_ack     db "A("
    msg_ack_l   equ $ - msg_ack
    msg_comma   db ","
    msg_eq2     db ") = ", 0
    msg_eq2_l   equ $ - msg_eq2
    nl          db 0x0A
    stack_warn  db "WARNING: Use small values only!", 0x0A
    stack_warn_l equ $ - stack_warn

section .text
    global _start

_start:
    ; แสดง warning
    mov rax, 1
    mov rdi, 1
    mov rsi, stack_warn
    mov rdx, stack_warn_l
    syscall

    ; คำนวณ A(m, n) สำหรับค่าเล็กๆ
    ; A(0,0) = 1
    mov rdi, 0
    mov rsi, 0
    call ackermann_print

    ; A(1,1) = 3
    mov rdi, 1
    mov rsi, 1
    call ackermann_print

    ; A(2,2) = 7
    mov rdi, 2
    mov rsi, 2
    call ackermann_print

    ; A(3,3) = 61
    mov rdi, 3
    mov rsi, 3
    call ackermann_print

    ; A(3,4) = 125
    mov rdi, 3
    mov rsi, 4
    call ackermann_print

    mov rax, 60
    xor rdi, rdi
    syscall

; ============================================================
; ackermann_print: wrapper ที่พิมพ์ผลลัพธ์
; ============================================================
ackermann_print:
    push rbp
    mov rbp, rsp
    push rbx
    push r12

    mov rbx, rdi        ; save m
    mov r12, rsi        ; save n

    ; พิมพ์ "A(m,n) = "
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_ack
    mov rdx, msg_ack_l
    syscall

    mov rax, rbx
    call print_u64

    mov rax, 1
    mov rdi, 1
    lea rsi, [rel msg_comma]
    mov rdx, 1
    syscall

    mov rax, r12
    call print_u64

    mov rax, 1
    mov rdi, 1
    mov rsi, msg_eq2
    mov rdx, msg_eq2_l
    syscall

    ; คำนวณ A(m, n)
    mov rdi, rbx
    mov rsi, r12
    call ackermann

    ; พิมพ์ผลลัพธ์
    call print_u64

    mov rax, 1
    mov rdi, 1
    lea rsi, [rel nl]
    mov rdx, 1
    syscall

    pop r12
    pop rbx
    pop rbp
    ret

; ============================================================
; ackermann: คำนวณ Ackermann function
; Input:  RDI = m, RSI = n
; Output: RAX = A(m, n)
; ============================================================
ackermann:
    push rbp
    mov rbp, rsp
    push rbx
    push r12

    mov rbx, rdi        ; m
    mov r12, rsi        ; n

    ; Case 1: m == 0, return n + 1
    test rbx, rbx
    jnz .m_nonzero
    lea rax, [r12 + 1]
    jmp .ack_done

.m_nonzero:
    ; Case 2: n == 0, return A(m-1, 1)
    test r12, r12
    jnz .both_nonzero
    lea rdi, [rbx - 1]  ; m - 1
    mov rsi, 1          ; 1
    call ackermann
    jmp .ack_done

.both_nonzero:
    ; Case 3: return A(m-1, A(m, n-1))
    ; ขั้นตอน 1: คำนวณ A(m, n-1)
    mov rdi, rbx        ; m
    lea rsi, [r12 - 1]  ; n - 1
    call ackermann

    ; ขั้นตอน 2: คำนวณ A(m-1, A(m, n-1))
    lea rdi, [rbx - 1]  ; m - 1
    mov rsi, rax        ; A(m, n-1)
    call ackermann

.ack_done:
    pop r12
    pop rbx
    pop rbp
    ret

; print_u64: พิมพ์ RAX as unsigned decimal
print_u64:
    push rbp
    mov rbp, rsp
    sub rsp, 32

    ; ใช้ stack เก็บ digits
    mov rcx, 0
    test rax, rax
    jnz .p64_loop

    ; เป็น 0
    push rax
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel zero_char]
    mov rdx, 1
    syscall
    pop rax
    jmp .p64_done

.p64_push_loop:
    ; push digits onto stack
    mov rbx, 0
.p64_loop:
    test rax, rax
    jz .p64_print_all

    mov rdx, 0
    mov rdi, 10
    push rdi
    div qword [rsp]
    pop rdi

    push rdx            ; push digit
    inc rcx
    jmp .p64_loop

.p64_print_all:
    ; pop and print digits
    test rcx, rcx
    jz .p64_done

.p64_pop_loop:
    pop rax
    add al, '0'
    mov [rbp - 1], al
    push rax
    mov rax, 1
    mov rdi, 1
    lea rsi, [rbp - 1]
    mov rdx, 1
    syscall
    pop rax
    dec rcx
    jnz .p64_pop_loop

.p64_done:
    mov rsp, rbp
    pop rbp
    ret

section .data
    zero_char   db '0'
```

---

## 10. Function Pointer Table

```nasm
; ============================================================
; ไฟล์: func_pointer_table.asm
; วัตถุประสงค์: สร้างและใช้งาน function pointer table
; คอมไพล์: nasm -f elf64 func_pointer_table.asm -o fpt.o && ld fpt.o -o fpt
; ============================================================
;
; Function pointer table มีประโยชน์สำหรับ:
; - Dispatch tables (แทน if-else chain)
; - Virtual function tables (vtable ใน C++)
; - Callback systems
; - Plugin architectures

section .data
    ; Function pointer table (jump table)
    ; แต่ละ entry คือ 8-byte address ของ function
    op_table:
        dq add_func         ; index 0: addition
        dq sub_func         ; index 1: subtraction
        dq mul_func         ; index 2: multiplication
        dq div_func         ; index 3: division
    OP_TABLE_SIZE equ ($ - op_table) / 8  ; จำนวน entries

    ; Messages
    results_hdr db "Operation Results:", 0x0A
    results_hdr_l equ $ - results_hdr
    op_names:
        dq op_add_str
        dq op_sub_str
        dq op_mul_str
        dq op_div_str
    op_add_str  db "ADD: ", 0
    op_sub_str  db "SUB: ", 0
    op_mul_str  db "MUL: ", 0
    op_div_str  db "DIV: ", 0
    op_str_len  equ 5
    nl2         db 0x0A

section .text
    global _start

_start:
    ; แสดง header
    mov rax, 1
    mov rdi, 1
    mov rsi, results_hdr
    mov rdx, results_hdr_l
    syscall

    ; ทดสอบทุก operation ด้วย a=20, b=4
    mov r12, 0      ; index

.dispatch_loop:
    cmp r12, OP_TABLE_SIZE
    jge .exit_loop

    ; พิมพ์ชื่อ operation
    mov rax, 1
    mov rdi, 1
    mov rsi, [op_names + r12 * 8]
    mov rdx, op_str_len
    syscall

    ; เรียก function ผ่าน table
    mov rdi, 20     ; a = 20
    mov rsi, 4      ; b = 4
    call [op_table + r12 * 8]   ; indirect call ผ่าน table

    ; พิมพ์ผลลัพธ์
    call print_signed

    mov rax, 1
    mov rdi, 1
    lea rsi, [rel nl2]
    mov rdx, 1
    syscall

    inc r12
    jmp .dispatch_loop

.exit_loop:
    ; ตัวอย่างเพิ่มเติม: เรียกผ่าน computed index
    ; สมมุติว่าได้รับ operation code จาก input
    mov rcx, 2          ; operation code 2 = multiply
    cmp rcx, OP_TABLE_SIZE
    jge .invalid_op

    ; Bounds check ก่อนเสมอ!
    mov rdi, 7
    mov rsi, 6
    call [op_table + rcx * 8]
    ; RAX = 42 (7 * 6)

.invalid_op:
    mov rax, 60
    xor rdi, rdi
    syscall

; ============================================================
; Operations (ทุก function ใช้ RDI=a, RSI=b, return RAX)
; ============================================================

add_func:
    mov rax, rdi
    add rax, rsi
    ret

sub_func:
    mov rax, rdi
    sub rax, rsi
    ret

mul_func:
    mov rax, rdi
    imul rax, rsi
    ret

div_func:
    ; ตรวจสอบ division by zero
    test rsi, rsi
    jz .div_zero
    mov rax, rdi
    cqo                 ; sign-extend RAX into RDX:RAX
    idiv rsi
    ret
.div_zero:
    mov rax, 0          ; return 0 on division by zero
    ret

; print_signed: พิมพ์ RAX as signed decimal
print_signed:
    push rbp
    mov rbp, rsp
    sub rsp, 32

    test rax, rax
    jns .ps_positive

    ; negative
    push rax
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel minus]
    mov rdx, 1
    syscall
    pop rax
    neg rax

.ps_positive:
    lea r9, [rbp - 20]
    mov rcx, 0

    test rax, rax
    jnz .ps_loop
    dec r9
    mov byte [r9], '0'
    inc rcx
    jmp .ps_print

.ps_loop:
    test rax, rax
    jz .ps_print
    mov rdx, 0
    push r9
    mov r9, 10
    div r9
    pop r9
    add dl, '0'
    dec r9
    mov [r9], dl
    inc rcx
    jmp .ps_loop

.ps_print:
    mov rax, 1
    mov rdi, 1
    mov rsi, r9
    mov rdx, rcx
    syscall

    mov rsp, rbp
    pop rbp
    ret

section .data
    minus   db "-"
```

---

## 11. Return Address Tampering (Security)

### 11.1 Stack Buffer Overflow และ Return Address Overwrite

```nasm
; ============================================================
; ไฟล์: return_addr_security.asm
; วัตถุประสงค์: อธิบาย security implications ของ return address
; คอมไพล์: nasm -f elf64 return_addr_security.asm -o ras.o && ld ras.o -o ras
; ============================================================
;
; ⚠️ WARNING: เนื้อหานี้เพื่อการศึกษา security เท่านั้น
;             อย่านำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาต!
;
; Stack Buffer Overflow คืออะไร?
; ───────────────────────────────
; ถ้า function มี buffer ใน stack และไม่มีการ bounds checking
; attacker สามารถ overwrite return address ได้!
;
; ตัวอย่าง stack layout ที่มีช่องโหว่:
; ┌─────────────────┐  higher address
; │  ...caller...   │
; │  return address │  ← ถ้า overflow buffer จะ overwrite ที่นี่!
; │  saved rbp      │  ← overwrite ก่อน
; │  buffer[0..N]   │  ← overflow เริ่มจากที่นี่ ↑
; └─────────────────┘  lower address
;
; การป้องกัน (Mitigations):
; 1. Stack Canaries (GCC -fstack-protector)
;    - วางค่าพิเศษ (canary) ระหว่าง buffer กับ saved rbp
;    - ตรวจสอบก่อน return
;
; 2. ASLR (Address Space Layout Randomization)
;    - OS randomize addresses ทุกครั้งที่รัน
;    - ทำให้ attacker ไม่รู้ address ที่แน่นอน
;
; 3. NX bit / DEP (Data Execution Prevention)
;    - ป้องกันการ execute code ใน stack/heap
;
; 4. CFI (Control Flow Integrity)
;    - ตรวจสอบว่า indirect calls ไปยัง valid targets เท่านั้น

section .data
    safe_msg    db "Safe function called correctly", 0x0A
    safe_msg_l  equ $ - safe_msg
    canary_ok   db "Stack canary: OK", 0x0A
    canary_ok_l equ $ - canary_ok
    canary_bad  db "STACK CORRUPTION DETECTED!", 0x0A
    canary_bad_l equ $ - canary_bad

section .text
    global _start

_start:
    call safe_function_with_canary
    mov rax, 60
    xor rdi, rdi
    syscall

; ============================================================
; safe_function_with_canary: แสดงการ implement manual stack canary
; ============================================================
safe_function_with_canary:
    push rbp
    mov rbp, rsp
    sub rsp, 48         ; จอง space: 32 bytes buffer + 8 bytes canary + padding

    ; === วาง canary ===
    ; ในระบบจริง canary ค่าจะ random ทุกครั้ง
    ; ที่นี่ใช้ค่า constant เพื่อการศึกษา
    mov rax, 0xDEADBEEFCAFEBABE  ; canary value
    mov [rbp - 8], rax            ; วาง canary ใต้ saved rbp

    ; === ใช้ buffer อย่างปลอดภัย ===
    ; buffer อยู่ที่ [rbp - 40] ถึง [rbp - 9]
    ; (32 bytes)
    lea rdi, [rbp - 40]     ; pointer ไปยัง buffer
    ; ใช้ buffer โดยไม่ overflow...
    mov qword [rbp - 40], 0x4142434445464748  ; "HGFEDCBA"

    ; แสดงข้อความ
    mov rax, 1
    mov rdi, 1
    mov rsi, safe_msg
    mov rdx, safe_msg_l
    syscall

    ; === ตรวจสอบ canary ก่อน return ===
    mov rax, [rbp - 8]                  ; อ่าน canary
    mov rcx, 0xDEADBEEFCAFEBABE        ; expected canary
    cmp rax, rcx
    jne .canary_corrupted

    ; canary ยังดีอยู่
    mov rax, 1
    mov rdi, 1
    mov rsi, canary_ok
    mov rdx, canary_ok_l
    syscall
    jmp .return

.canary_corrupted:
    ; Stack corrupted! อาจโดน attack
    mov rax, 1
    mov rdi, 1
    mov rsi, canary_bad
    mov rdx, canary_bad_l
    syscall
    ; ควร terminate โปรแกรมทันที
    mov rax, 60
    mov rdi, 1      ; exit code 1 (error)
    syscall

.return:
    mov rsp, rbp
    pop rbp
    ret

; ============================================================
; ตัวอย่างการตรวจสอบ return address (ROP detection concept)
; ============================================================
validate_return_addr:
    push rbp
    mov rbp, rsp

    ; อ่าน return address
    mov rax, [rbp + 8]  ; return address อยู่ที่ [rbp + 8]

    ; ตรวจสอบว่า return address อยู่ใน valid code range
    ; (ในระบบจริงต้องรู้ code section boundaries)
    lea rcx, [rel _start]           ; start of code
    cmp rax, rcx
    jb .suspicious                   ; ถ้าต่ำกว่า code start → suspicious

    ; อนุมัติ (simplified check)
    xor rax, rax    ; return 0 = OK

    pop rbp
    ret

.suspicious:
    mov rax, 1      ; return 1 = suspicious
    pop rbp
    ret
```

---

## 12. Near vs Far Procedures (16/32-bit Context)

```nasm
; ============================================================
; ไฟล์: near_far_procedures.asm
; วัตถุประสงค์: อธิบาย Near vs Far procedures ใน real-mode context
; คอมไพล์: nasm -f bin near_far_procedures.asm -o boot.bin
; หมายเหตุ: ตัวอย่างนี้เป็น bootloader/real-mode code
;            รันบน QEMU: qemu-system-x86_64 -drive format=raw,file=boot.bin
; ============================================================
;
; ใน Real Mode (16-bit):
; - CALL near: push IP, jmp
; - CALL far: push CS, push IP, jmp ไปยัง segment อื่น
; - RET:  pop IP
; - RETF: pop IP, pop CS
;
; ใน Protected/Long Mode (32/64-bit):
; - near call: ใช้บ่อยที่สุด
; - far call: ใช้สำหรับ privilege level changes (ring 0 → ring 3)
;   หรือ task switching

[BITS 16]
[ORG 0x7C00]            ; BIOS loads bootloader ที่ address นี้

start:
    ; ตั้งค่า segment registers
    cli                 ; ปิด interrupts ชั่วคราว
    xor ax, ax
    mov ds, ax          ; DS = 0
    mov es, ax
    mov ss, ax
    mov sp, 0x7C00      ; stack ก่อน bootloader
    sti                 ; เปิด interrupts

    ; Near call (ภายใน segment เดียวกัน)
    call near_proc      ; push IP, jmp near_proc

    ; Far call (ข้าม segment)
    ; format: CALL segment:offset
    ; ที่นี่จะจำลองด้วย push cs + call
    push cs             ; push current CS
    call far_proc       ; จำลอง far call

    ; หยุดที่นี่
    jmp $               ; infinite loop

; ============================================================
near_proc:
    ; บน stack: [sp] = return IP
    ; แสดงอักษร 'N' ผ่าน BIOS INT 10h
    mov ah, 0x0E        ; teletype output
    mov al, 'N'
    int 0x10
    ret                 ; near return: pop IP

; ============================================================
far_proc:
    ; บน stack: [sp] = return IP, [sp+2] = return CS
    mov ah, 0x0E
    mov al, 'F'
    int 0x10
    ; retf               ; far return: pop IP, pop CS
    ; (ใช้ retf ถ้าเป็น far call จริงๆ)
    ret                 ; ใช้ near ret เพราะจำลอง

; Padding และ boot signature
times 510 - ($ - $$) db 0
dw 0xAA55               ; boot signature
```

---

## 13. การเรียก System Functions และ Library Calls

```nasm
; ============================================================
; ไฟล์: calling_c_functions.asm
; วัตถุประสงค์: เรียก C library functions จาก Assembly
; คอมไพล์: nasm -f elf64 calling_c_functions.asm -o ccf.o
;           gcc -no-pie ccf.o -o ccf
; หมายเหตุ: ต้อง link กับ C runtime library
; ============================================================

section .data
    fmt_hello   db "Hello from C printf!", 0x0A, 0
    fmt_num     db "Number: %d", 0x0A, 0
    fmt_str     db "String: %s", 0x0A, 0
    test_str    db "assembly world", 0

section .text
    global main         ; ใช้ main แทน _start เพื่อ link กับ C runtime
    extern printf       ; declare external C function
    extern malloc
    extern free

main:
    push rbp
    mov rbp, rsp
    sub rsp, 16         ; align stack (System V ABI ต้องการ 16-byte alignment)
                        ; ก่อน CALL stack ต้องอยู่ที่ 16n - 8
                        ; หลัง push rbp: misaligned, ดังนั้น sub rsp, 16
                        ; เพื่อให้ aligned ก่อน call printf

    ; เรียก printf("Hello from C printf!\n")
    lea rdi, [rel fmt_hello]    ; format string (arg1)
    xor rax, rax                ; 0 floating point args
    call printf

    ; เรียก printf("Number: %d\n", 42)
    lea rdi, [rel fmt_num]      ; format string
    mov rsi, 42                 ; integer argument
    xor rax, rax
    call printf

    ; เรียก printf("String: %s\n", test_str)
    lea rdi, [rel fmt_str]
    lea rsi, [rel test_str]
    xor rax, rax
    call printf

    ; เรียก malloc(64)
    mov rdi, 64         ; size = 64 bytes
    call malloc
    ; RAX = pointer ไปยัง allocated memory (หรือ NULL ถ้า fail)
    test rax, rax
    jz .malloc_failed

    ; ใช้ memory
    mov qword [rax], 0x1234567890ABCDEF
    mov rbx, rax        ; เก็บ pointer

    ; เรียก free(ptr)
    mov rdi, rbx
    call free

.malloc_failed:
    ; คืนค่า 0 จาก main
    xor eax, eax
    mov rsp, rbp
    pop rbp
    ret
```

---

## 14. Advanced: CALL/RET Matching และ Stack Discipline

```nasm
; ============================================================
; ไฟล์: call_ret_discipline.asm
; วัตถุประสงค์: แสดง CALL/RET matching ที่ถูกต้องและผิดพลาด
; คอมไพล์: nasm -f elf64 call_ret_discipline.asm -o crd.o && ld crd.o -o crd
; ============================================================
;
; กฎสำคัญ: CALL และ RET ต้อง match กันเสมอ!
;
; ❌ ผิด: กระโดดออกจาก procedure โดยไม่ ret
;   func:
;       jmp somewhere   ← return address ยังค้างใน stack!
;
; ❌ ผิด: RET โดยไม่มี CALL
;   _start:
;       ret             ← ไม่รู้จะ return ไปที่ไหน!
;
; ❌ ผิด: stack ไม่ balance ก่อน RET
;   func:
;       push rax        ← push 1 ครั้ง
;       ; ลืม pop      ← return address ผิด!
;       ret
;
; ✓ ถูก: CALL/RET balance ครบถ้วน
;   func:
;       push rbp
;       mov rbp, rsp
;       push rax        ← push
;       pop rax         ← pop (balance)
;       pop rbp
;       ret

section .data
    ok_msg  db "Stack discipline: OK", 0x0A
    ok_len  equ $ - ok_msg
    bad_msg db "Stack discipline: BAD (program will crash)", 0x0A
    bad_len equ $ - bad_msg

section .text
    global _start

_start:
    call well_behaved_func
    mov rax, 60
    xor rdi, rdi
    syscall

; ============================================================
; well_behaved_func: ตัวอย่าง function ที่รักษา stack discipline
; ============================================================
well_behaved_func:
    push rbp
    mov rbp, rsp
    sub rsp, 32             ; local variables

    push rbx                ; save callee-saved register
    push r12
    push r13

    ; ทำงานบางอย่าง
    mov rbx, 100
    mov r12, 200
    mov r13, rbx
    add r13, r12            ; r13 = 300

    ; ตรวจสอบ stack pointer
    mov rax, rsp
    lea rcx, [rbp - 32 - 24]    ; rbp - 32 (locals) - 24 (3 pushes)
    cmp rax, rcx
    jne .stack_corrupted

    ; แสดงว่า OK
    mov rax, 1
    mov rdi, 1
    mov rsi, ok_msg
    mov rdx, ok_len
    syscall

    pop r13                 ; restore (reverse order)
    pop r12
    pop rbx

    ; ตรวจสอบอีกครั้ง
    mov rax, rsp
    lea rcx, [rbp - 32]     ; ควรเป็น rbp - 32 หลัง pop 3 ครั้ง
    cmp rax, rcx
    jne .stack_corrupted

    mov rsp, rbp
    pop rbp
    ret

.stack_corrupted:
    mov rax, 1
    mov rdi, 1
    mov rsi, bad_msg
    mov rdx, bad_len
    syscall

    ; Force exit เพื่อหลีกเลี่ยง undefined behavior
    mov rax, 60
    mov rdi, 1
    syscall
```

---

## 15. Complete Application: Calculator with Procedures

```nasm
; ============================================================
; ไฟล์: calc_with_procs.asm
; วัตถุประสงค์: เครื่องคิดเลขที่ใช้ procedures อย่างครบถ้วน
; คอมไพล์: nasm -f elf64 calc_with_procs.asm -o calc.o && ld calc.o -o calc
; ============================================================

section .data
    ; Menu
    menu_str    db 0x0A, "=== Assembly Calculator ===", 0x0A
                db "1. Add", 0x0A
                db "2. Subtract", 0x0A
                db "3. Multiply", 0x0A
                db "4. Divide", 0x0A
                db "5. Power (x^n)", 0x0A
                db "6. GCD", 0x0A
                db "0. Exit", 0x0A
                db "Choice: ", 0
    menu_len    equ $ - menu_str
    prompt_a    db "Enter A: ", 0
    prompt_a_l  equ $ - prompt_a
    prompt_b    db "Enter B: ", 0
    prompt_b_l  equ $ - prompt_b
    result_hdr  db "Result: ", 0
    result_hdr_l equ $ - result_hdr
    div0_err    db "Error: Division by zero!", 0x0A, 0
    div0_err_l  equ $ - div0_err
    nl3         db 0x0A
    invalid_msg db "Invalid choice!", 0x0A, 0
    invalid_l   equ $ - invalid_msg

section .bss
    input_buf   resb 32     ; buffer สำหรับรับ input

section .text
    global _start

_start:
    ; Main loop
.main_loop:
    ; แสดง menu
    call print_menu

    ; รับ choice
    call read_number    ; RAX = choice
    mov r15, rax        ; เก็บ choice

    ; ตรวจสอบ choice
    cmp r15, 0
    je .exit_program
    cmp r15, 6
    jg .invalid_choice

    ; รับ A
    mov rax, 1
    mov rdi, 1
    mov rsi, prompt_a
    mov rdx, prompt_a_l
    syscall
    call read_number
    mov r13, rax        ; A

    ; รับ B
    mov rax, 1
    mov rdi, 1
    mov rsi, prompt_b
    mov rdx, prompt_b_l
    syscall
    call read_number
    mov r14, rax        ; B

    ; Dispatch ตาม choice
    cmp r15, 1
    je .do_add
    cmp r15, 2
    je .do_sub
    cmp r15, 3
    je .do_mul
    cmp r15, 4
    je .do_div
    cmp r15, 5
    je .do_pow
    cmp r15, 6
    je .do_gcd

.do_add:
    mov rdi, r13
    mov rsi, r14
    call calc_add
    jmp .show_result

.do_sub:
    mov rdi, r13
    mov rsi, r14
    call calc_sub
    jmp .show_result

.do_mul:
    mov rdi, r13
    mov rsi, r14
    call calc_mul
    jmp .show_result

.do_div:
    test r14, r14
    jz .div_by_zero
    mov rdi, r13
    mov rsi, r14
    call calc_div
    jmp .show_result

.div_by_zero:
    mov rax, 1
    mov rdi, 1
    mov rsi, div0_err
    mov rdx, div0_err_l
    syscall
    jmp .main_loop

.do_pow:
    mov rdi, r13
    mov rsi, r14
    call calc_power
    jmp .show_result

.do_gcd:
    mov rdi, r13
    mov rsi, r14
    call calc_gcd
    jmp .show_result

.show_result:
    push rax
    mov rax, 1
    mov rdi, 1
    mov rsi, result_hdr
    mov rdx, result_hdr_l
    syscall
    pop rax
    call print_signed2

    mov rax, 1
    mov rdi, 1
    lea rsi, [rel nl3]
    mov rdx, 1
    syscall

    jmp .main_loop

.invalid_choice:
    mov rax, 1
    mov rdi, 1
    mov rsi, invalid_msg
    mov rdx, invalid_l
    syscall
    jmp .main_loop

.exit_program:
    mov rax, 60
    xor rdi, rdi
    syscall

; ============================================================
; print_menu
; ============================================================
print_menu:
    mov rax, 1
    mov rdi, 1
    mov rsi, menu_str
    mov rdx, menu_len
    syscall
    ret

; ============================================================
; calc_add: RAX = RDI + RSI
; ============================================================
calc_add:
    mov rax, rdi
    add rax, rsi
    ret

; ============================================================
; calc_sub: RAX = RDI - RSI
; ============================================================
calc_sub:
    mov rax, rdi
    sub rax, rsi
    ret

; ============================================================
; calc_mul: RAX = RDI * RSI
; ============================================================
calc_mul:
    mov rax, rdi
    imul rax, rsi
    ret

; ============================================================
; calc_div: RAX = RDI / RSI (integer division)
; ============================================================
calc_div:
    mov rax, rdi
    cqo
    idiv rsi
    ret

; ============================================================
; calc_power: RAX = RDI ^ RSI (iterative)
; Input: RDI = base, RSI = exponent (>= 0)
; ============================================================
calc_power:
    push rbp
    mov rbp, rsp
    push rbx
    push r12

    mov rbx, rdi        ; base
    mov r12, rsi        ; exponent

    ; edge cases
    test r12, r12
    jz .pow_zero_exp    ; base^0 = 1

    mov rax, 1          ; result = 1

.pow_loop:
    test r12, r12
    jz .pow_done
    imul rax, rbx       ; result *= base
    dec r12
    jmp .pow_loop

.pow_zero_exp:
    mov rax, 1
    jmp .pow_done

.pow_done:
    pop r12
    pop rbx
    pop rbp
    ret

; ============================================================
; calc_gcd: RAX = gcd(RDI, RSI) - Euclidean algorithm
; ============================================================
calc_gcd:
    push rbp
    mov rbp, rsp

    ; ใช้ค่า absolute
    mov rax, rdi
    test rax, rax
    jns .gcd_a_ok
    neg rax
.gcd_a_ok:
    mov rcx, rsi
    test rcx, rcx
    jns .gcd_b_ok
    neg rcx
.gcd_b_ok:

.gcd_loop:
    test rcx, rcx
    jz .gcd_done
    mov rdx, 0
    div rcx             ; RAX = RAX / RCX, RDX = remainder
    mov rax, rcx
    mov rcx, rdx
    jmp .gcd_loop

.gcd_done:
    pop rbp
    ret

; ============================================================
; read_number: อ่าน integer จาก stdin
; Output: RAX = number (signed)
; ============================================================
read_number:
    push rbp
    mov rbp, rsp
    push rbx
    push r12

    ; อ่าน input
    mov rax, 0          ; syscall: read
    mov rdi, 0          ; fd: stdin
    lea rsi, [rel input_buf]
    mov rdx, 31         ; max bytes
    syscall

    ; แปลง string เป็น integer
    lea rsi, [rel input_buf]
    xor rax, rax        ; result = 0
    xor rbx, rbx        ; negative flag = 0

    ; ตรวจสอบ sign
    movzx rcx, byte [rsi]
    cmp rcx, '-'
    jne .rn_parse
    inc rsi
    mov rbx, 1          ; negative

.rn_parse:
    movzx rcx, byte [rsi]
    cmp rcx, '0'
    jb .rn_done
    cmp rcx, '9'
    ja .rn_done

    sub rcx, '0'
    imul rax, 10
    add rax, rcx
    inc rsi
    jmp .rn_parse

.rn_done:
    test rbx, rbx
    jz .rn_positive
    neg rax

.rn_positive:
    pop r12
    pop rbx
    pop rbp
    ret

; ============================================================
; print_signed2: พิมพ์ RAX เป็น signed decimal
; ============================================================
print_signed2:
    push rbp
    mov rbp, rsp
    sub rsp, 48

    test rax, rax
    jns .ps2_pos

    push rax
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel minus2]
    mov rdx, 1
    syscall
    pop rax
    neg rax

.ps2_pos:
    lea r9, [rbp - 20]
    mov rcx, 0

    test rax, rax
    jnz .ps2_loop
    dec r9
    mov byte [r9], '0'
    inc rcx
    jmp .ps2_print_it

.ps2_loop:
    test rax, rax
    jz .ps2_print_it
    xor rdx, rdx
    push r9
    mov r9, 10
    div r9
    pop r9
    add dl, '0'
    dec r9
    mov [r9], dl
    inc rcx
    jmp .ps2_loop

.ps2_print_it:
    mov rax, 1
    mov rdi, 1
    mov rsi, r9
    mov rdx, rcx
    syscall

    mov rsp, rbp
    pop rbp
    ret

section .data
    minus2  db "-"
```

---

## 16. สรุปและแนวปฏิบัติที่ดี (Best Practices)

### 16.1 Checklist สำหรับ Procedures

```nasm
; ============================================================
; ไฟล์: best_practices_demo.asm
; วัตถุประสงค์: แสดง best practices สำหรับการเขียน procedures
; คอมไพล์: nasm -f elf64 best_practices_demo.asm -o bp.o && ld bp.o -o bp
; ============================================================

; ============================================================
; BEST PRACTICES:
;
; 1. เสมอ document function interface:
;    ; Input:  RDI = ..., RSI = ...
;    ; Output: RAX = ...
;    ; Clobbers: RCX, RDX (caller-saved ที่เราใช้)
;    ; Saves: RBX, R12 (callee-saved ที่เราใช้)
;
; 2. Align stack ให้ครบ 16 bytes ก่อน CALL
;    ; RSP % 16 == 0 ก่อน CALL
;    ; (หลัง push rbp มันจะ misaligned, ดังนั้น sub rsp ต้องเป็น 8, 24, 40, ... หรือ 16, 32, 48, ...)
;
; 3. Save/restore callee-saved registers (RBX, RBP, R12-R15)
;
; 4. ตรวจสอบ input validity
;    ; NULL pointer check
;    ; bounds check
;    ; division by zero
;
; 5. ใช้ CALL/RET matching เสมอ
;    ; อย่า jmp ออกจาก function โดยไม่ทำ cleanup
;
; 6. Document side effects และ prerequisites
;
; ============================================================

section .data
    good_msg    db "Well-documented procedure!", 0x0A
    good_len    equ $ - good_msg
    err_null    db "ERROR: null pointer", 0x0A
    err_null_l  equ $ - err_null

section .text
    global _start

_start:
    ; ทดสอบ well-documented procedure
    lea rdi, [rel good_msg]     ; valid pointer
    mov rsi, good_len           ; valid length
    call safe_write

    ; ทดสอบด้วย NULL pointer
    xor rdi, rdi                ; NULL pointer
    mov rsi, 10
    call safe_write

    mov rax, 60
    xor rdi, rdi
    syscall

; ============================================================
; safe_write: เขียนข้อความ พร้อม validation ครบถ้วน
;
; Input:
;   RDI = pointer to buffer (must NOT be NULL)
;   RSI = length (must be > 0)
;
; Output:
;   RAX = number of bytes written, or -1 on error
;
; Clobbers:
;   RAX, RDX (used internally, caller-saved so OK)
;
; Preserves (callee-saved registers used):
;   None in this function
;
; Prerequisites:
;   Buffer must be readable
;   Length must be positive
;
; Side effects:
;   Writes to stdout (fd 1)
; ============================================================
safe_write:
    push rbp
    mov rbp, rsp

    ; === INPUT VALIDATION ===

    ; Check: pointer must not be NULL
    test rdi, rdi
    jz .err_null_ptr

    ; Check: length must be > 0
    test rsi, rsi
    jle .err_bad_len

    ; === FUNCTION BODY ===
    mov rax, 1          ; syscall: write
    ; rdi already = fd (wait, we used rdi for pointer!)
    ; ต้อง restructure...
    mov rdx, rsi        ; length
    mov rsi, rdi        ; buffer pointer
    mov rdi, 1          ; fd = stdout
    syscall

    ; RAX = bytes written (return value from syscall)
    pop rbp
    ret

.err_null_ptr:
    ; แสดง error message
    mov rax, 1
    mov rdi, 1
    mov rsi, err_null
    mov rdx, err_null_l
    syscall

    mov rax, -1         ; return error
    pop rbp
    ret

.err_bad_len:
    mov rax, -1
    pop rbp
    ret
```

---

## แบบฝึกหัด (Exercises)

### ระดับพื้นฐาน

**Exercise 1:** เขียน procedure `max_of_two` ที่รับ RDI และ RSI แล้วคืนค่าที่มากกว่าใน RAX
พร้อม prologue/epilogue ครบถ้วน

```nasm
; แนว:
; max_of_two:
;     push rbp
;     mov rbp, rsp
;     cmp rdi, rsi
;     ; ใส่ code ที่นี่...
;     pop rbp
;     ret
```

**Exercise 2:** เขียน function ที่คำนวณ sum ของ array โดยรับ:
- RDI = pointer ไปยัง array (int64)
- RSI = จำนวนสมาชิก
- Output: RAX = sum

### ระดับกลาง

**Exercise 3:** Implement `recursive_sum(n)` ที่คำนวณ 1+2+3+...+n แบบ recursive
แล้วเปรียบเทียบกับ version แบบ iterative และ version ที่ใช้สูตร n*(n+1)/2

**Exercise 4:** สร้าง function pointer table สำหรับ string operations:
- `str_length(ptr)`: คำนวณความยาว string
- `str_copy(dst, src)`: copy string
- `str_compare(s1, s2)`: เปรียบเทียบ strings

### ระดับสูง

**Exercise 5:** Implement `fibonacci_memoized(n)` ที่:
- ใช้ array ขนาด 100 เก็บ calculated values
- ถ้า fib(n) คำนวณแล้ว ให้ดึงจาก cache
- ถ้ายัง ให้คำนวณแล้วเก็บใน cache
- แสดงผล fib(0) ถึง fib(50)

**Exercise 6:** Implement `quicksort` แบบ recursive:
- Input: RDI = pointer ไปยัง array, RSI = left index, RDX = right index
- จัดเรียง in-place
- ทดสอบกับ array ขนาด 20 elements

**Exercise 7:** สร้าง simple call stack inspector ที่:
- Walk up call stack โดยอ่าน saved RBP chain
- แสดง return addresses ทั้งหมด
- หยุดเมื่อ RBP เป็น 0 หรือ pointer ไม่ valid

---

## คำสั่งคอมไพล์และรัน (Compilation Reference)

```bash
# Standard 64-bit Linux executable
nasm -f elf64 filename.asm -o filename.o
ld filename.o -o filename
./filename

# กับ debug info (สำหรับ GDB)
nasm -f elf64 -g -F dwarf filename.asm -o filename.o
ld filename.o -o filename
gdb ./filename

# Link กับ C library (ใช้ main แทน _start)
nasm -f elf64 filename.asm -o filename.o
gcc -no-pie filename.o -o filename
# หรือ
gcc -m64 filename.o -o filename

# 32-bit mode (บน 64-bit system)
nasm -f elf32 filename.asm -o filename.o
ld -m elf_i386 filename.o -o filename

# Bootloader (raw binary)
nasm -f bin bootloader.asm -o boot.bin
qemu-system-x86_64 -drive format=raw,file=boot.bin

# รัน debugger
gdb ./filename
(gdb) break _start
(gdb) run
(gdb) info registers
(gdb) x/20xg $rsp    # แสดง 20 qwords จาก RSP
(gdb) backtrace       # แสดง call stack
```

---

## สรุปเนื้อหา (Summary)

| Concept | คำอธิบาย | ตัวอย่าง |
|---------|----------|----------|
| Near CALL | เรียก procedure ใน segment เดียวกัน | `call my_func` |
| Far CALL | เรียกข้าม segment | `call far [ptr]` |
| Indirect CALL | เรียกผ่าน register/memory | `call rax` หรือ `call [rax]` |
| RET | Near return (pop RIP) | `ret` |
| RETF | Far return (pop RIP + CS) | `retf` |
| RET N | Return + deallocate N bytes | `ret 16` |
| Prologue | push rbp; mov rbp, rsp; sub rsp, N | มาตรฐาน |
| Epilogue | mov rsp, rbp; pop rbp; ret | มาตรฐาน |
| ENTER N, 0 | เทียบเท่า prologue | `enter 16, 0` |
| LEAVE | เทียบเท่า epilogue ไม่รวม ret | `leave` |
| TCO | แทน CALL+RET ด้วย JMP | `jmp func` |
| Stack Canary | ค่า sentinel ป้องกัน overflow | manual implementation |
| Function Table | Array ของ function pointers | dispatch table |

---

*จบ Part 021: Procedures และ Subroutines*
*Part ถัดไป: Part 022 - Macros และ Conditional Assembly*
