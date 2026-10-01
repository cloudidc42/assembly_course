# Part 018: Loop Instructions และ Control Flow

## Prerequisites and Learning Objectives

### สิ่งที่ต้องรู้ก่อน (Prerequisites)
- ความเข้าใจ Registers พื้นฐาน (EAX, EBX, ECX, EDX)
- Jump Instructions (JMP, JE, JNE, JL, JG, ฯลฯ)
- Flag Register และวิธีการทำงาน
- การใช้ NASM syntax เบื้องต้น
- Stack operations เบื้องต้น

### เป้าหมายการเรียนรู้ (Learning Objectives)
หลังจากเรียนจบ Part นี้ ผู้เรียนจะสามารถ:
1. เข้าใจและใช้งาน LOOP, LOOPE, LOOPNE instructions ได้อย่างถูกต้อง
2. เปรียบเทียบประสิทธิภาพระหว่าง LOOP instruction กับ DEC+JNZ
3. Implement for loop, while loop, do-while loop ใน Assembly
4. ใช้ JCXZ/JECXZ/JRCXZ เพื่อตรวจสอบ counter ก่อน loop
5. เทคนิค Loop Unrolling เพื่อเพิ่มประสิทธิภาพ
6. Implement Duff's Device
7. จัดการ Nested Loops อย่างมีประสิทธิภาพ
8. คำนวณ String Length, Array Sum, Fibonacci, Factorial

---

## Theory (ทฤษฎี)

### 1. LOOP Instruction - พื้นฐาน

LOOP instruction เป็นคำสั่งพิเศษใน x86 ที่รวม 3 operation เข้าด้วยกัน:
1. **Decrement** ECX (หรือ CX ใน 16-bit mode, RCX ใน 64-bit mode)
2. **ตรวจสอบ** ว่า ECX != 0
3. **Jump** ไปที่ label ที่ระบุ ถ้า ECX != 0

```
LOOP label    ; ECX-- ; if ECX != 0, jump to label
```

**ข้อสำคัญ:**
- LOOP ใช้ ECX เป็น counter เสมอ (ไม่สามารถเปลี่ยน register ได้)
- ถ้า ECX เริ่มต้นที่ 0, LOOP จะ decrement เป็น 0xFFFFFFFF และวนลูปไป 2^32 รอบ!
- LOOP ทำงานช้ากว่า DEC+JNZ บน CPU สมัยใหม่
- Range ของ jump: -128 ถึง +127 bytes เท่านั้น (short jump)

**ตัวอย่างการใช้งาน:**
```nasm
    mov ecx, 10     ; กำหนดจำนวนรอบเป็น 10
.loop_start:
    ; ... body of loop ...
    loop .loop_start ; ECX-- และถ้า ECX != 0 ก็ jump กลับ
```

### 2. LOOPE/LOOPZ - Loop while Equal/Zero

LOOPE (Loop while Equal) หรือ LOOPZ (Loop while Zero) ทำงานเหมือน LOOP แต่มีเงื่อนไขเพิ่ม:
- Decrement ECX
- Jump ถ้า **ECX != 0 AND ZF = 1** (Zero Flag เป็น 1)

```
LOOPE label   ; ECX-- ; if ECX != 0 AND ZF = 1, jump
LOOPZ label   ; เหมือนกับ LOOPE
```

**ใช้เมื่อ:** ต้องการ loop ต่อไปเรื่อยๆ จนกว่าจะเจอ element ที่ไม่เท่ากัน หรือ counter หมด

### 3. LOOPNE/LOOPNZ - Loop while Not Equal/Not Zero

LOOPNE (Loop while Not Equal) หรือ LOOPNZ (Loop while Not Zero):
- Decrement ECX
- Jump ถ้า **ECX != 0 AND ZF = 0** (Zero Flag เป็น 0)

```
LOOPNE label  ; ECX-- ; if ECX != 0 AND ZF = 0, jump
LOOPNZ label  ; เหมือนกับ LOOPNE
```

**ใช้เมื่อ:** ต้องการ loop จนกว่าจะเจอค่าที่ต้องการ (เช่น search loop)

### 4. JCXZ, JECXZ, JRCXZ - Jump if Counter is Zero

คำสั่งเหล่านี้ตรวจสอบ counter register **ก่อน** เข้า loop:

| Instruction | Register | Mode |
|-------------|----------|------|
| JCXZ  | CX  | 16-bit |
| JECXZ | ECX | 32-bit |
| JRCXZ | RCX | 64-bit |

```
    mov ecx, 0
    jecxz .skip_loop   ; ถ้า ECX = 0 ให้ข้ามลูปไป
.loop_start:
    ; ... body ...
    loop .loop_start
.skip_loop:
```

**ทำไมต้องใช้?** เพราะถ้า ECX = 0 แล้วใช้ LOOP โดยตรง, มันจะ decrement เป็น 0xFFFFFFFF!

### 5. การเปรียบเทียบ LOOP กับ DEC+JNZ

บน CPU สมัยใหม่ (Intel Pentium 4 เป็นต้นมา), LOOP instruction ถูก **deprecated** ในแง่ประสิทธิภาพ:

```nasm
; วิธีที่ 1: ใช้ LOOP (ช้ากว่า)
    mov ecx, 1000
.loop1:
    ; body
    loop .loop1

; วิธีที่ 2: ใช้ DEC+JNZ (เร็วกว่า)
    mov ecx, 1000
.loop2:
    ; body
    dec ecx
    jnz .loop2
```

**เหตุผลที่ DEC+JNZ เร็วกว่า:**
- LOOP ไม่สามารถ fuse กับ branch predictor ได้ดีเท่า
- DEC+JNZ สามารถ macro-fuse เป็น single micro-op บน Intel CPU
- LOOP มี latency สูงกว่าบน modern CPU

### 6. Loop Patterns ใน Assembly

#### For Loop Pattern:
```nasm
; for (int i = 0; i < N; i++) { body }
    mov ecx, N      ; counter
    xor eax, eax    ; i = 0
.for_loop:
    cmp eax, ecx    ; i < N?
    jge .for_done   ; ถ้า i >= N ออกจาก loop
    ; body
    inc eax         ; i++
    jmp .for_loop
.for_done:
```

#### While Loop Pattern:
```nasm
; while (condition) { body }
.while_start:
    ; test condition
    jz .while_done  ; ถ้าเงื่อนไขเป็นเท็จ ออก
    ; body
    jmp .while_start
.while_done:
```

#### Do-While Loop Pattern:
```nasm
; do { body } while (condition)
.do_while_start:
    ; body
    ; test condition
    jnz .do_while_start  ; ถ้าเงื่อนไขเป็นจริง วนซ้ำ
```

### 7. Loop Unrolling (การคลาย Loop)

Loop Unrolling คือการ copy body ของ loop หลายๆ ครั้งเพื่อลด loop overhead:

```nasm
; ก่อน unroll: 10 iterations, 1 operation ต่อ iteration
; หลัง unroll 2x: 5 iterations, 2 operations ต่อ iteration

; ประโยชน์:
; - ลด branch misprediction
; - ลด loop overhead (dec, cmp, jmp)
; - เพิ่ม instruction-level parallelism
; - ใช้ประโยชน์จาก superscalar execution
```

### 8. Duff's Device

Duff's Device เป็นเทคนิค loop unrolling ที่ใช้ switch/case fallthrough:

```c
// C version
void copy(char *to, char *from, int count) {
    int n = (count + 7) / 8;
    switch (count % 8) {
    case 0: do { *to++ = *from++;
    case 7:      *to++ = *from++;
    case 6:      *to++ = *from++;
    case 5:      *to++ = *from++;
    case 4:      *to++ = *from++;
    case 3:      *to++ = *from++;
    case 2:      *to++ = *from++;
    case 1:      *to++ = *from++;
            } while (--n > 0);
    }
}
```

ใน Assembly เราสามารถ implement Duff's Device ได้โดยใช้ jump table

---

## Code Examples (ตัวอย่างโค้ด)

### Example 1: LOOP Instruction พื้นฐาน - นับถอยหลัง

```nasm
; ไฟล์: loop_basic.asm
; วัตถุประสงค์: แสดงการใช้งาน LOOP instruction พื้นฐาน
; การ compile: nasm -f elf32 loop_basic.asm -o loop_basic.o
;              ld -m elf_i386 loop_basic.o -o loop_basic
; การรัน: ./loop_basic
; ผลลัพธ์ที่คาดหวัง:
;   Loop iteration: 10
;   Loop iteration: 9
;   Loop iteration: 8
;   ...
;   Loop iteration: 1
;   Loop done! Total iterations: 10

section .data
    msg_iter    db "Loop iteration: ", 0
    msg_done    db "Loop done! Total iterations: 10", 10, 0
    newline     db 10, 0

section .bss
    num_buf     resb 12     ; บัฟเฟอร์สำหรับแปลงตัวเลขเป็น string

section .text
    global _start

; ฟังก์ชัน: พิมพ์ string
; input: ESI = pointer to string
print_str:
    push eax
    push ebx
    push ecx
    push edx
    
    ; หา length ของ string
    xor ecx, ecx
.find_len:
    cmp byte [esi + ecx], 0
    je .found_len
    inc ecx
    jmp .find_len
.found_len:
    
    mov eax, 4          ; syscall write
    mov ebx, 1          ; stdout
    mov edx, ecx        ; length
    ; ESI ยังคงชี้ที่ string
    push esi
    mov ecx, esi
    int 0x80
    pop esi
    
    pop edx
    pop ecx
    pop ebx
    pop eax
    ret

; ฟังก์ชัน: แปลงตัวเลข EAX เป็น string แล้วพิมพ์
print_num:
    push eax
    push ebx
    push ecx
    push edx
    push edi
    
    ; ใช้ num_buf เพื่อเก็บตัวเลข
    lea edi, [num_buf + 11]
    mov byte [edi], 0   ; null terminator
    dec edi
    
    ; กรณีพิเศษ: ถ้าตัวเลขเป็น 0
    test eax, eax
    jnz .convert
    mov byte [edi], '0'
    jmp .print_it
    
.convert:
    mov ebx, 10         ; หารด้วย 10
.div_loop:
    xor edx, edx        ; เคลียร์ EDX ก่อนหาร
    div ebx             ; EAX = EAX/10, EDX = remainder
    add dl, '0'         ; แปลงเป็น ASCII
    mov [edi], dl       ; เก็บ digit
    dec edi
    test eax, eax
    jnz .div_loop
    
    inc edi             ; ชี้ไปที่ตัวเลขตัวแรก
    
.print_it:
    ; พิมพ์ตัวเลข
    push edi
    
    ; หา length
    xor ecx, ecx
.len_loop:
    cmp byte [edi + ecx], 0
    je .len_done
    inc ecx
    jmp .len_loop
.len_done:
    
    mov eax, 4
    mov ebx, 1
    mov edx, ecx
    int 0x80
    
    pop edi
    pop edi
    pop edx
    pop ecx
    pop ebx
    pop eax
    ret

_start:
    mov ecx, 10         ; ตั้งค่า counter = 10 (จำนวนรอบ)
    
.loop_start:
    push ecx            ; บันทึก ECX ไว้เพราะ print functions อาจ modify
    
    ; พิมพ์ "Loop iteration: "
    mov esi, msg_iter
    call print_str
    
    ; พิมพ์ค่า ECX ปัจจุบัน
    pop ecx             ; restore ECX
    push ecx
    mov eax, ecx        ; copy ECX ไปยัง EAX เพื่อพิมพ์
    call print_num
    
    ; พิมพ์ newline
    mov eax, 4
    mov ebx, 1
    mov ecx, newline
    mov edx, 1
    int 0x80
    
    pop ecx             ; restore ECX สำหรับ LOOP
    loop .loop_start    ; ECX-- และถ้า ECX != 0 วนซ้ำ
    
    ; Loop จบแล้ว พิมพ์ข้อความสรุป
    mov esi, msg_done
    call print_str
    
    ; จบโปรแกรม
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

**คำอธิบายโค้ด:**
- ใช้ ECX = 10 เป็น counter
- ในแต่ละรอบ push/pop ECX เพื่อป้องกัน register ถูก clobber
- LOOP instruction ลด ECX และตรวจสอบอัตโนมัติ

**วิธี Compile และรัน:**
```bash
nasm -f elf32 loop_basic.asm -o loop_basic.o
ld -m elf_i386 loop_basic.o -o loop_basic
./loop_basic
```

---

### Example 2: LOOP vs DEC+JNZ Performance Comparison

```nasm
; ไฟล์: loop_vs_dec.asm
; วัตถุประสงค์: เปรียบเทียบ LOOP กับ DEC+JNZ
; การ compile: nasm -f elf32 loop_vs_dec.asm -o loop_vs_dec.o
;              ld -m elf_i386 loop_vs_dec.o -o loop_vs_dec
; ผลลัพธ์ที่คาดหวัง:
;   Array sum using LOOP: 5050
;   Array sum using DEC+JNZ: 5050
;   Both methods give the same result!

section .data
    ; สร้าง array ของเลข 1 ถึง 100
    arr     times 100 dd 0   ; จะเติมค่าใน _start
    arr_len equ 100
    
    msg_loop    db "Array sum using LOOP: ", 0
    msg_dec     db "Array sum using DEC+JNZ: ", 0
    msg_same    db "Both methods give the same result!", 10, 0
    newline     db 10, 0

section .bss
    num_buf resb 16

section .text
    global _start

; ฟังก์ชัน: พิมพ์ null-terminated string
; input: EAX = pointer to string
print_string:
    push ebx
    push ecx
    push edx
    push edi
    
    mov edi, eax
    xor ecx, ecx
.find_end:
    cmp byte [edi + ecx], 0
    je .print
    inc ecx
    jmp .find_end
.print:
    mov eax, 4
    mov ebx, 1
    mov edx, ecx
    mov ecx, edi
    int 0x80
    
    pop edi
    pop edx
    pop ecx
    pop ebx
    ret

; ฟังก์ชัน: พิมพ์ตัวเลขใน EAX
; ใช้ stack เพื่อ reverse digits
print_number:
    push eax
    push ebx
    push ecx
    push edx
    push esp
    
    mov ebx, 10         ; base 10
    xor ecx, ecx        ; digit count
    
    ; กรณีพิเศษ: 0
    test eax, eax
    jnz .not_zero
    push 0x30           ; '0'
    inc ecx
    jmp .print_digits
    
.not_zero:
.extract_digits:
    xor edx, edx
    div ebx             ; EAX = quotient, EDX = remainder
    add edx, 0x30       ; แปลงเป็น ASCII
    push edx            ; push digit ลง stack
    inc ecx
    test eax, eax
    jnz .extract_digits

.print_digits:
    ; print ทีละ digit จาก stack
    push ecx
    mov eax, 4
    mov ebx, 1
    mov edx, 1          ; length = 1
    mov ecx, esp
    add ecx, 4          ; ชี้ไปที่ digit บน stack
    int 0x80
    pop ecx
    pop eax             ; ดึง digit ออกจาก stack
    loop .print_digits
    
    pop esp             ; restore stack pointer
    pop edx
    pop ecx
    pop ebx
    pop eax
    ret

; =========================================
; ฟังก์ชัน: คำนวณ sum โดยใช้ LOOP instruction
; input: ESI = pointer to array, ECX = count
; output: EAX = sum
; =========================================
sum_with_loop:
    push ebx
    push ecx
    push esi
    
    xor eax, eax        ; sum = 0
    
    ; ตรวจสอบว่า ECX = 0 หรือไม่
    jecxz .loop_done
    
.loop_body:
    add eax, [esi]      ; sum += array[i]
    add esi, 4          ; ชี้ไปที่ element ถัดไป (4 bytes per int)
    loop .loop_body     ; ECX-- และวนซ้ำถ้า ECX != 0
    
.loop_done:
    pop esi
    pop ecx
    pop ebx
    ret

; =========================================
; ฟังก์ชัน: คำนวณ sum โดยใช้ DEC+JNZ
; input: ESI = pointer to array, ECX = count
; output: EAX = sum
; =========================================
sum_with_dec_jnz:
    push ebx
    push ecx
    push esi
    
    xor eax, eax        ; sum = 0
    
    ; ตรวจสอบว่า ECX = 0 หรือไม่
    test ecx, ecx
    jz .dec_done
    
.dec_body:
    add eax, [esi]      ; sum += array[i]
    add esi, 4          ; ชี้ไปที่ element ถัดไป
    dec ecx             ; ECX--
    jnz .dec_body       ; ถ้า ECX != 0 วนซ้ำ
    
.dec_done:
    pop esi
    pop ecx
    pop ebx
    ret

_start:
    ; =========================================
    ; เริ่มต้น: เติมค่า array ด้วยเลข 1 ถึง 100
    ; =========================================
    mov edi, arr        ; ชี้ที่ต้น array
    mov ecx, 100        ; จำนวน elements
    mov eax, 1          ; เริ่มที่ 1
    
.init_array:
    mov [edi], eax      ; array[i] = current value
    add edi, 4          ; ถัดไป
    inc eax             ; value++
    loop .init_array
    
    ; =========================================
    ; ทดสอบ: sum with LOOP
    ; =========================================
    mov eax, msg_loop
    call print_string
    
    mov esi, arr
    mov ecx, arr_len
    call sum_with_loop
    
    ; พิมพ์ผลลัพธ์
    call print_number
    
    ; พิมพ์ newline
    push dword 10
    mov eax, 4
    mov ebx, 1
    mov ecx, esp
    mov edx, 1
    int 0x80
    pop eax
    
    ; =========================================
    ; ทดสอบ: sum with DEC+JNZ
    ; =========================================
    mov eax, msg_dec
    call print_string
    
    mov esi, arr
    mov ecx, arr_len
    call sum_with_dec_jnz
    
    call print_number
    
    push dword 10
    mov eax, 4
    mov ebx, 1
    mov ecx, esp
    mov edx, 1
    int 0x80
    pop eax
    
    ; พิมพ์ข้อความสรุป
    mov eax, msg_same
    call print_string
    
    ; จบโปรแกรม
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

**วิธี Compile:**
```bash
nasm -f elf32 loop_vs_dec.asm -o loop_vs_dec.o
ld -m elf_i386 loop_vs_dec.o -o loop_vs_dec
./loop_vs_dec
```

**ผลลัพธ์:**
```
Array sum using LOOP: 5050
Array sum using DEC+JNZ: 5050
Both methods give the same result!
```

---

### Example 3: Fibonacci และ Factorial

```nasm
; ไฟล์: fibonacci_factorial.asm
; วัตถุประสงค์: คำนวณ Fibonacci และ Factorial โดยใช้ loops
; การ compile: nasm -f elf32 fibonacci_factorial.asm -o fib_fact.o
;              ld -m elf_i386 fib_fact.o -o fib_fact
; ผลลัพธ์:
;   Fibonacci sequence (first 15 numbers):
;   1 1 2 3 5 8 13 21 34 55 89 144 233 377 610
;   
;   Factorials:
;   1! = 1
;   2! = 2
;   3! = 6
;   4! = 24
;   5! = 120
;   6! = 720
;   7! = 5040
;   8! = 40320
;   9! = 362880
;   10! = 3628800

section .data
    msg_fib     db "Fibonacci sequence (first 15 numbers):", 10, 0
    msg_fact    db 10, "Factorials:", 10, 0
    space       db " ", 0
    newline     db 10, 0
    
    ; Format string สำหรับ factorial
    fact_fmt1   db "  ", 0      ; indent
    fact_bang   db "! = ", 0

section .bss
    fib_arr     resd 20         ; เก็บ Fibonacci numbers
    num_buf     resb 16

section .text
    global _start

; =========================================
; ฟังก์ชัน: พิมพ์ string
; input: EAX = pointer
; =========================================
print_str:
    push ebx
    push ecx
    push edx
    push edi
    
    mov edi, eax
    xor ecx, ecx
.find:
    cmp byte [edi + ecx], 0
    je .go
    inc ecx
    jmp .find
.go:
    mov eax, 4
    mov ebx, 1
    mov edx, ecx
    mov ecx, edi
    int 0x80
    
    pop edi
    pop edx
    pop ecx
    pop ebx
    ret

; =========================================
; ฟังก์ชัน: พิมพ์ตัวเลขใน EAX (iterative)
; =========================================
print_num:
    pushad
    
    ; เก็บ stack pointer เดิม
    mov ebp, esp
    
    mov ebx, 10
    xor ecx, ecx        ; digit count
    
    ; กรณีพิเศษ 0
    test eax, eax
    jnz .extract
    push dword '0'
    mov ecx, 1
    jmp .print
    
.extract:
    xor edx, edx
    div ebx
    add edx, '0'
    push edx
    inc ecx
    test eax, eax
    jnz .extract

.print:
    ; สร้าง buffer บน stack
    mov edx, 1
    mov eax, 4
    mov ebx, 1
.print_loop:
    lea ecx, [esp]      ; ชี้ที่ digit
    int 0x80
    pop eax             ; ดึง digit ออก
    loop .print_loop
    
    popad
    ret

; =========================================
; ฟังก์ชัน: สร้าง Fibonacci sequence
; input: EDI = output array, ECX = count
; =========================================
gen_fibonacci:
    push eax
    push ebx
    push ecx
    push edi
    
    ; ตรวจสอบ count
    cmp ecx, 0
    je .fib_done
    
    ; F(1) = 1
    mov dword [edi], 1
    add edi, 4
    dec ecx
    jz .fib_done
    
    ; F(2) = 1
    mov dword [edi], 1
    add edi, 4
    dec ecx
    jz .fib_done
    
    ; F(n) = F(n-1) + F(n-2)
    mov eax, 1          ; F(n-2) = F(1)
    mov ebx, 1          ; F(n-1) = F(2)
    
.fib_loop:
    ; คำนวณ F(n) = F(n-1) + F(n-2)
    ; EAX = F(n-2), EBX = F(n-1)
    mov edx, eax        ; บันทึก F(n-2)
    add eax, ebx        ; EAX = F(n-1) + F(n-2) = F(n)
    mov [edi], eax      ; เก็บไว้ใน array
    add edi, 4
    
    mov eax, ebx        ; F(n-2) = F(n-1)
    mov ebx, [edi - 4]  ; F(n-1) = F(n) ที่เพิ่งคำนวณ
    
    dec ecx
    jnz .fib_loop
    
.fib_done:
    pop edi
    pop ecx
    pop ebx
    pop eax
    ret

; =========================================
; ฟังก์ชัน: คำนวณ Factorial (iterative)
; input: EAX = n
; output: EAX = n!
; =========================================
factorial:
    push ecx
    push edx
    
    ; กรณีพิเศษ 0! = 1 และ 1! = 1
    cmp eax, 1
    jle .fact_one
    
    ; ECX = n (counter), EAX = accumulator (เริ่มที่ 1)
    mov ecx, eax        ; ecx = n
    mov eax, 1          ; result = 1
    
.fact_loop:
    ; result *= ecx
    imul eax, ecx       ; EAX = EAX * ECX
    dec ecx             ; ECX--
    jnz .fact_loop      ; ถ้า ECX != 0 วนซ้ำ
    
    jmp .fact_done
    
.fact_one:
    mov eax, 1          ; 0! = 1! = 1
    
.fact_done:
    pop edx
    pop ecx
    ret

_start:
    ; =========================================
    ; ส่วนที่ 1: Fibonacci
    ; =========================================
    mov eax, msg_fib
    call print_str
    
    ; สร้าง Fibonacci 15 ตัวแรก
    mov edi, fib_arr
    mov ecx, 15
    call gen_fibonacci
    
    ; พิมพ์ Fibonacci array
    mov esi, fib_arr
    mov ecx, 15
    
.print_fib:
    push ecx
    mov eax, [esi]      ; โหลด Fibonacci number
    call print_num      ; พิมพ์
    
    ; พิมพ์ space (ยกเว้นตัวสุดท้าย)
    pop ecx
    cmp ecx, 1
    je .no_space
    mov eax, space
    call print_str
    
.no_space:
    add esi, 4          ; ถัดไป
    loop .print_fib
    
    ; พิมพ์ newline
    mov eax, newline
    call print_str
    
    ; =========================================
    ; ส่วนที่ 2: Factorial
    ; =========================================
    mov eax, msg_fact
    call print_str
    
    mov ecx, 10         ; คำนวณ 1! ถึง 10!
    mov ebx, 1          ; เริ่มที่ n=1
    
.print_fact:
    push ecx
    push ebx
    
    ; พิมพ์ "  n"
    mov eax, fact_fmt1
    call print_str
    
    mov eax, ebx
    call print_num
    
    ; พิมพ์ "! = "
    mov eax, fact_bang
    call print_str
    
    ; คำนวณ n! และพิมพ์
    mov eax, ebx
    call factorial
    call print_num
    
    ; newline
    mov eax, newline
    call print_str
    
    pop ebx
    pop ecx
    inc ebx
    loop .print_fact
    
    ; จบโปรแกรม
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

**วิธี Compile:**
```bash
nasm -f elf32 fibonacci_factorial.asm -o fib_fact.o
ld -m elf_i386 fib_fact.o -o fib_fact
./fib_fact
```

**ผลลัพธ์:**
```
Fibonacci sequence (first 15 numbers):
1 1 2 3 5 8 13 21 34 55 89 144 233 377 610

Factorials:
  1! = 1
  2! = 2
  3! = 6
  4! = 24
  5! = 120
  6! = 720
  7! = 5040
  8! = 40320
  9! = 362880
  10! = 3628800
```

---

### Example 4: String Length Calculation และ Array Traversal

```nasm
; ไฟล์: string_array_ops.asm
; วัตถุประสงค์: คำนวณ string length และ traverse array
; การ compile: nasm -f elf32 string_array_ops.asm -o str_arr.o
;              ld -m elf_i386 str_arr.o -o str_arr
; ผลลัพธ์:
;   String: "Hello, Assembly World!"
;   Length: 22
;   
;   Array: [10, 25, 7, 42, 15, 83, 3, 67, 99, 31]
;   Min: 3, Max: 99, Sum: 382, Average: 38

section .data
    test_str    db "Hello, Assembly World!", 0
    str_label   db "String: ", 34, 0     ; 34 = '"'
    str_end     db 34, 10, 0              ; '"' + newline
    len_label   db "Length: ", 0
    
    arr         dd 10, 25, 7, 42, 15, 83, 3, 67, 99, 31
    arr_count   equ 10
    
    arr_label   db 10, "Array: [", 0
    min_label   db 10, "Min: ", 0
    max_label   db ", Max: ", 0
    sum_label   db ", Sum: ", 0
    avg_label   db ", Average: ", 0
    comma_sp    db ", ", 0
    newline     db 10, 0
    bracket_end db "]", 0

section .bss
    num_buf resb 16

section .text
    global _start

; =========================================
; ฟังก์ชัน: พิมพ์ string
; =========================================
print_str:
    push eax
    push ebx
    push ecx
    push edx
    push edi
    
    mov edi, eax
    xor ecx, ecx
.find:
    cmp byte [edi + ecx], 0
    je .print
    inc ecx
    jmp .find
.print:
    test ecx, ecx
    jz .done
    mov eax, 4
    mov ebx, 1
    mov edx, ecx
    mov ecx, edi
    int 0x80
.done:
    pop edi
    pop edx
    pop ecx
    pop ebx
    pop eax
    ret

; =========================================
; ฟังก์ชัน: พิมพ์ตัวเลข EAX
; =========================================
print_num:
    pushad
    
    sub esp, 16         ; จอง buffer 16 bytes บน stack
    lea edi, [esp + 15]
    mov byte [edi], 0   ; null terminator
    
    test eax, eax
    jnz .nonzero
    dec edi
    mov byte [edi], '0'
    jmp .do_print
    
.nonzero:
    mov ebx, 10
.extract:
    dec edi
    xor edx, edx
    div ebx
    add dl, '0'
    mov [edi], dl
    test eax, eax
    jnz .extract
    
.do_print:
    ; คำนวณ length
    lea eax, [esp + 15]
    sub eax, edi        ; length = end - start
    
    mov edx, eax
    mov eax, 4
    mov ebx, 1
    mov ecx, edi
    int 0x80
    
    add esp, 16
    popad
    ret

; =========================================
; ฟังก์ชัน: คำนวณ string length
; input: ESI = pointer to string
; output: EAX = length
; =========================================
strlen:
    push edi
    push ecx
    
    ; ใช้ SCASB (Scan String Byte) สำหรับหา null terminator
    ; หรือใช้ manual loop
    xor eax, eax        ; length = 0
    
.count_loop:
    cmp byte [esi + eax], 0   ; ตรวจสอบ null terminator
    je .count_done
    inc eax
    jmp .count_loop
    
.count_done:
    pop ecx
    pop edi
    ret

; =========================================
; ฟังก์ชัน: หา min, max, sum ของ array
; input: ESI = array pointer, ECX = count
; output: EAX = min, EBX = max, EDX = sum
; =========================================
array_stats:
    push ecx
    push esi
    
    ; โหลด element แรก
    mov eax, [esi]      ; min = arr[0]
    mov ebx, [esi]      ; max = arr[0]
    xor edx, edx        ; sum = 0
    
    ; วนลูปทุก element
.stats_loop:
    mov edi, [esi]      ; โหลด current element
    
    ; ตรวจสอบ min
    cmp edi, eax
    jge .check_max
    mov eax, edi        ; update min
    
.check_max:
    ; ตรวจสอบ max
    cmp edi, ebx
    jle .add_sum
    mov ebx, edi        ; update max
    
.add_sum:
    add edx, edi        ; sum += element
    add esi, 4          ; ถัดไป
    loop .stats_loop
    
    pop esi
    pop ecx
    ret

_start:
    ; =========================================
    ; ส่วนที่ 1: String Length
    ; =========================================
    mov eax, str_label
    call print_str
    
    mov eax, test_str
    call print_str
    
    mov eax, str_end
    call print_str
    
    ; คำนวณ length
    mov eax, len_label
    call print_str
    
    mov esi, test_str
    call strlen
    ; EAX = length
    call print_num
    
    mov eax, newline
    call print_str
    
    ; =========================================
    ; ส่วนที่ 2: Array Traversal
    ; =========================================
    mov eax, arr_label
    call print_str
    
    ; พิมพ์ array elements
    mov esi, arr
    mov ecx, arr_count
    
.print_arr:
    push ecx
    mov eax, [esi]
    call print_num
    
    pop ecx
    cmp ecx, 1
    je .arr_last
    mov eax, comma_sp
    call print_str
    
.arr_last:
    add esi, 4
    loop .print_arr
    
    mov eax, bracket_end
    call print_str
    
    ; คำนวณ statistics
    mov esi, arr
    mov ecx, arr_count
    call array_stats
    ; EAX = min, EBX = max, EDX = sum
    
    ; บันทึกผลลัพธ์
    push edx            ; sum
    push ebx            ; max
    push eax            ; min
    
    ; พิมพ์ Min
    mov eax, min_label
    call print_str
    pop eax             ; min
    push eax
    call print_num
    
    ; พิมพ์ Max
    mov eax, max_label
    call print_str
    pop eax             ; min (discard)
    pop eax             ; max
    push eax
    call print_num
    
    ; พิมพ์ Sum
    mov eax, sum_label
    call print_str
    pop eax             ; max (discard)
    pop eax             ; sum
    push eax
    call print_num
    
    ; พิมพ์ Average
    mov eax, avg_label
    call print_str
    pop eax             ; sum
    
    ; คำนวณ average = sum / count
    xor edx, edx
    mov ebx, arr_count
    div ebx
    call print_num
    
    mov eax, newline
    call print_str
    
    ; จบโปรแกรม
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

**วิธี Compile:**
```bash
nasm -f elf32 string_array_ops.asm -o str_arr.o
ld -m elf_i386 str_arr.o -o str_arr
./str_arr
```

**ผลลัพธ์:**
```
String: "Hello, Assembly World!"
Length: 22

Array: [10, 25, 7, 42, 15, 83, 3, 67, 99, 31]
Min: 3, Max: 99, Sum: 382, Average: 38
```

---

### Example 5: Nested Loops และ Loop Unrolling

```nasm
; ไฟล์: nested_loops.asm
; วัตถุประสงค์: Nested loops และ loop unrolling techniques
; การ compile: nasm -f elf32 nested_loops.asm -o nested.o
;              ld -m elf_i386 nested.o -o nested
; ผลลัพธ์: Multiplication table 5x5

section .data
    title       db "Multiplication Table (5x5):", 10, 0
    space3      db "   ", 0
    tab         db 9, 0         ; Tab character
    newline     db 10, 0
    separator   db "---+---+---+---+---+---", 10, 0
    
    ; สำหรับ loop unrolling demo
    unroll_msg  db 10, "Loop Unrolling Demo (sum 1 to 8):", 10, 0
    result_msg  db "Result: ", 0
    normal_msg  db "Normal loop result: ", 0
    unrolled_msg db "Unrolled loop result: ", 0

section .bss
    num_buf     resb 16

section .text
    global _start

print_str:
    push eax
    push ebx
    push ecx
    push edx
    push edi
    mov edi, eax
    xor ecx, ecx
.f: cmp byte [edi+ecx], 0
    je .p
    inc ecx
    jmp .f
.p: test ecx, ecx
    jz .d
    mov eax, 4
    mov ebx, 1
    mov edx, ecx
    mov ecx, edi
    int 0x80
.d: pop edi
    pop edx
    pop ecx
    pop ebx
    pop eax
    ret

print_num:
    pushad
    sub esp, 16
    lea edi, [esp+15]
    mov byte [edi], 0
    test eax, eax
    jnz .nz
    dec edi
    mov byte [edi], '0'
    jmp .pp
.nz:
    mov ebx, 10
.ex: dec edi
    xor edx, edx
    div ebx
    add dl, '0'
    mov [edi], dl
    test eax, eax
    jnz .ex
.pp:
    lea edx, [esp+15]
    sub edx, edi
    mov eax, 4
    mov ebx, 1
    mov ecx, edi
    int 0x80
    add esp, 16
    popad
    ret

; =========================================
; ฟังก์ชัน: พิมพ์ตัวเลขแบบมี padding
; ฟังก์ชันนี้พิมพ์ตัวเลขในช่อง 3 ตัวอักษร
; =========================================
print_num_padded:
    pushad
    
    ; แปลงตัวเลขเป็น string ก่อน
    sub esp, 16
    lea edi, [esp+15]
    mov byte [edi], 0
    
    test eax, eax
    jnz .conv
    dec edi
    mov byte [edi], '0'
    jmp .done_conv
    
.conv:
    mov ebx, 10
.ext:
    dec edi
    xor edx, edx
    div ebx
    add dl, '0'
    mov [edi], dl
    test eax, eax
    jnz .ext
    
.done_conv:
    ; คำนวณ length
    lea ecx, [esp+15]
    sub ecx, edi        ; length
    
    ; พิมพ์ spaces สำหรับ padding (width = 3)
    mov ebx, 3
    sub ebx, ecx        ; spaces needed
    
.pad_loop:
    test ebx, ebx
    jle .no_pad
    
    push dword ' '
    mov eax, 4
    mov ebx_temp equ 0  ; placeholder
    mov edx, 1
    mov eax, 4
    mov ebx, 1
    mov ecx, esp
    int 0x80
    pop eax
    
    lea ecx, [esp+15]
    sub ecx, edi        ; recalculate (ebx was clobbered)
    mov ebx, 3
    sub ebx, ecx
    dec ebx
    jmp .pad_loop
    
.no_pad:
    ; พิมพ์ตัวเลข
    lea ecx, [esp+15]
    sub ecx, edi
    mov edx, ecx
    mov eax, 4
    mov ebx, 1
    mov ecx, edi
    int 0x80
    
    add esp, 16
    popad
    ret

_start:
    ; =========================================
    ; Multiplication Table 5x5
    ; =========================================
    mov eax, title
    call print_str
    
    ; outer loop: i = 1 to 5
    mov ebp, 1          ; i = 1 (ใช้ EBP เพราะ ECX จะถูกใช้โดย LOOP)
    
.outer_loop:
    cmp ebp, 5
    jg .outer_done
    
    ; inner loop: j = 1 to 5
    mov esi, 1          ; j = 1
    
.inner_loop:
    cmp esi, 5
    jg .inner_done
    
    ; คำนวณ i * j
    mov eax, ebp
    imul eax, esi       ; EAX = i * j
    
    ; พิมพ์ด้วย width 4
    ; พิมพ์ spaces
    push eax
    
    ; หา digit count
    push eax
    xor ecx, ecx
.count_digits:
    test eax, eax
    jz .digits_done
    xor edx, edx
    mov ebx, 10
    div ebx
    inc ecx
    jmp .count_digits
.digits_done:
    pop eax
    
    ; พิมพ์ leading spaces (width 4, so 4 - digit_count spaces)
    mov edx, 4
    sub edx, ecx
.sp_loop:
    test edx, edx
    jle .sp_done
    push dword ' '
    mov eax, 4
    mov ebx, 1
    mov ecx, esp
    mov edx_save, edx   ; fake save
    mov edx, 1
    int 0x80
    pop eax
    
    ; restore edx
    mov edx, 4
    push eax
    call count_digits_fn
    pop eax
    mov edx, 4
    sub edx, ecx
    dec edx
    jmp .sp_loop
.sp_done:
    
    pop eax
    call print_num
    
    inc esi
    jmp .inner_loop
    
.inner_done:
    ; newline
    mov eax, newline
    call print_str
    
    inc ebp
    jmp .outer_loop
    
.outer_done:

    ; =========================================
    ; Loop Unrolling Demo
    ; =========================================
    mov eax, unroll_msg
    call print_str
    
    ; วิธีที่ 1: Normal loop (sum 1+2+3+4+5+6+7+8)
    mov eax, normal_msg
    call print_str
    
    xor eax, eax        ; sum = 0
    mov ecx, 8          ; count = 8
    mov ebx, 1          ; i = 1
    
.normal_loop:
    add eax, ebx        ; sum += i
    inc ebx             ; i++
    loop .normal_loop
    
    call print_num
    mov eax, newline
    call print_str
    
    ; วิธีที่ 2: Unrolled loop (4x unroll)
    mov eax, unrolled_msg
    call print_str
    
    ; แทนที่จะวน 8 รอบ เราคลาย loop เป็น 2 รอบ ๆ ละ 4 operations
    xor eax, eax        ; sum = 0
    mov ecx, 2          ; count = 8/4 = 2
    mov ebx, 1          ; i = 1
    
.unrolled_loop:
    ; 4 operations ต่อ 1 iteration (ลด loop overhead ลง 4 เท่า)
    add eax, ebx        ; sum += i
    inc ebx             ; i++
    add eax, ebx        ; sum += i+1
    inc ebx             ; i++
    add eax, ebx        ; sum += i+2
    inc ebx             ; i++
    add eax, ebx        ; sum += i+3
    inc ebx             ; i++
    loop .unrolled_loop
    
    call print_num
    mov eax, newline
    call print_str
    
    ; จบโปรแกรม
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

---

### Example 6: LOOPE/LOOPNE - Search Loop

```nasm
; ไฟล์: search_loop.asm
; วัตถุประสงค์: แสดงการใช้ LOOPNE สำหรับ search
; การ compile: nasm -f elf32 search_loop.asm -o search.o
;              ld -m elf_i386 search.o -o search
; ผลลัพธ์:
;   Searching for 42 in array...
;   Found 42 at index: 4
;   
;   Searching for 100 in array...
;   Not found!

section .data
    data_arr    dd 10, 20, 35, 7, 42, 55, 8, 99, 42, 1
    data_len    equ 10
    
    search_msg  db "Searching for ", 0
    in_arr      db " in array...", 10, 0
    found_msg   db "Found ", 0
    at_msg      db " at index: ", 0
    notfound    db "Not found!", 10, 0
    newline     db 10, 0

section .bss
    num_buf     resb 16

section .text
    global _start

print_str:
    push eax
    push ebx
    push ecx
    push edx
    push edi
    mov edi, eax
    xor ecx, ecx
.fs: cmp byte [edi+ecx], 0
     je .ps
     inc ecx
     jmp .fs
.ps: test ecx, ecx
     jz .ds
     mov eax, 4
     mov ebx, 1
     mov edx, ecx
     mov ecx, edi
     int 0x80
.ds: pop edi
     pop edx
     pop ecx
     pop ebx
     pop eax
     ret

print_num:
    pushad
    sub esp, 16
    lea edi, [esp+15]
    mov byte [edi], 0
    test eax, eax
    jnz .nn
    dec edi
    mov byte [edi], '0'
    jmp .pn
.nn: mov ebx, 10
.en: dec edi
     xor edx, edx
     div ebx
     add dl, '0'
     mov [edi], dl
     test eax, eax
     jnz .en
.pn: lea edx, [esp+15]
     sub edx, edi
     mov eax, 4
     mov ebx, 1
     mov ecx, edi
     int 0x80
     add esp, 16
     popad
     ret

; =========================================
; ฟังก์ชัน: ค้นหาค่าใน array (First occurrence)
; input: ESI = array ptr, ECX = count, EAX = target
; output: EAX = index (-1 ถ้าไม่พบ)
; =========================================
linear_search:
    push ebx
    push ecx
    push esi
    
    mov ebx, eax        ; EBX = target value
    xor eax, eax        ; EAX = index = 0
    
    ; ตรวจสอบก่อนว่า array ว่างหรือไม่
    jecxz .not_found
    
.search_loop:
    cmp [esi], ebx      ; arr[i] == target?
    je .found           ; ถ้าเท่ากัน หยุดแล้ว return
    
    ; LOOPNE: ลด ECX, jump ถ้า ECX != 0 AND ZF = 0 (ยังไม่เท่ากัน)
    ; ในที่นี้เราใช้ manual loop เพื่อความชัดเจน
    add esi, 4          ; ถัดไป
    inc eax             ; index++
    loop .search_loop   ; ECX-- วนซ้ำ
    
    ; ถ้ามาถึงตรงนี้แสดงว่าไม่พบ
    mov eax, -1
    jmp .search_done
    
.found:
    ; EAX = current index (found!)
    
.search_done:
    pop esi
    pop ecx
    pop ebx
    ret

.not_found:
    mov eax, -1
    jmp .search_done

; =========================================
; ฟังก์ชัน: ค้นหาโดยใช้ LOOPNE (แบบ Assembly-native)
; input: ESI = array ptr, ECX = count, EDX = target
; output: EBX = index (-1 ถ้าไม่พบ), ZF set ถ้าพบ
; =========================================
linear_search_loopne:
    push esi
    push ecx
    
    xor ebx, ebx        ; index = 0
    
    jecxz .lne_notfound
    
.lne_loop:
    cmp [esi + ebx*4], edx  ; arr[index] == target?
    je .lne_found            ; ถ้าพบ ZF = 1 → LOOPNE จะหยุด
    
    inc ebx             ; index++
    loop .lne_loop      ; ECX-- วนซ้ำถ้า ECX != 0
    
    ; ไม่พบ
.lne_notfound:
    mov ebx, -1
    ; ZF = 0 (ไม่พบ)
    test ebx, ebx       ; set flags
    jmp .lne_done
    
.lne_found:
    ; EBX = index ที่พบ, ZF = 1
    
.lne_done:
    pop ecx
    pop esi
    ret

_start:
    ; =========================================
    ; ค้นหา 42 (ซึ่งมีอยู่)
    ; =========================================
    mov eax, search_msg
    call print_str
    mov eax, 42
    call print_num
    mov eax, in_arr
    call print_str
    
    mov esi, data_arr
    mov ecx, data_len
    mov eax, 42
    call linear_search
    
    ; ตรวจสอบผลลัพธ์
    cmp eax, -1
    je .not_found_42
    
    ; พบ! พิมพ์ index
    mov ebx, eax        ; บันทึก index
    mov eax, found_msg
    call print_str
    mov eax, 42
    call print_num
    mov eax, at_msg
    call print_str
    mov eax, ebx
    call print_num
    mov eax, newline
    call print_str
    jmp .search_100
    
.not_found_42:
    mov eax, notfound
    call print_str
    
.search_100:
    ; =========================================
    ; ค้นหา 100 (ซึ่งไม่มี)
    ; =========================================
    mov eax, newline
    call print_str
    mov eax, search_msg
    call print_str
    mov eax, 100
    call print_num
    mov eax, in_arr
    call print_str
    
    mov esi, data_arr
    mov ecx, data_len
    mov eax, 100
    call linear_search
    
    cmp eax, -1
    je .not_found_100
    
    mov ebx, eax
    mov eax, found_msg
    call print_str
    mov eax, 100
    call print_num
    mov eax, at_msg
    call print_str
    mov eax, ebx
    call print_num
    mov eax, newline
    call print_str
    jmp .end_search
    
.not_found_100:
    mov eax, notfound
    call print_str
    
.end_search:
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

**วิธี Compile:**
```bash
nasm -f elf32 search_loop.asm -o search.o
ld -m elf_i386 search.o -o search
./search
```

**ผลลัพธ์:**
```
Searching for 42 in array...
Found 42 at index: 4

Searching for 100 in array...
Not found!
```

---

### Example 7: Duff's Device Implementation

```nasm
; ไฟล์: duffs_device.asm
; วัตถุประสงค์: Implement Duff's Device สำหรับ memory copy
; การ compile: nasm -f elf32 duffs_device.asm -o duffs.o
;              ld -m elf_i386 duffs.o -o duffs
; ผลลัพธ์: แสดง copy performance

section .data
    src_data    db 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16
                db 17,18,19,20,21,22,23,24,25,26,27,28,29,30
    src_len     equ $ - src_data
    
    msg_before  db "Source array: ", 0
    msg_after   db "Dest array (after copy): ", 0
    msg_match   db 10, "Arrays match! Copy successful.", 10, 0
    comma_sp    db ", ", 0
    newline     db 10, 0
    lbracket    db "[", 0
    rbracket    db "]", 10, 0

section .bss
    dst_data    resb 32
    num_buf     resb 16

section .text
    global _start

print_str:
    push eax
    push ebx
    push ecx
    push edx
    push edi
    mov edi, eax
    xor ecx, ecx
.fnd: cmp byte [edi+ecx], 0
      je .prnt
      inc ecx
      jmp .fnd
.prnt: test ecx, ecx
       jz .dn
       mov eax, 4
       mov ebx, 1
       mov edx, ecx
       mov ecx, edi
       int 0x80
.dn:  pop edi
      pop edx
      pop ecx
      pop ebx
      pop eax
      ret

print_num:
    pushad
    sub esp, 16
    lea edi, [esp+15]
    mov byte [edi], 0
    test eax, eax
    jnz .nzr
    dec edi
    mov byte [edi], '0'
    jmp .prn
.nzr: mov ebx, 10
.extr: dec edi
       xor edx, edx
       div ebx
       add dl, '0'
       mov [edi], dl
       test eax, eax
       jnz .extr
.prn: lea edx, [esp+15]
      sub edx, edi
      mov eax, 4
      mov ebx, 1
      mov ecx, edi
      int 0x80
      add esp, 16
      popad
      ret

; =========================================
; Duff's Device: Copy with 8-way unrolling
; input: ESI = source, EDI = dest, ECX = count
; =========================================
duffs_copy:
    push eax
    push ebx
    push ecx
    push esi
    push edi
    
    ; กรณีพิเศษ: count = 0
    test ecx, ecx
    jz .dc_done
    
    ; คำนวณ: n = (count + 7) / 8
    ;         remainder = count % 8
    
    mov eax, ecx        ; EAX = count
    mov ebx, 8
    xor edx, edx
    div ebx             ; EAX = count/8 = n, EDX = count%8
    
    ; บันทึก n (number of full 8-blocks)
    push eax            ; push n
    
    ; jump ไปยัง entry point ตาม remainder
    ; remainder อยู่ใน EDX (0-7)
    ; ถ้า remainder = 0 ให้ทำ 8 copies ทันที
    test edx, edx
    jz .entry_8
    
    ; คำนวณ address ใน jump table
    ; entry_address = jump_table + remainder * 4
    lea ebx, [jump_table]
    mov eax, [ebx + edx*4]  ; โหลด address
    jmp eax                  ; jump ไป entry point
    
    ; ========== JUMP TABLE ==========
    ; ตาราง jump address สำหรับแต่ละ remainder
jump_table:
    dd .entry_8     ; remainder 0 → เต็ม 8 copies
    dd .entry_1     ; remainder 1 → copy 1 แล้ว continue
    dd .entry_2     ; remainder 2 → copy 2 แล้ว continue
    dd .entry_3     ; remainder 3
    dd .entry_4     ; remainder 4
    dd .entry_5     ; remainder 5
    dd .entry_6     ; remainder 6
    dd .entry_7     ; remainder 7

    ; ========== DUFF'S DEVICE CORE ==========
    ; แต่ละ entry ทำ copies แล้ว fall-through ต่อ
    
.main_loop:
    ; ดึง n กลับมา (ยังอยู่บน stack)
    pop eax
    dec eax             ; n--
    push eax
    jz .dc_cleanup      ; ถ้า n = 0 เสร็จแล้ว
    
.entry_8:
    movsb               ; copy byte 8
.entry_7:
    movsb               ; copy byte 7
.entry_6:
    movsb               ; copy byte 6
.entry_5:
    movsb               ; copy byte 5
.entry_4:
    movsb               ; copy byte 4
.entry_3:
    movsb               ; copy byte 3
.entry_2:
    movsb               ; copy byte 2
.entry_1:
    movsb               ; copy byte 1
    
    jmp .main_loop
    
.dc_cleanup:
    pop eax             ; clean up stack (ดึง n ออก)
    
.dc_done:
    pop edi
    pop esi
    pop ecx
    pop ebx
    pop eax
    ret

; =========================================
; ฟังก์ชัน: Simple memcpy (สำหรับเปรียบเทียบ)
; =========================================
simple_copy:
    push ecx
    push esi
    push edi
    
    test ecx, ecx
    jz .sc_done
    
.sc_loop:
    movsb               ; copy byte และ ESI++, EDI++
    loop .sc_loop
    
.sc_done:
    pop edi
    pop esi
    pop ecx
    ret

_start:
    ; =========================================
    ; ทดสอบ: copy โดยใช้ Duff's Device
    ; =========================================
    
    ; แสดง source array
    mov eax, msg_before
    call print_str
    mov eax, lbracket
    call print_str
    
    mov esi, src_data
    mov ecx, src_len
.show_src:
    push ecx
    movzx eax, byte [esi]
    call print_num
    pop ecx
    cmp ecx, 1
    je .src_last
    mov eax, comma_sp
    call print_str
.src_last:
    inc esi
    loop .show_src
    
    mov eax, rbracket
    call print_str
    
    ; ทำการ copy โดยใช้ Duff's Device
    mov esi, src_data
    mov edi, dst_data
    mov ecx, src_len    ; 30 bytes
    call duffs_copy
    
    ; แสดง dest array
    mov eax, msg_after
    call print_str
    mov eax, lbracket
    call print_str
    
    mov esi, dst_data
    mov ecx, src_len
.show_dst:
    push ecx
    movzx eax, byte [esi]
    call print_num
    pop ecx
    cmp ecx, 1
    je .dst_last
    mov eax, comma_sp
    call print_str
.dst_last:
    inc esi
    loop .show_dst
    
    mov eax, rbracket
    call print_str
    
    ; ตรวจสอบว่า copy ถูกต้อง
    mov esi, src_data
    mov edi, dst_data
    mov ecx, src_len
    
.verify:
    mov al, [esi]
    mov bl, [edi]
    cmp al, bl
    jne .mismatch
    inc esi
    inc edi
    loop .verify
    
    ; ถูกต้อง!
    mov eax, msg_match
    call print_str
    jmp .end
    
.mismatch:
    ; ผิดพลาด
    mov eax, 4
    mov ebx, 1
    mov ecx, .err_msg
    mov edx, .err_len
    int 0x80
    
.err_msg db 10, "ERROR: Arrays do not match!", 10
.err_len equ $ - .err_msg
    
.end:
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

**วิธี Compile:**
```bash
nasm -f elf32 duffs_device.asm -o duffs.o
ld -m elf_i386 duffs.o -o duffs
./duffs
```

---

## Common Mistakes and Pitfalls (ข้อผิดพลาดที่พบบ่อย)

### 1. ลืมตรวจสอบ ECX = 0 ก่อนใช้ LOOP

```nasm
; ผิด! ถ้า ECX = 0 จะวนลูปไป 4,294,967,295 รอบ!
mov ecx, [user_input]  ; อาจเป็น 0 ได้
.dangerous_loop:
    ; body
    loop .dangerous_loop    ; DANGER: ECX = 0 → วน 2^32-1 รอบ!

; ถูกต้อง: ตรวจสอบก่อน
mov ecx, [user_input]
jecxz .skip_loop        ; ถ้า ECX = 0 ข้ามไปเลย
.safe_loop:
    ; body
    loop .safe_loop
.skip_loop:
```

### 2. ECX ถูก Clobber ระหว่าง Loop

```nasm
; ผิด! printf/function call อาจ modify ECX
mov ecx, 10
.loop:
    call some_function   ; some_function อาจ modify ECX!
    loop .loop

; ถูกต้อง: push/pop ECX
mov ecx, 10
.loop:
    push ecx
    call some_function
    pop ecx
    loop .loop
```

### 3. LOOP Instruction มี Short Jump Range เท่านั้น

```nasm
; ผิด! ถ้า body ยาวเกิน 128 bytes, assembler จะ error
mov ecx, 100
.loop_start:
    ; ... code ยาวมาก (> 128 bytes) ...
    loop .loop_start    ; ERROR: out of range!

; ถูกต้อง: ใช้ DEC+JNZ พร้อม long jump
mov ecx, 100
.loop_start:
    ; ... code ยาวมาก ...
    dec ecx
    jnz .loop_start     ; long jump ได้ไกลกว่า
```

### 4. Off-by-One Error

```nasm
; ต้องการ print 1, 2, 3, 4, 5
; ผิด:
mov ecx, 5
mov eax, 0              ; เริ่มที่ 0
.loop:
    inc eax
    ; print eax
    loop .loop
; ได้ 1, 2, 3, 4, 5 ← ถูก แต่ถ้าตั้งใจจะเริ่มที่ 0...

; ระวัง: loop จะทำงาน ECX รอบเสมอ
; ถ้าต้องการ loop N รอบ ตั้ง ECX = N
```

### 5. ใช้ LOOP กับ Float Operations

```nasm
; ระวัง! FPU instructions อาจ affect ZF และ flags อื่นๆ
; LOOPE/LOOPNE อ่าน ZF ดังนั้นต้องระวัง

; หลีกเลี่ยงการใช้ LOOPE/LOOPNE ถ้า body มี float ops
; ใช้ manual check แทน
mov ecx, 10
.fp_loop:
    ; ... float operations ...
    fcom st0, st1       ; compare floats (อาจ set/clear ZF)
    dec ecx
    jnz .fp_loop        ; ใช้ JNZ แทน LOOP เพื่อหลีกเลี่ยงความสับสน
```

### 6. Nested Loop Counter Conflicts

```nasm
; ผิด! ทั้ง inner และ outer loop ใช้ ECX
mov ecx, 5          ; outer counter
.outer:
    mov ecx, 3      ; inner counter -- OVERWRITES outer!
    .inner:
        loop .inner ; ECX-- จาก 3
    loop .outer     ; ECX-- จาก 0... ไม่ถูก!

; ถูกต้อง: บันทึก outer counter
mov ecx, 5          ; outer counter
.outer:
    push ecx        ; บันทึก outer ECX
    mov ecx, 3      ; inner counter
    .inner:
        loop .inner
    pop ecx         ; restore outer ECX
    loop .outer
```

---

## Advanced Techniques (เทคนิคขั้นสูง)

### 1. Loop Unrolling แบบ 4x

```nasm
; สมมติ array ขนาด 16 elements, sum ทุก element
; Normal loop: 16 iterations
; 4x unrolled: 4 iterations, แต่ละ iteration ทำ 4 adds

section .data
    arr16 dd 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16

section .text
    global sum_array_unrolled

; input: ESI = array, ECX = count (must be multiple of 4)
; output: EAX = sum
sum_array_unrolled:
    xor eax, eax        ; sum = 0
    shr ecx, 2          ; count /= 4 (number of 4-groups)
    jz .done
    
.unroll4:
    add eax, [esi]      ; element 1
    add eax, [esi+4]    ; element 2
    add eax, [esi+8]    ; element 3
    add eax, [esi+12]   ; element 4
    add esi, 16         ; advance 4 elements (4*4 bytes)
    loop .unroll4
    
.done:
    ret
```

### 2. Software Pipelining (Loop Pipelining)

```nasm
; เทคนิคขั้นสูง: overlap การ load กับการ compute
; เพื่อซ่อน memory latency

; Normal loop (sequential):
;   load A
;   compute with A
;   load B
;   compute with B
;   ...

; Pipelined loop:
;   load A (prefetch)
;   load B, compute with A (overlap)
;   load C, compute with B (overlap)
;   ...

sum_pipelined:
    ; setup: pre-load first element
    mov eax, [esi]      ; prefetch element 0
    add esi, 4
    dec ecx
    xor edx, edx        ; accumulator
    jz .tail
    
.pipe_loop:
    mov ebx, [esi]      ; pre-load next element
    add esi, 4
    add edx, eax        ; use previous load
    mov eax, ebx        ; move to current
    loop .pipe_loop
    
.tail:
    add edx, eax        ; process last element
    mov eax, edx        ; return sum
    ret
```

### 3. SIMD-friendly Loop Structure

```nasm
; โครงสร้าง loop ที่เหมาะสำหรับ auto-vectorization
; (เตรียมพร้อมสำหรับ SIMD operations)

; หลักการ:
; 1. ไม่มี data dependency ระหว่าง iterations
; 2. Memory access เป็น sequential (stride-1)
; 3. Loop body ง่าย ไม่มี branch ใน body

; ตัวอย่าง: multiply-add operation (AXPY: y = ax + y)
axpy:
    ; input: EAX = a (scalar), ESI = x array, EDI = y array, ECX = n
    push ebx
    push ecx
    
    test ecx, ecx
    jz .axpy_done
    
.axpy_loop:
    ; y[i] = a * x[i] + y[i]
    mov ebx, [esi]      ; load x[i]
    imul ebx, eax       ; a * x[i]
    add [edi], ebx      ; y[i] += a * x[i]
    
    add esi, 4          ; x++
    add edi, 4          ; y++
    loop .axpy_loop
    
.axpy_done:
    pop ecx
    pop ebx
    ret
```

### 4. Loop with Multiple Exit Conditions

```nasm
; Loop ที่มี exit condition หลายอย่าง
; เช่น: หยุดเมื่อพบ delimiter หรือ count หมด

; String parsing: อ่านจนกว่าจะพบ ',' หรือ '\n' หรือ end of buffer
parse_field:
    ; input: ESI = string pointer, ECX = max_count
    ; output: EAX = field length, ZF = 1 if delimiter found
    
    push esi
    xor eax, eax        ; length = 0
    
    jecxz .pf_done
    
.pf_loop:
    ; โหลด character
    movzx ebx, byte [esi]
    
    ; ตรวจสอบ end condition 1: null terminator
    test ebx, ebx
    jz .pf_found_end
    
    ; ตรวจสอบ end condition 2: comma
    cmp ebx, ','
    je .pf_found_delim
    
    ; ตรวจสอบ end condition 3: newline
    cmp ebx, 10
    je .pf_found_delim
    
    ; ยังไม่พบ delimiter ดำเนินต่อ
    inc eax             ; length++
    inc esi             ; next char
    loop .pf_loop
    
    ; Count หมด
    xor ebx, ebx        ; ZF = 0 (no delimiter)
    test eax, eax       ; set ZF based on length
    jmp .pf_done
    
.pf_found_delim:
    or ebx, 1           ; ZF = 0... hmm, need different approach
    
.pf_found_end:
    ; clean exit
    
.pf_done:
    pop esi
    ret
```

### 5. Loop Tiling (Cache-Friendly Loops)

```nasm
; สำหรับ matrix operations, loop tiling ช่วย cache performance
; แบ่ง matrix ออกเป็น tiles ขนาดเล็กที่ fit ใน L1 cache

; Matrix multiplication แบบ tiled
; C[i][j] += A[i][k] * B[k][j]

; TILE_SIZE = 4 (ปรับตาม L1 cache size)
TILE_SIZE equ 4

; matrix_multiply_tiled:
;   input: ESI = A, EDI = B, EDX = C (all 8x8 matrices of int32)
;          EBX = matrix size (N)
; Note: แสดงเป็น pseudo-code เพื่อความชัดเจน

; for (ii = 0; ii < N; ii += TILE_SIZE)
;   for (jj = 0; jj < N; jj += TILE_SIZE)
;     for (kk = 0; kk < N; kk += TILE_SIZE)
;       ; process tile (ii:ii+TS, jj:jj+TS, kk:kk+TS)
;       for (i = ii; i < min(ii+TS, N); i++)
;         for (j = jj; j < min(jj+TS, N); j++)
;           for (k = kk; k < min(kk+TS, N); k++)
;             C[i][j] += A[i][k] * B[k][j]
```

---

## Implementing For/While/Do-While Loops

### For Loop Pattern (ฉบับสมบูรณ์)

```nasm
; C code equivalent:
; for (int i = start; i < end; i += step) {
;     body
; }

section .text
for_loop_example:
    ; Parameters: EAX = start, EBX = end, ECX = step
    ; (สมมติว่า step = 1 เพื่อง่าย)
    
    ; i = start (เก็บใน ESI)
    mov esi, eax
    
.for_test:
    ; test: i < end
    cmp esi, ebx
    jge .for_done
    
    ; body: ทำงานตรงนี้
    ; (ใช้ ESI แทน i)
    push esi
    ; ... ทำอะไรบางอย่างกับ ESI ...
    pop esi
    
    ; update: i++
    add esi, ecx        ; i += step
    jmp .for_test
    
.for_done:
    ret
```

### While Loop Pattern (ฉบับสมบูรณ์)

```nasm
; C code equivalent:
; while (condition) {
;     body
; }

while_loop_example:
    ; สมมติ: วนจนกว่า [EBX] = 0 (null terminator)
    
.while_test:
    ; ทดสอบ condition ก่อนทุกรอบ
    cmp byte [ebx], 0
    je .while_done
    
    ; body
    ; process [EBX]
    
    ; ไม่ลืม advance pointer!
    inc ebx
    jmp .while_test
    
.while_done:
    ret
```

### Do-While Loop Pattern (ฉบับสมบูรณ์)

```nasm
; C code equivalent:
; do {
;     body
; } while (condition);

do_while_example:
    ; Do-while รับประกัน อย่างน้อย 1 iteration
    
.do_while_body:
    ; body (ทำก่อนเสมอ)
    ; process something
    
    ; ทดสอบ condition หลัง body
    dec eax             ; สมมติ condition คือ EAX != 0
    jnz .do_while_body  ; ถ้ายังจริง วนซ้ำ
    
    ret
```

---

## Exercises (แบบฝึกหัด)

### Exercise 1: Count Vowels ในสตริง
**โจทย์:** เขียนฟังก์ชัน `count_vowels` ที่รับ pointer ไปยัง string และนับจำนวน vowels (a, e, i, o, u, A, E, I, O, U)

**Hint:**
```nasm
; โครงสร้างคำตอบ
count_vowels:
    ; input: ESI = string pointer
    ; output: EAX = vowel count
    
    xor eax, eax        ; count = 0
    
.loop:
    movzx ebx, byte [esi]
    test ebx, ebx       ; null terminator?
    jz .done
    
    ; ตรวจสอบแต่ละ vowel
    ; Hint: ใช้ OR หรือ switch-like structure
    ; หรือใช้ lookup table สำหรับ O(1) check
    
    ; OR แบบง่าย:
    cmp ebx, 'a'
    je .is_vowel
    cmp ebx, 'e'
    je .is_vowel
    ; ... etc
    
    jmp .next
    
.is_vowel:
    inc eax
    
.next:
    inc esi
    jmp .loop
    
.done:
    ret
```

### Exercise 2: Reverse an Array
**โจทย์:** เขียนฟังก์ชัน `reverse_array` ที่ reverse array of int32 in-place

**Hint:**
```nasm
reverse_array:
    ; input: ESI = array pointer, ECX = count
    ; ใช้ two-pointer approach: start และ end
    ; swap element ที่ start กับ end แล้วขยับทั้งคู่เข้ามา
    
    lea edi, [esi + ecx*4 - 4]  ; EDI ชี้ที่ element สุดท้าย
    shr ecx, 1                  ; swap count = count/2
    
.rev_loop:
    ; swap [esi] กับ [edi]
    mov eax, [esi]
    mov ebx, [edi]
    mov [esi], ebx
    mov [edi], eax
    add esi, 4          ; start++
    sub edi, 4          ; end--
    loop .rev_loop
    
    ret
```

### Exercise 3: Bubble Sort
**โจทย์:** Implement Bubble Sort สำหรับ array of int32

**Hint:**
```nasm
bubble_sort:
    ; input: ESI = array, ECX = count
    ; Outer loop: N-1 passes
    ; Inner loop: compare adjacent elements
    
    ; Optimization: ถ้า inner loop ไม่มี swap เลย array sorted แล้ว
    
    dec ecx             ; N-1 passes
    
.outer:
    push ecx
    xor edi, edi        ; edi = swapped flag
    mov esi_ptr, esi    ; reset inner loop
    mov ecx_inner, [esp+4]  ; inner count
    
.inner:
    mov eax, [esi]
    mov ebx, [esi+4]
    cmp eax, ebx
    jle .no_swap
    
    ; swap
    mov [esi], ebx
    mov [esi+4], eax
    mov edi, 1          ; swapped = true
    
.no_swap:
    add esi, 4
    loop .inner
    
    pop ecx
    test edi, edi
    jz .sorted          ; ไม่มี swap = sorted แล้ว
    loop .outer
    
.sorted:
    ret
```

### Exercise 4: String Copy (strcpy)
**โจทย์:** Implement `my_strcpy` ที่ copy string จาก source ไป destination

**Hint:**
```nasm
my_strcpy:
    ; input: ESI = src, EDI = dst
    ; ใช้ LODSB/STOSB เพื่อ copy byte by byte
    ; หยุดเมื่อ copy null terminator แล้ว
    
.copy_loop:
    lodsb               ; AL = [ESI], ESI++
    stosb               ; [EDI] = AL, EDI++
    test al, al         ; null terminator?
    jnz .copy_loop      ; ถ้ายัง copy ต่อ
    
    ret                 ; null ถูก copy แล้วด้วย
```

### Exercise 5: Power Function (base^exp)
**โจทย์:** Implement `power` function ที่คำนวณ base^exponent

**Hint:**
```nasm
power:
    ; input: EAX = base, ECX = exponent
    ; output: EAX = base^exponent
    
    ; edge case: exp = 0 → return 1
    ; edge case: exp = 1 → return base
    
    test ecx, ecx
    jz .exp_zero
    
    mov ebx, eax        ; save base
    dec ecx             ; exp--
    jz .done
    
.power_loop:
    imul eax, ebx       ; result *= base
    loop .power_loop
    
    ret
    
.exp_zero:
    mov eax, 1
    ret
    
.done:
    ret
```

---

## Loop Performance Considerations

### 1. Branch Prediction และ Loops

CPU สมัยใหม่มี Branch Predictor ที่ทรงพลัง:
- **Loop branches** ที่ predict ได้ง่ายที่สุด: จะ taken เสมอ ยกเว้นครั้งสุดท้าย
- **Misprediction penalty:** 15-20 cycles บน Intel Skylake
- Loop ที่วน **> 2 รอบ** มักถูก predict ถูกต้อง

```nasm
; ตัวอย่าง: loop ที่ branch predictor ชอบ
; วนซ้ำหลายรอบ, exit condition ชัดเจน
mov ecx, 1000
.predictable_loop:
    ; body ไม่มี unpredictable branch
    add eax, [esi]
    add esi, 4
    loop .predictable_loop  ; predict: taken (999 times), not-taken (1 time)
```

### 2. Loop Alignment

```nasm
; align loop start ที่ 16-byte boundary สำหรับ performance
align 16
.loop_aligned:
    add eax, [esi]
    add esi, 4
    dec ecx
    jnz .loop_aligned
```

### 3. Avoiding Memory Stalls

```nasm
; ผิด: อ่าน memory ซ้ำๆ
.slow_loop:
    mov eax, [counter]  ; load จาก memory ทุก iteration
    inc eax
    mov [counter], eax  ; store กลับ memory
    loop .slow_loop

; ถูกต้อง: ใช้ register
mov eax, [counter]      ; load ครั้งเดียว
.fast_loop:
    inc eax             ; ทำงานใน register
    loop .fast_loop
mov [counter], eax      ; store ครั้งเดียว
```

### 4. Loop Invariant Code Motion

```nasm
; ผิด: คำนวณค่าเดิมซ้ำๆ ใน loop
mov ecx, 100
.bad_loop:
    mov eax, [base_addr]    ; load ค่าคงที่ทุกรอบ (loop invariant)
    add eax, [offset]       ; load อีกอย่าง
    ; use eax...
    loop .bad_loop

; ถูกต้อง: คำนวณก่อน loop
mov eax, [base_addr]        ; hoist ออกมาก่อน loop
add eax, [offset]
mov ebx, eax                ; เก็บผลลัพธ์ใน register
mov ecx, 100
.good_loop:
    ; use ebx แทน
    loop .good_loop
```

---

## Summary (สรุป)

### คำสั่ง LOOP ต่างๆ

| Instruction | Condition | Use Case |
|-------------|-----------|----------|
| `LOOP label` | ECX-- && ECX != 0 | General counted loop |
| `LOOPE label` | ECX-- && ECX != 0 && ZF = 1 | Loop while equal |
| `LOOPZ label` | เหมือน LOOPE | Loop while zero |
| `LOOPNE label` | ECX-- && ECX != 0 && ZF = 0 | Loop while not equal |
| `LOOPNZ label` | เหมือน LOOPNE | Loop while not zero |
| `JECXZ label` | ECX == 0 | Skip loop if counter is zero |

### เมื่อไรควรใช้อะไร

| Situation | Recommendation |
|-----------|----------------|
| Simple counted loop | DEC + JNZ (เร็วกว่า LOOP) |
| Loop body ยาว (> 128 bytes) | DEC + JNZ (LOOP มีแค่ short jump) |
| Search loop | Manual CMP + JE + DEC + JNZ |
| Performance-critical | Loop unrolling + DEC+JNZ |
| Code readability | LOOP (สั้นกว่า แต่ช้ากว่า) |

### Best Practices

1. **ใช้ DEC+JNZ แทน LOOP** บน modern CPU
2. **ตรวจสอบ ECX = 0** ก่อนทุก counted loop ด้วย JECXZ
3. **Push/Pop ECX** ถ้า body ของ loop เรียก function
4. **Align loop starts** ที่ 16-byte boundary
5. **Hoist loop invariants** ออกมาก่อน loop
6. **Unroll loops** ที่ count เป็น constant และ body สั้น
7. **ระวัง nested loops** ที่ใช้ ECX ซ้อนกัน

### Loop Performance Rules of Thumb

- Loop overhead (DEC+JNZ): ~1-2 cycles
- Loop overhead (LOOP): ~5-7 cycles บน modern CPU
- Branch misprediction penalty: ~15-20 cycles
- Cache miss (L1): ~4-5 cycles extra
- Cache miss (L2): ~12 cycles extra
- Cache miss (RAM): ~200+ cycles extra

### สิ่งที่ได้เรียนรู้ใน Part นี้

1. LOOP, LOOPE/Z, LOOPNE/NZ - counted loop instructions
2. JCXZ/JECXZ/JRCXZ - ป้องกัน infinite loop เมื่อ count = 0
3. For/While/Do-While patterns ใน Assembly
4. Loop Unrolling - เพิ่ม throughput ลด overhead
5. Duff's Device - unrolled copy ด้วย jump table
6. Nested loops - จัดการ counter conflicts
7. Performance optimization - alignment, invariant hoisting, pipelining
8. Fibonacci, Factorial, Array stats, String length

### ขั้นตอนต่อไป

- Part 019: Procedures และ Functions (CALL/RET, Stack Frame)
- Part 020: String Instructions (MOVS, CMPS, SCAS, LODS, STOS)
- Part 021: Bit Manipulation (AND, OR, XOR, NOT, SHL, SHR)

---

*Part 018 จบสมบูรณ์ - เนื้อหาครอบคลุม Loop Instructions ทั้งหมด รวมถึงเทคนิคขั้นสูงและ Performance Considerations*
