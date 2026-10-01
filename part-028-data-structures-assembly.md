# Part 028: Data Structures ใน Assembly

## บทนำ

ในบทนี้เราจะเรียนรู้การสร้างและใช้งาน Data Structures พื้นฐานในภาษา Assembly
การเข้าใจ Data Structures ในระดับ Assembly จะช่วยให้เข้าใจการทำงานของหน่วยความจำและประสิทธิภาพของโปรแกรมได้อย่างลึกซึ้ง

---

## 1. Arrays (อาร์เรย์)

### 1.1 Static Arrays (อาร์เรย์แบบ Static)

Static Array คืออาร์เรย์ที่กำหนดขนาดไว้ตั้งแต่แรกใน .data หรือ .bss section

```nasm
; ไฟล์: static_array.asm
; การสร้างและใช้งาน Static Array ใน NASM
; คอมไพล์: nasm -f elf64 static_array.asm -o static_array.o
; ลิงก์:   ld static_array.o -o static_array

section .data
    ; อาร์เรย์ของจำนวนเต็ม 32-bit จำนวน 10 ตัว
    int_array   dd 10, 20, 30, 40, 50, 60, 70, 80, 90, 100
    array_size  equ 10                      ; ขนาดของอาร์เรย์

    ; อาร์เรย์ของ byte
    byte_array  db 1, 2, 3, 4, 5, 6, 7, 8
    
    ; ข้อความสำหรับแสดงผล
    msg_sum     db "Sum = ", 0
    msg_nl      db 10, 0
    
section .bss
    ; จองพื้นที่สำหรับอาร์เรย์ที่ยังไม่กำหนดค่า
    result_buf  resb 20                     ; จอง 20 bytes

section .text
    global _start

; ฟังก์ชันแปลง integer เป็น string
int_to_str:
    ; rdi = จำนวนที่ต้องการแปลง
    ; rsi = buffer สำหรับเก็บผลลัพธ์
    push rbp
    mov rbp, rsp
    push rbx
    push rcx
    push rdx
    push rdi
    push rsi
    
    mov rax, rdi                            ; ค่าที่ต้องการแปลง
    mov rbx, 10                             ; base 10
    mov rcx, 0                              ; นับจำนวนหลัก
    
    ; กรณีที่เป็น 0
    test rax, rax
    jnz .convert_loop
    mov byte [rsi], '0'
    mov byte [rsi+1], 0
    jmp .done
    
.convert_loop:
    test rax, rax
    jz .reverse
    xor rdx, rdx
    div rbx                                 ; rax = quotient, rdx = remainder
    add rdx, '0'                            ; แปลงเป็น ASCII
    push rdx                                ; เก็บ digit ไว้ใน stack
    inc rcx
    jmp .convert_loop
    
.reverse:
    ; นำ digit จาก stack มาเรียงลำดับ
    mov rdi, rsi                            ; pointer ไปยัง buffer
.pop_loop:
    test rcx, rcx
    jz .null_term
    pop rdx
    mov [rdi], dl
    inc rdi
    dec rcx
    jmp .pop_loop
    
.null_term:
    mov byte [rdi], 0
    
.done:
    pop rsi
    pop rdi
    pop rdx
    pop rcx
    pop rbx
    pop rbp
    ret

; ฟังก์ชันพิมพ์ string
print_str:
    ; rdi = pointer ไปยัง string
    push rsi
    push rdx
    push rax
    push rdi
    
    mov rsi, rdi
    xor rdx, rdx
.len_loop:
    cmp byte [rsi+rdx], 0
    je .print
    inc rdx
    jmp .len_loop
.print:
    mov rax, 1                              ; sys_write
    mov rdi, 1                              ; stdout
    syscall
    
    pop rdi
    pop rax
    pop rdx
    pop rsi
    ret

_start:
    ; คำนวณผลรวมของ int_array
    xor rax, rax                            ; ล้าง accumulator
    xor rcx, rcx                            ; index = 0
    lea rsi, [int_array]                    ; pointer ไปยังอาร์เรย์
    
.sum_loop:
    cmp rcx, array_size                     ; ตรวจสอบว่าถึงขอบเขตหรือยัง
    jge .print_sum
    
    mov ebx, [rsi + rcx*4]                  ; โหลด element ที่ rcx (int = 4 bytes)
    add eax, ebx                            ; บวกเข้าไปใน accumulator
    inc rcx
    jmp .sum_loop
    
.print_sum:
    ; แสดงผล "Sum = "
    mov rdi, msg_sum
    call print_str
    
    ; แปลงผลรวมเป็น string และแสดงผล
    mov rdi, rax
    mov rsi, result_buf
    call int_to_str
    
    mov rdi, result_buf
    call print_str
    
    ; ขึ้นบรรทัดใหม่
    mov rdi, msg_nl
    call print_str
    
    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

### 1.2 Dynamic Memory Allocation สำหรับ Arrays

```nasm
; ไฟล์: dynamic_array.asm
; การจัดสรรหน่วยความจำแบบ Dynamic สำหรับ Array
; คอมไพล์: nasm -f elf64 dynamic_array.asm -o dynamic_array.o
; ลิงก์:   ld dynamic_array.o -o dynamic_array

section .data
    ; ขนาดของ array ที่ต้องการ
    array_capacity  dq 100                  ; ความจุสูงสุด (จำนวน elements)
    element_size    equ 8                   ; ขนาดของแต่ละ element (8 bytes = int64)

section .bss
    array_ptr       resq 1                  ; pointer ไปยัง dynamic array
    array_count     resq 1                  ; จำนวน elements ปัจจุบัน

section .text
    global _start

; ฟังก์ชัน: allocate_array
; จัดสรรหน่วยความจำสำหรับ dynamic array
; Input:  rdi = จำนวน elements ที่ต้องการ
; Output: rax = pointer ไปยังหน่วยความจำที่จัดสรร (หรือ -1 ถ้าล้มเหลว)
allocate_array:
    push rbp
    mov rbp, rsp
    
    ; คำนวณขนาดที่ต้องการ (elements * element_size)
    imul rdi, element_size                  ; rdi = total bytes needed
    
    ; เรียก mmap เพื่อจัดสรรหน่วยความจำ
    mov rax, 9                              ; sys_mmap
    xor rsi, rsi                            ; addr = NULL (kernel เลือก)
    mov rdx, 3                              ; PROT_READ | PROT_WRITE
    mov r10, 0x22                           ; MAP_PRIVATE | MAP_ANONYMOUS
    mov r8, -1                              ; fd = -1 (anonymous)
    xor r9, r9                              ; offset = 0
    syscall
    
    ; ตรวจสอบความสำเร็จ
    cmp rax, -1
    je .alloc_failed
    
    ; เก็บ pointer
    mov [array_ptr], rax
    mov qword [array_count], 0              ; จำนวน elements = 0
    
    pop rbp
    ret
    
.alloc_failed:
    mov rax, -1
    pop rbp
    ret

; ฟังก์ชัน: array_push
; เพิ่ม element เข้าไปใน dynamic array
; Input:  rdi = ค่าที่ต้องการเพิ่ม
; Output: rax = 0 ถ้าสำเร็จ, -1 ถ้าเต็ม
array_push:
    push rbp
    mov rbp, rsp
    push rbx
    
    ; ตรวจสอบว่าเต็มหรือยัง
    mov rax, [array_count]
    cmp rax, [array_capacity]
    jge .array_full
    
    ; คำนวณ offset ของ element ใหม่
    mov rbx, [array_ptr]                    ; base address
    imul rax, element_size                  ; offset = count * element_size
    add rbx, rax                            ; address = base + offset
    
    ; เก็บค่า
    mov [rbx], rdi
    
    ; เพิ่ม count
    inc qword [array_count]
    
    xor rax, rax                            ; return 0 (success)
    pop rbx
    pop rbp
    ret
    
.array_full:
    mov rax, -1                             ; return -1 (failure)
    pop rbx
    pop rbp
    ret

; ฟังก์ชัน: array_get
; อ่านค่า element จาก index ที่กำหนด
; Input:  rdi = index
; Output: rax = ค่าที่ index นั้น (หรือ -1 ถ้า index ไม่ถูกต้อง)
array_get:
    push rbp
    mov rbp, rsp
    
    ; ตรวจสอบว่า index ถูกต้องหรือไม่
    cmp rdi, 0
    jl .invalid_index
    cmp rdi, [array_count]
    jge .invalid_index
    
    ; คำนวณ address
    imul rdi, element_size
    mov rax, [array_ptr]
    add rax, rdi
    mov rax, [rax]                          ; อ่านค่า
    
    pop rbp
    ret
    
.invalid_index:
    mov rax, -1
    pop rbp
    ret

_start:
    ; จัดสรรหน่วยความจำสำหรับ 100 elements
    mov rdi, 100
    call allocate_array
    
    ; เพิ่มค่าเข้าไปใน array
    mov rdi, 42
    call array_push
    
    mov rdi, 100
    call array_push
    
    mov rdi, 255
    call array_push
    
    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

### 1.3 2D Arrays (อาร์เรย์ 2 มิติ)

```nasm
; ไฟล์: 2d_array.asm
; การสร้างและใช้งาน 2D Array ใน NASM
; Row-major vs Column-major ordering
; คอมไพล์: nasm -f elf64 2d_array.asm -o 2d_array.o

section .data
    ; Matrix 3x4 แบบ row-major order
    ; [row0col0, row0col1, row0col2, row0col3,
    ;  row1col0, row1col1, row1col2, row1col3,
    ;  row2col0, row2col1, row2col2, row2col3]
    matrix_3x4  dd  1,  2,  3,  4,   \
                    5,  6,  7,  8,   \
                    9, 10, 11, 12
    
    ROWS        equ 3
    COLS        equ 4
    ELEM_SIZE   equ 4                       ; 4 bytes per int32

section .text
    global _start

; ฟังก์ชัน: matrix_get_row_major
; อ่านค่าจาก 2D matrix (row-major order)
; Input:  rdi = base address ของ matrix
;         rsi = row index
;         rdx = column index
;         rcx = จำนวน columns
; Output: eax = ค่าที่ตำแหน่ง [row][col]
matrix_get_row_major:
    ; address = base + (row * num_cols + col) * elem_size
    push rbp
    mov rbp, rsp
    
    ; คำนวณ offset
    imul rsi, rcx                           ; row * num_cols
    add rsi, rdx                            ; + col
    imul rsi, ELEM_SIZE                     ; * element_size
    
    ; อ่านค่า
    mov eax, [rdi + rsi]
    
    pop rbp
    ret

; ฟังก์ชัน: matrix_set_row_major
; เขียนค่าลงใน 2D matrix (row-major order)
; Input:  rdi = base address ของ matrix
;         rsi = row index
;         rdx = column index
;         rcx = จำนวน columns
;         r8d = ค่าที่ต้องการเขียน
matrix_set_row_major:
    push rbp
    mov rbp, rsp
    
    ; คำนวณ offset
    imul rsi, rcx                           ; row * num_cols
    add rsi, rdx                            ; + col
    imul rsi, ELEM_SIZE                     ; * element_size
    
    ; เขียนค่า
    mov [rdi + rsi], r8d
    
    pop rbp
    ret

; ฟังก์ชัน: transpose_matrix
; สลับแถวและคอลัมน์ของ matrix (in-place สำหรับ square matrix)
; Input:  rdi = base address ของ matrix
;         rsi = ขนาด n ของ matrix n x n
transpose_matrix:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    
    mov r12, rdi                            ; base address
    mov r13, rsi                            ; n
    
    xor rbx, rbx                            ; i = 0
.outer_loop:
    cmp rbx, r13
    jge .done
    
    lea r14, [rbx+1]                        ; j = i + 1
.inner_loop:
    cmp r14, r13
    jge .next_row
    
    ; สลับ [i][j] กับ [j][i]
    ; คำนวณ address ของ [i][j]
    mov rax, rbx
    imul rax, r13
    add rax, r14
    imul rax, ELEM_SIZE
    
    ; คำนวณ address ของ [j][i]
    mov rcx, r14
    imul rcx, r13
    add rcx, rbx
    imul rcx, ELEM_SIZE
    
    ; สลับค่า
    mov edx, [r12 + rax]
    mov r8d, [r12 + rcx]
    mov [r12 + rax], r8d
    mov [r12 + rcx], edx
    
    inc r14
    jmp .inner_loop
    
.next_row:
    inc rbx
    jmp .outer_loop
    
.done:
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

_start:
    ; ทดสอบการเข้าถึง matrix
    ; อ่านค่าที่ matrix[1][2] = 7
    lea rdi, [matrix_3x4]
    mov rsi, 1                              ; row 1
    mov rdx, 2                              ; col 2
    mov rcx, COLS                           ; จำนวน columns = 4
    call matrix_get_row_major
    ; rax = 7
    
    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 2. Linked Lists (ลิสต์เชื่อมโยง)

### 2.1 Singly Linked List

```nasm
; ไฟล์: linked_list.asm
; การสร้างและใช้งาน Singly Linked List ใน NASM
; คอมไพล์: nasm -f elf64 linked_list.asm -o linked_list.o
; ลิงก์:   ld linked_list.o -o linked_list

; โครงสร้าง Node:
; offset 0: data (8 bytes - int64)
; offset 8: next pointer (8 bytes - address)
; รวม: 16 bytes per node

NODE_DATA   equ 0                           ; offset ของ data field
NODE_NEXT   equ 8                           ; offset ของ next pointer
NODE_SIZE   equ 16                          ; ขนาดของ node

section .data
    head_ptr    dq 0                        ; pointer ไปยัง head node (null = 0)
    
    msg_empty   db "List is empty", 10, 0
    msg_val     db "Value: ", 0
    msg_nl      db 10, 0
    msg_arrow   db " -> ", 0
    msg_null    db "NULL", 10, 0

section .bss
    print_buf   resb 32

section .text
    global _start

; ฟังก์ชัน: allocate_node
; จัดสรรหน่วยความจำสำหรับ node ใหม่
; Input:  rdi = ค่า data ของ node
; Output: rax = pointer ไปยัง node ใหม่
allocate_node:
    push rbp
    mov rbp, rsp
    push rdi                                ; เก็บ data ไว้ก่อน
    
    ; จัดสรรหน่วยความจำ 16 bytes
    mov rax, 9                              ; sys_mmap
    xor rdi, rdi                            ; addr = NULL
    mov rsi, NODE_SIZE                      ; size = 16 bytes
    mov rdx, 3                              ; PROT_READ | PROT_WRITE
    mov r10, 0x22                           ; MAP_PRIVATE | MAP_ANONYMOUS
    mov r8, -1                              ; fd = -1
    xor r9, r9                              ; offset = 0
    syscall
    
    ; ตรวจสอบความสำเร็จ
    cmp rax, -1
    je .alloc_fail
    
    ; กำหนดค่า node
    pop rdi                                 ; คืน data
    mov [rax + NODE_DATA], rdi              ; เก็บ data
    mov qword [rax + NODE_NEXT], 0          ; next = NULL
    
    pop rbp
    ret
    
.alloc_fail:
    pop rdi
    mov rax, 0                              ; return NULL
    pop rbp
    ret

; ฟังก์ชัน: list_push_front
; เพิ่ม node ไปที่ด้านหน้าของ list
; Input:  rdi = ค่าที่ต้องการเพิ่ม
list_push_front:
    push rbp
    mov rbp, rsp
    push rdi
    
    ; สร้าง node ใหม่
    call allocate_node
    test rax, rax                           ; ตรวจสอบว่าสำเร็จหรือไม่
    jz .push_fail
    
    ; เชื่อม node ใหม่กับ head เดิม
    mov rcx, [head_ptr]                     ; โหลด head เดิม
    mov [rax + NODE_NEXT], rcx              ; new_node->next = old_head
    mov [head_ptr], rax                     ; head = new_node
    
.push_fail:
    pop rdi
    pop rbp
    ret

; ฟังก์ชัน: list_push_back
; เพิ่ม node ไปที่ด้านหลังของ list
; Input:  rdi = ค่าที่ต้องการเพิ่ม
list_push_back:
    push rbp
    mov rbp, rsp
    push rdi
    push rbx
    
    ; สร้าง node ใหม่
    call allocate_node
    test rax, rax
    jz .push_back_fail
    
    mov rbx, rax                            ; เก็บ pointer ของ new node
    
    ; ตรวจสอบว่า list ว่างหรือไม่
    mov rax, [head_ptr]
    test rax, rax
    jz .empty_list
    
    ; วิ่งไปหา tail node
.find_tail:
    mov rcx, [rax + NODE_NEXT]              ; โหลด next pointer
    test rcx, rcx
    jz .found_tail
    mov rax, rcx                            ; เดินไปยัง next node
    jmp .find_tail
    
.found_tail:
    ; เชื่อม tail กับ new node
    mov [rax + NODE_NEXT], rbx
    jmp .push_back_done
    
.empty_list:
    ; list ว่าง ตั้ง head เป็น new node
    mov [head_ptr], rbx
    
.push_back_done:
.push_back_fail:
    pop rbx
    pop rdi
    pop rbp
    ret

; ฟังก์ชัน: list_pop_front
; ลบ node แรกออกจาก list
; Output: rax = ค่าของ node ที่ถูกลบ (หรือ -1 ถ้า list ว่าง)
list_pop_front:
    push rbp
    mov rbp, rsp
    
    ; ตรวจสอบว่า list ว่างหรือไม่
    mov rax, [head_ptr]
    test rax, rax
    jz .list_empty
    
    ; เก็บ data ของ head
    mov rcx, [rax + NODE_DATA]
    
    ; อัพเดท head
    mov rdx, [rax + NODE_NEXT]
    mov [head_ptr], rdx
    
    ; คืนหน่วยความจำของ node เดิม (munmap)
    push rcx
    mov rdi, rax                            ; addr = node address
    mov rsi, NODE_SIZE                      ; length = 16
    mov rax, 11                             ; sys_munmap
    syscall
    pop rcx
    
    mov rax, rcx                            ; return data value
    pop rbp
    ret
    
.list_empty:
    mov rax, -1
    pop rbp
    ret

; ฟังก์ชัน: list_search
; ค้นหาค่าใน list
; Input:  rdi = ค่าที่ต้องการค้นหา
; Output: rax = pointer ไปยัง node (หรือ 0 ถ้าไม่พบ)
list_search:
    push rbp
    mov rbp, rsp
    
    mov rax, [head_ptr]                     ; เริ่มจาก head
    
.search_loop:
    test rax, rax                           ; ตรวจสอบว่าถึง NULL หรือยัง
    jz .not_found
    
    cmp [rax + NODE_DATA], rdi              ; เปรียบเทียบค่า
    je .found
    
    mov rax, [rax + NODE_NEXT]              ; เดินไปยัง next node
    jmp .search_loop
    
.found:
    pop rbp
    ret
    
.not_found:
    xor rax, rax                            ; return NULL
    pop rbp
    ret

; ฟังก์ชัน: list_delete
; ลบ node ที่มีค่าตรงกับที่ระบุ
; Input:  rdi = ค่าที่ต้องการลบ
; Output: rax = 0 ถ้าสำเร็จ, -1 ถ้าไม่พบ
list_delete:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    
    mov r12, rdi                            ; เก็บค่าที่ต้องการลบ
    
    ; ตรวจสอบว่า list ว่างหรือไม่
    mov rbx, [head_ptr]
    test rbx, rbx
    jz .delete_not_found
    
    ; ตรวจสอบว่า head node คือค่าที่ต้องการลบ
    cmp [rbx + NODE_DATA], r12
    jne .search_node
    
    ; ลบ head node
    mov rax, [rbx + NODE_NEXT]
    mov [head_ptr], rax
    
    ; คืนหน่วยความจำ
    mov rdi, rbx
    mov rsi, NODE_SIZE
    mov rax, 11                             ; sys_munmap
    syscall
    
    xor rax, rax
    jmp .delete_done
    
.search_node:
    ; ค้นหา node ก่อน node ที่ต้องการลบ
    mov rax, rbx                            ; prev = head
    
.search_loop:
    mov rcx, [rax + NODE_NEXT]              ; curr = prev->next
    test rcx, rcx
    jz .delete_not_found
    
    cmp [rcx + NODE_DATA], r12
    je .delete_node
    
    mov rax, rcx                            ; prev = curr
    jmp .search_loop
    
.delete_node:
    ; ลบ curr (prev->next = curr->next)
    mov rdx, [rcx + NODE_NEXT]
    mov [rax + NODE_NEXT], rdx
    
    ; คืนหน่วยความจำ
    push rax
    mov rdi, rcx
    mov rsi, NODE_SIZE
    mov rax, 11                             ; sys_munmap
    syscall
    pop rax
    
    xor rax, rax
    jmp .delete_done
    
.delete_not_found:
    mov rax, -1
    
.delete_done:
    pop r12
    pop rbx
    pop rbp
    ret

_start:
    ; สร้าง linked list: 1 -> 2 -> 3 -> 4 -> 5
    mov rdi, 1
    call list_push_back
    
    mov rdi, 2
    call list_push_back
    
    mov rdi, 3
    call list_push_back
    
    mov rdi, 4
    call list_push_back
    
    mov rdi, 5
    call list_push_back
    
    ; ลบค่า 3
    mov rdi, 3
    call list_delete
    
    ; list ตอนนี้: 1 -> 2 -> 4 -> 5
    
    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 3. Stack (สแตก) ด้วย Array

```nasm
; ไฟล์: stack_array.asm
; การสร้าง Stack โดยใช้ Array ใน NASM
; คอมไพล์: nasm -f elf64 stack_array.asm -o stack_array.o
; ลิงก์:   ld stack_array.o -o stack_array

STACK_MAX   equ 100                         ; ขนาดสูงสุดของ stack

section .data
    msg_push    db "Push: ", 0
    msg_pop     db "Pop: ", 0
    msg_top     db "Top: ", 0
    msg_empty   db "Stack is empty!", 10, 0
    msg_full    db "Stack is full!", 10, 0
    msg_nl      db 10, 0
    msg_sp      db " ", 0

section .bss
    ; โครงสร้าง stack
    stack_data  resq STACK_MAX              ; พื้นที่เก็บข้อมูล (100 * 8 bytes)
    stack_top   resq 1                      ; index ของ top element (-1 = empty)
    print_buf   resb 32

section .text
    global _start

; ฟังก์ชัน: stack_init
; เริ่มต้น stack
stack_init:
    mov qword [stack_top], -1               ; top = -1 (empty)
    ret

; ฟังก์ชัน: stack_push
; Push ค่าเข้าไปใน stack
; Input:  rdi = ค่าที่ต้องการ push
; Output: rax = 0 ถ้าสำเร็จ, -1 ถ้า stack เต็ม
stack_push:
    push rbp
    mov rbp, rsp
    
    ; ตรวจสอบว่า stack เต็มหรือไม่
    mov rax, [stack_top]
    cmp rax, STACK_MAX - 1
    jge .push_full
    
    ; เพิ่ม top
    inc rax
    mov [stack_top], rax
    
    ; เก็บค่า
    lea rcx, [stack_data]
    mov [rcx + rax*8], rdi                  ; stack_data[top] = value
    
    xor rax, rax                            ; return 0 (success)
    pop rbp
    ret
    
.push_full:
    mov rax, -1                             ; return -1 (full)
    pop rbp
    ret

; ฟังก์ชัน: stack_pop
; Pop ค่าออกจาก stack
; Output: rax = ค่าที่ pop ออกมา (หรือ -1 ถ้า empty)
stack_pop:
    push rbp
    mov rbp, rsp
    
    ; ตรวจสอบว่า stack ว่างหรือไม่
    mov rax, [stack_top]
    cmp rax, -1
    jle .pop_empty
    
    ; อ่านค่าจาก top
    lea rcx, [stack_data]
    mov rax, [rcx + rax*8]                  ; value = stack_data[top]
    
    ; ลด top
    dec qword [stack_top]
    
    pop rbp
    ret
    
.pop_empty:
    mov rax, -1                             ; return -1 (empty)
    pop rbp
    ret

; ฟังก์ชัน: stack_peek
; ดูค่า top โดยไม่ pop
; Output: rax = ค่า top (หรือ -1 ถ้า empty)
stack_peek:
    mov rax, [stack_top]
    cmp rax, -1
    jle .peek_empty
    
    lea rcx, [stack_data]
    mov rax, [rcx + rax*8]
    ret
    
.peek_empty:
    mov rax, -1
    ret

; ฟังก์ชัน: stack_is_empty
; ตรวจสอบว่า stack ว่างหรือไม่
; Output: rax = 1 ถ้าว่าง, 0 ถ้าไม่ว่าง
stack_is_empty:
    mov rax, [stack_top]
    cmp rax, -1
    sete al                                 ; al = 1 ถ้า top == -1
    movzx rax, al
    ret

; ฟังก์ชัน: stack_size
; คืนค่าจำนวน elements ใน stack
; Output: rax = จำนวน elements
stack_size:
    mov rax, [stack_top]
    inc rax                                 ; top + 1 = count (เพราะ 0-indexed)
    ret

_start:
    ; เริ่มต้น stack
    call stack_init
    
    ; Push ค่าเข้า stack
    mov rdi, 10
    call stack_push
    
    mov rdi, 20
    call stack_push
    
    mov rdi, 30
    call stack_push
    
    ; ดู top (ควรเป็น 30)
    call stack_peek
    ; rax = 30
    
    ; Pop ค่า (ควรได้ 30, 20, 10 ตามลำดับ)
    call stack_pop     ; rax = 30
    call stack_pop     ; rax = 20
    call stack_pop     ; rax = 10
    
    ; ตรวจสอบว่า stack ว่าง
    call stack_is_empty
    ; rax = 1 (ว่าง)
    
    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 4. Queue: Circular Buffer (คิวแบบ Circular)

```nasm
; ไฟล์: circular_queue.asm
; การสร้าง Queue แบบ Circular Buffer ใน NASM
; คอมไพล์: nasm -f elf64 circular_queue.asm -o circular_queue.o

QUEUE_MAX   equ 8                           ; ขนาดของ circular buffer (ควรเป็น power of 2)
QUEUE_MASK  equ QUEUE_MAX - 1              ; mask สำหรับการ wrap-around

section .bss
    ; โครงสร้าง circular queue
    queue_data  resq QUEUE_MAX              ; buffer
    queue_head  resq 1                      ; index สำหรับ dequeue (front)
    queue_tail  resq 1                      ; index สำหรับ enqueue (back)
    queue_count resq 1                      ; จำนวน elements

section .text
    global _start

; ฟังก์ชัน: queue_init
; เริ่มต้น circular queue
queue_init:
    mov qword [queue_head], 0
    mov qword [queue_tail], 0
    mov qword [queue_count], 0
    ret

; ฟังก์ชัน: queue_enqueue
; เพิ่มค่าเข้าไปใน queue
; Input:  rdi = ค่าที่ต้องการเพิ่ม
; Output: rax = 0 ถ้าสำเร็จ, -1 ถ้า queue เต็ม
queue_enqueue:
    push rbp
    mov rbp, rsp
    
    ; ตรวจสอบว่า queue เต็มหรือไม่
    mov rax, [queue_count]
    cmp rax, QUEUE_MAX
    jge .enqueue_full
    
    ; เก็บค่าที่ tail
    mov rcx, [queue_tail]
    lea rdx, [queue_data]
    mov [rdx + rcx*8], rdi
    
    ; เลื่อน tail ไปข้างหน้า (wrap-around ด้วย AND mask)
    inc rcx
    and rcx, QUEUE_MASK
    mov [queue_tail], rcx
    
    ; เพิ่ม count
    inc qword [queue_count]
    
    xor rax, rax
    pop rbp
    ret
    
.enqueue_full:
    mov rax, -1
    pop rbp
    ret

; ฟังก์ชัน: queue_dequeue
; ลบและคืนค่าจาก queue
; Output: rax = ค่าที่ dequeue (หรือ -1 ถ้า empty)
queue_dequeue:
    push rbp
    mov rbp, rsp
    
    ; ตรวจสอบว่า queue ว่างหรือไม่
    mov rax, [queue_count]
    test rax, rax
    jz .dequeue_empty
    
    ; อ่านค่าจาก head
    mov rcx, [queue_head]
    lea rdx, [queue_data]
    mov rax, [rdx + rcx*8]
    
    ; เลื่อน head ไปข้างหน้า (wrap-around)
    inc rcx
    and rcx, QUEUE_MASK
    mov [queue_head], rcx
    
    ; ลด count
    dec qword [queue_count]
    
    pop rbp
    ret
    
.dequeue_empty:
    mov rax, -1
    pop rbp
    ret

; ฟังก์ชัน: queue_peek_front
; ดูค่า front โดยไม่ dequeue
; Output: rax = ค่า front (หรือ -1 ถ้า empty)
queue_peek_front:
    mov rax, [queue_count]
    test rax, rax
    jz .peek_empty
    
    mov rcx, [queue_head]
    lea rdx, [queue_data]
    mov rax, [rdx + rcx*8]
    ret
    
.peek_empty:
    mov rax, -1
    ret

_start:
    ; เริ่มต้น queue
    call queue_init
    
    ; เพิ่มค่าเข้า queue: 1, 2, 3, 4, 5
    mov rdi, 1
    call queue_enqueue
    mov rdi, 2
    call queue_enqueue
    mov rdi, 3
    call queue_enqueue
    mov rdi, 4
    call queue_enqueue
    mov rdi, 5
    call queue_enqueue
    
    ; Dequeue ค่า (ควรได้ 1, 2, 3 ตามลำดับ FIFO)
    call queue_dequeue     ; rax = 1
    call queue_dequeue     ; rax = 2
    call queue_dequeue     ; rax = 3
    
    ; เพิ่มค่าใหม่ (circular wrap-around)
    mov rdi, 6
    call queue_enqueue
    mov rdi, 7
    call queue_enqueue
    
    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 5. Hash Table (ตารางแฮช)

```nasm
; ไฟล์: hash_table.asm
; การสร้าง Hash Table แบบ Open Addressing ใน NASM
; คอมไพล์: nasm -f elf64 hash_table.asm -o hash_table.o

; โครงสร้าง Hash Table Entry:
; offset 0: key (8 bytes)
; offset 8: value (8 bytes)
; offset 16: used flag (8 bytes: 0=empty, 1=occupied, 2=deleted)
; รวม: 24 bytes per entry

HASH_SIZE       equ 16                      ; ขนาดของตาราง (ควรเป็น power of 2)
HASH_MASK       equ HASH_SIZE - 1
ENTRY_SIZE      equ 24                      ; ขนาดของแต่ละ entry
ENTRY_KEY       equ 0
ENTRY_VALUE     equ 8
ENTRY_USED      equ 16

SLOT_EMPTY      equ 0
SLOT_OCCUPIED   equ 1
SLOT_DELETED    equ 2

section .bss
    hash_table  resb HASH_SIZE * ENTRY_SIZE ; พื้นที่สำหรับ hash table

section .text
    global _start

; ฟังก์ชัน: hash_function
; คำนวณค่า hash ของ key
; Input:  rdi = key (integer)
; Output: rax = hash value (0 ถึง HASH_SIZE-1)
hash_function:
    ; ใช้ Fibonacci hashing
    mov rax, rdi
    imul rax, 0x9e3779b97f4a7c15           ; Fibonacci multiplier
    shr rax, 60                             ; เอาแค่ 4 bits สูงสุด (log2(16) = 4)
    and rax, HASH_MASK
    ret

; ฟังก์ชัน: hash_insert
; เพิ่ม key-value pair เข้าไปใน hash table
; Input:  rdi = key, rsi = value
; Output: rax = 0 ถ้าสำเร็จ, -1 ถ้าเต็ม
hash_insert:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    
    mov r12, rdi                            ; เก็บ key
    mov r13, rsi                            ; เก็บ value
    
    ; คำนวณ hash
    call hash_function
    mov r14, rax                            ; เก็บ initial hash
    
    ; Linear probing
    xor rbx, rbx                            ; probe_count = 0
    
.probe_loop:
    cmp rbx, HASH_SIZE
    jge .insert_full                        ; ค้นหาครบทุก slot แล้ว
    
    ; คำนวณ address ของ slot
    mov rcx, r14
    imul rcx, ENTRY_SIZE
    lea rdx, [hash_table]
    add rdx, rcx                            ; rdx = address ของ slot ปัจจุบัน
    
    ; ตรวจสอบ slot
    mov rax, [rdx + ENTRY_USED]
    
    cmp rax, SLOT_EMPTY
    je .found_slot
    
    cmp rax, SLOT_DELETED
    je .found_slot
    
    ; ตรวจสอบว่า key ซ้ำหรือไม่ (update)
    cmp [rdx + ENTRY_KEY], r12
    je .update_value
    
    ; เลื่อนไป slot ถัดไป (linear probing)
    inc r14
    and r14, HASH_MASK
    inc rbx
    jmp .probe_loop
    
.found_slot:
    ; เพิ่มข้อมูลลงใน slot
    mov [rdx + ENTRY_KEY], r12
    mov [rdx + ENTRY_VALUE], r13
    mov qword [rdx + ENTRY_USED], SLOT_OCCUPIED
    xor rax, rax
    jmp .insert_done
    
.update_value:
    ; อัพเดท value ถ้า key ซ้ำ
    mov [rdx + ENTRY_VALUE], r13
    xor rax, rax
    jmp .insert_done
    
.insert_full:
    mov rax, -1
    
.insert_done:
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

; ฟังก์ชัน: hash_lookup
; ค้นหา value จาก key
; Input:  rdi = key
; Output: rax = value (หรือ -1 ถ้าไม่พบ)
hash_lookup:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r14
    
    mov r12, rdi                            ; เก็บ key
    
    ; คำนวณ hash
    call hash_function
    mov r14, rax
    
    xor rbx, rbx                            ; probe_count = 0
    
.lookup_loop:
    cmp rbx, HASH_SIZE
    jge .lookup_notfound
    
    ; คำนวณ address
    mov rcx, r14
    imul rcx, ENTRY_SIZE
    lea rdx, [hash_table]
    add rdx, rcx
    
    mov rax, [rdx + ENTRY_USED]
    
    cmp rax, SLOT_EMPTY
    je .lookup_notfound                     ; slot ว่าง = ไม่มี key นี้
    
    cmp rax, SLOT_DELETED
    je .skip_deleted
    
    ; ตรวจสอบ key
    cmp [rdx + ENTRY_KEY], r12
    je .lookup_found
    
.skip_deleted:
    inc r14
    and r14, HASH_MASK
    inc rbx
    jmp .lookup_loop
    
.lookup_found:
    mov rax, [rdx + ENTRY_VALUE]
    jmp .lookup_done
    
.lookup_notfound:
    mov rax, -1
    
.lookup_done:
    pop r14
    pop r12
    pop rbx
    pop rbp
    ret

; ฟังก์ชัน: hash_delete
; ลบ key-value pair ออกจาก hash table
; Input:  rdi = key
; Output: rax = 0 ถ้าสำเร็จ, -1 ถ้าไม่พบ
hash_delete:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r14
    
    mov r12, rdi
    call hash_function
    mov r14, rax
    xor rbx, rbx
    
.delete_loop:
    cmp rbx, HASH_SIZE
    jge .delete_notfound
    
    mov rcx, r14
    imul rcx, ENTRY_SIZE
    lea rdx, [hash_table]
    add rdx, rcx
    
    mov rax, [rdx + ENTRY_USED]
    cmp rax, SLOT_EMPTY
    je .delete_notfound
    
    cmp rax, SLOT_DELETED
    je .skip_deleted
    
    cmp [rdx + ENTRY_KEY], r12
    je .delete_found
    
.skip_deleted:
    inc r14
    and r14, HASH_MASK
    inc rbx
    jmp .delete_loop
    
.delete_found:
    ; ทำเครื่องหมาย slot ว่าถูกลบแล้ว (lazy deletion)
    mov qword [rdx + ENTRY_USED], SLOT_DELETED
    xor rax, rax
    jmp .delete_done
    
.delete_notfound:
    mov rax, -1
    
.delete_done:
    pop r14
    pop r12
    pop rbx
    pop rbp
    ret

_start:
    ; ทดสอบ hash table
    ; Insert: key=1, value=100
    mov rdi, 1
    mov rsi, 100
    call hash_insert
    
    ; Insert: key=2, value=200
    mov rdi, 2
    mov rsi, 200
    call hash_insert
    
    ; Insert: key=5, value=500
    mov rdi, 5
    mov rsi, 500
    call hash_insert
    
    ; Lookup key=2 (ควรได้ 200)
    mov rdi, 2
    call hash_lookup
    ; rax = 200
    
    ; Delete key=2
    mov rdi, 2
    call hash_delete
    
    ; Lookup key=2 อีกครั้ง (ควรได้ -1)
    mov rdi, 2
    call hash_lookup
    ; rax = -1
    
    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 6. Binary Tree (ต้นไม้ Binary)

```nasm
; ไฟล์: binary_tree.asm
; การสร้าง Binary Search Tree ใน NASM
; คอมไพล์: nasm -f elf64 binary_tree.asm -o binary_tree.o

; โครงสร้าง BST Node:
; offset 0:  key (8 bytes)
; offset 8:  left pointer (8 bytes)
; offset 16: right pointer (8 bytes)
; รวม: 24 bytes per node

BST_KEY     equ 0
BST_LEFT    equ 8
BST_RIGHT   equ 16
BST_SIZE    equ 24

section .data
    root_ptr    dq 0                        ; pointer ไปยัง root node

section .text
    global _start

; ฟังก์ชัน: bst_alloc_node
; จัดสรรหน่วยความจำสำหรับ BST node
; Input:  rdi = key
; Output: rax = pointer ไปยัง node ใหม่
bst_alloc_node:
    push rbp
    mov rbp, rsp
    push rdi
    
    ; mmap สำหรับ node
    mov rax, 9                              ; sys_mmap
    xor rdi, rdi
    mov rsi, BST_SIZE
    mov rdx, 3
    mov r10, 0x22
    mov r8, -1
    xor r9, r9
    syscall
    
    test rax, rax
    js .alloc_fail
    
    pop rdi
    ; กำหนดค่า node
    mov [rax + BST_KEY], rdi
    mov qword [rax + BST_LEFT], 0
    mov qword [rax + BST_RIGHT], 0
    
    pop rbp
    ret
    
.alloc_fail:
    pop rdi
    xor rax, rax
    pop rbp
    ret

; ฟังก์ชัน: bst_insert
; เพิ่ม key เข้าไปใน BST
; Input:  rdi = root pointer (address ของ root_ptr variable)
;         rsi = key ที่ต้องการเพิ่ม
bst_insert:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    
    mov r12, rdi                            ; address ของ root ptr variable
    mov r13, rsi                            ; key ที่ต้องการเพิ่ม
    
    ; ถ้า root = NULL ให้สร้าง root node
    mov rbx, [r12]
    test rbx, rbx
    jz .create_root
    
    ; วนหา position ที่จะเพิ่ม node
.traverse:
    cmp r13, [rbx + BST_KEY]
    jl .go_left
    je .already_exists                      ; key ซ้ำ ไม่เพิ่ม
    
    ; ไปทาง right
    mov rcx, [rbx + BST_RIGHT]
    test rcx, rcx
    jz .insert_right
    mov rbx, rcx
    jmp .traverse
    
.go_left:
    ; ไปทาง left
    mov rcx, [rbx + BST_LEFT]
    test rcx, rcx
    jz .insert_left
    mov rbx, rcx
    jmp .traverse
    
.create_root:
    ; สร้าง root node
    mov rdi, r13
    call bst_alloc_node
    test rax, rax
    jz .insert_fail
    mov [r12], rax                          ; root = new node
    jmp .insert_done
    
.insert_left:
    ; สร้าง node ใหม่ทาง left
    mov rdi, r13
    call bst_alloc_node
    test rax, rax
    jz .insert_fail
    mov [rbx + BST_LEFT], rax
    jmp .insert_done
    
.insert_right:
    ; สร้าง node ใหม่ทาง right
    mov rdi, r13
    call bst_alloc_node
    test rax, rax
    jz .insert_fail
    mov [rbx + BST_RIGHT], rax
    
.already_exists:
.insert_done:
.insert_fail:
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

; ฟังก์ชัน: bst_search
; ค้นหา key ใน BST
; Input:  rdi = root node pointer
;         rsi = key ที่ต้องการค้นหา
; Output: rax = pointer ไปยัง node (หรือ 0 ถ้าไม่พบ)
bst_search:
    push rbp
    mov rbp, rsp
    
    mov rax, rdi                            ; current = root
    
.search_loop:
    test rax, rax                           ; ตรวจสอบ NULL
    jz .search_notfound
    
    cmp rsi, [rax + BST_KEY]
    je .search_found
    jl .go_left
    
    ; ไปทาง right
    mov rax, [rax + BST_RIGHT]
    jmp .search_loop
    
.go_left:
    mov rax, [rax + BST_LEFT]
    jmp .search_loop
    
.search_found:
    ; rax = node pointer
    pop rbp
    ret
    
.search_notfound:
    xor rax, rax
    pop rbp
    ret

; ฟังก์ชัน: bst_inorder
; Traverse BST แบบ inorder (ได้ค่าเรียงจากน้อยไปมาก)
; Input:  rdi = root node pointer
;         rsi = callback function address (rdi = key)
bst_inorder:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    
    test rdi, rdi                           ; ตรวจสอบ NULL
    jz .inorder_done
    
    mov rbx, rdi                            ; เก็บ current node
    mov r12, rsi                            ; เก็บ callback
    
    ; ไปทาง left ก่อน
    mov rdi, [rbx + BST_LEFT]
    mov rsi, r12
    call bst_inorder
    
    ; เรียก callback ด้วยค่า key ปัจจุบัน
    mov rdi, [rbx + BST_KEY]
    call r12
    
    ; ไปทาง right
    mov rdi, [rbx + BST_RIGHT]
    mov rsi, r12
    call bst_inorder
    
.inorder_done:
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

; ฟังก์ชัน: bst_find_min
; หา node ที่มี key น้อยที่สุดใน subtree
; Input:  rdi = root of subtree
; Output: rax = pointer ไปยัง min node
bst_find_min:
    mov rax, rdi
.find_min_loop:
    mov rcx, [rax + BST_LEFT]
    test rcx, rcx
    jz .found_min
    mov rax, rcx
    jmp .find_min_loop
.found_min:
    ret

_start:
    ; สร้าง BST ด้วยค่า: 50, 30, 70, 20, 40, 60, 80
    lea rdi, [root_ptr]
    mov rsi, 50
    call bst_insert
    
    lea rdi, [root_ptr]
    mov rsi, 30
    call bst_insert
    
    lea rdi, [root_ptr]
    mov rsi, 70
    call bst_insert
    
    lea rdi, [root_ptr]
    mov rsi, 20
    call bst_insert
    
    lea rdi, [root_ptr]
    mov rsi, 40
    call bst_insert
    
    lea rdi, [root_ptr]
    mov rsi, 60
    call bst_insert
    
    lea rdi, [root_ptr]
    mov rsi, 80
    call bst_insert
    
    ; ค้นหาค่า 40 (ควรพบ)
    mov rdi, [root_ptr]
    mov rsi, 40
    call bst_search
    ; rax != 0 (found)
    
    ; ค้นหาค่า 35 (ไม่มี)
    mov rdi, [root_ptr]
    mov rsi, 35
    call bst_search
    ; rax = 0 (not found)
    
    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 7. Sorting Algorithms (อัลกอริทึมการเรียงลำดับ)

### 7.1 Bubble Sort

```nasm
; ไฟล์: bubble_sort.asm
; Bubble Sort ใน NASM
; คอมไพล์: nasm -f elf64 bubble_sort.asm -o bubble_sort.o

section .data
    test_array  dq 64, 34, 25, 12, 22, 11, 90
    arr_size    equ 7

section .text
    global _start

; ฟังก์ชัน: bubble_sort
; เรียงลำดับ array แบบ ascending
; Input:  rdi = pointer ไปยัง array
;         rsi = จำนวน elements
bubble_sort:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    
    mov r12, rdi                            ; base address
    mov r13, rsi                            ; n
    
    ; outer loop: i = 0 to n-2
    xor rbx, rbx                            ; i = 0
    
.outer_loop:
    lea rax, [r13 - 1]
    cmp rbx, rax
    jge .sort_done
    
    ; inner loop: j = 0 to n-i-2
    xor r14, r14                            ; j = 0
    
.inner_loop:
    mov rax, r13
    sub rax, rbx
    dec rax                                 ; n - i - 1
    cmp r14, rax
    jge .next_outer
    
    ; เปรียบเทียบ array[j] กับ array[j+1]
    mov rax, [r12 + r14*8]                  ; array[j]
    mov rcx, [r12 + r14*8 + 8]             ; array[j+1]
    
    cmp rax, rcx
    jle .no_swap                            ; ถ้า array[j] <= array[j+1] ไม่ต้องสลับ
    
    ; สลับค่า
    mov [r12 + r14*8], rcx
    mov [r12 + r14*8 + 8], rax
    
.no_swap:
    inc r14
    jmp .inner_loop
    
.next_outer:
    inc rbx
    jmp .outer_loop
    
.sort_done:
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

_start:
    lea rdi, [test_array]
    mov rsi, arr_size
    call bubble_sort
    ; test_array ตอนนี้: 11, 12, 22, 25, 34, 64, 90
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

### 7.2 Insertion Sort

```nasm
; ไฟล์: insertion_sort.asm
; Insertion Sort ใน NASM
; คอมไพล์: nasm -f elf64 insertion_sort.asm -o insertion_sort.o

section .data
    arr     dq 5, 2, 8, 1, 9, 3, 7, 4, 6
    n       equ 9

section .text
    global _start

; ฟังก์ชัน: insertion_sort
; Input:  rdi = pointer ไปยัง array (int64)
;         rsi = จำนวน elements
insertion_sort:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    
    mov r12, rdi                            ; base address
    mov r13, rsi                            ; n
    
    mov rbx, 1                              ; i = 1
    
.outer:
    cmp rbx, r13
    jge .sort_done
    
    ; key = array[i]
    mov r14, [r12 + rbx*8]                  ; key
    
    ; j = i - 1
    lea rcx, [rbx - 1]
    
.inner:
    cmp rcx, 0
    jl .insert_key
    
    mov rax, [r12 + rcx*8]                  ; array[j]
    cmp rax, r14
    jle .insert_key                         ; ถ้า array[j] <= key หยุด
    
    ; เลื่อน array[j] ไปทางขวา
    mov [r12 + rcx*8 + 8], rax
    dec rcx
    jmp .inner
    
.insert_key:
    ; เก็บ key ที่ตำแหน่ง j+1
    mov [r12 + rcx*8 + 8], r14
    
    inc rbx
    jmp .outer
    
.sort_done:
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

_start:
    lea rdi, [arr]
    mov rsi, n
    call insertion_sort
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

### 7.3 Quicksort

```nasm
; ไฟล์: quicksort.asm
; Quicksort ใน NASM (recursive)
; คอมไพล์: nasm -f elf64 quicksort.asm -o quicksort.o

section .data
    arr     dq 10, 7, 8, 9, 1, 5
    n       equ 6

section .text
    global _start

; ฟังก์ชัน: partition
; แบ่ง array โดยใช้ pivot (last element)
; Input:  rdi = base address ของ array
;         rsi = low index
;         rdx = high index
; Output: rax = pivot index
partition:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    push r15
    
    mov r12, rdi                            ; base
    mov r13, rsi                            ; low
    mov r14, rdx                            ; high
    
    ; pivot = array[high]
    mov r15, [r12 + r14*8]
    
    ; i = low - 1
    lea rbx, [r13 - 1]
    
    mov rcx, r13                            ; j = low
    
.partition_loop:
    cmp rcx, r14
    jge .partition_done
    
    ; ถ้า array[j] <= pivot
    mov rax, [r12 + rcx*8]
    cmp rax, r15
    jg .next_j
    
    ; i++
    inc rbx
    
    ; swap array[i] กับ array[j]
    mov rdx, [r12 + rbx*8]
    mov [r12 + rbx*8], rax
    mov [r12 + rcx*8], rdx
    
.next_j:
    inc rcx
    jmp .partition_loop
    
.partition_done:
    ; swap array[i+1] กับ array[high] (pivot)
    inc rbx                                 ; i+1
    mov rax, [r12 + rbx*8]
    mov [r12 + rbx*8], r15
    mov [r12 + r14*8], rax
    
    mov rax, rbx                            ; return i+1 (pivot index)
    
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

; ฟังก์ชัน: quicksort_helper
; Recursive quicksort
; Input:  rdi = base address
;         rsi = low index
;         rdx = high index
quicksort_helper:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    
    mov r12, rdi
    mov r13, rsi
    mov r14, rdx
    
    ; Base case: low >= high
    cmp r13, r14
    jge .qs_done
    
    ; Partition
    mov rdi, r12
    mov rsi, r13
    mov rdx, r14
    call partition
    mov rbx, rax                            ; pivot index
    
    ; Recursively sort left half
    mov rdi, r12
    mov rsi, r13
    lea rdx, [rbx - 1]
    call quicksort_helper
    
    ; Recursively sort right half
    mov rdi, r12
    lea rsi, [rbx + 1]
    mov rdx, r14
    call quicksort_helper
    
.qs_done:
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

; ฟังก์ชัน: quicksort (public interface)
; Input:  rdi = pointer ไปยัง array
;         rsi = จำนวน elements
quicksort:
    push rbp
    mov rbp, rsp
    
    test rsi, rsi
    jle .qs_empty
    
    ; เรียก quicksort_helper(arr, 0, n-1)
    ; rdi ไม่เปลี่ยน
    mov rdx, rsi
    dec rdx                                 ; high = n - 1
    xor rsi, rsi                            ; low = 0
    call quicksort_helper
    
.qs_empty:
    pop rbp
    ret

_start:
    lea rdi, [arr]
    mov rsi, n
    call quicksort
    ; arr ตอนนี้: 1, 5, 7, 8, 9, 10
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

### 7.4 Merge Sort

```nasm
; ไฟล์: mergesort.asm
; Merge Sort ใน NASM
; คอมไพล์: nasm -f elf64 mergesort.asm -o mergesort.o

section .data
    arr     dq 38, 27, 43, 3, 9, 82, 10
    n       equ 7

section .bss
    temp_buf    resq 100                    ; temporary buffer สำหรับ merge

section .text
    global _start

; ฟังก์ชัน: merge
; รวม 2 sorted subarrays เป็น 1
; Input:  rdi = base address
;         rsi = left index
;         rdx = mid index
;         rcx = right index
merge:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    push r15
    
    mov r12, rdi                            ; base
    mov r13, rsi                            ; left
    mov r14, rdx                            ; mid
    mov r15, rcx                            ; right
    
    ; คำนวณขนาดของ left และ right subarrays
    mov rax, r14
    sub rax, r13
    inc rax                                 ; n1 = mid - left + 1
    push rax
    
    mov rbx, r15
    sub rbx, r14                            ; n2 = right - mid
    
    ; คัดลอก left subarray ไปยัง temp_buf
    xor rcx, rcx                            ; i = 0
    pop rax                                 ; n1
    push rax
.copy_left:
    cmp rcx, rax
    jge .copy_right_start
    mov rdx, [r12 + r13*8 + rcx*8]
    mov [temp_buf + rcx*8], rdx
    inc rcx
    jmp .copy_left
    
    ; คัดลอก right subarray ไปยัง temp_buf หลัง left
.copy_right_start:
    pop rax                                 ; n1
    push rax
    push rbx                                ; n2
    xor rcx, rcx                            ; j = 0
.copy_right:
    cmp rcx, rbx
    jge .merge_start
    lea rdx, [r14 + 1]
    add rdx, rcx
    mov r8, [r12 + rdx*8]
    mov [temp_buf + rax*8 + rcx*8], r8
    inc rcx
    jmp .copy_right
    
.merge_start:
    pop rbx                                 ; n2
    pop rax                                 ; n1
    
    xor rcx, rcx                            ; i = 0 (left index)
    xor rdx, rdx                            ; j = 0 (right index)
    mov r8, r13                             ; k = left (merged index)
    
.merge_loop:
    cmp rcx, rax                            ; i < n1?
    jge .copy_remaining_right
    cmp rdx, rbx                            ; j < n2?
    jge .copy_remaining_left
    
    ; เปรียบเทียบ temp_buf[i] กับ temp_buf[n1+j]
    mov r9, [temp_buf + rcx*8]
    mov r10, [temp_buf + rax*8 + rdx*8]
    
    cmp r9, r10
    jg .take_right
    
    ; เอา left element
    mov [r12 + r8*8], r9
    inc rcx
    inc r8
    jmp .merge_loop
    
.take_right:
    ; เอา right element
    mov [r12 + r8*8], r10
    inc rdx
    inc r8
    jmp .merge_loop
    
.copy_remaining_left:
    cmp rcx, rax
    jge .merge_done
    mov r9, [temp_buf + rcx*8]
    mov [r12 + r8*8], r9
    inc rcx
    inc r8
    jmp .copy_remaining_left
    
.copy_remaining_right:
    cmp rdx, rbx
    jge .merge_done
    mov r10, [temp_buf + rax*8 + rdx*8]
    mov [r12 + r8*8], r10
    inc rdx
    inc r8
    jmp .copy_remaining_right
    
.merge_done:
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

; ฟังก์ชัน: mergesort_helper
; Input:  rdi = base address
;         rsi = left index
;         rdx = right index
mergesort_helper:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    
    mov r12, rdi
    mov r13, rsi
    mov r14, rdx
    
    ; Base case
    cmp r13, r14
    jge .ms_done
    
    ; คำนวณ mid
    mov rax, r13
    add rax, r14
    shr rax, 1                              ; mid = (left + right) / 2
    mov rbx, rax
    
    ; Sort left half
    mov rdi, r12
    mov rsi, r13
    mov rdx, rbx
    call mergesort_helper
    
    ; Sort right half
    mov rdi, r12
    lea rsi, [rbx + 1]
    mov rdx, r14
    call mergesort_helper
    
    ; Merge
    mov rdi, r12
    mov rsi, r13
    mov rdx, rbx
    mov rcx, r14
    call merge
    
.ms_done:
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

mergesort:
    push rbp
    mov rbp, rsp
    
    test rsi, rsi
    jle .ms_empty
    
    mov rdx, rsi
    dec rdx
    xor rsi, rsi
    call mergesort_helper
    
.ms_empty:
    pop rbp
    ret

_start:
    lea rdi, [arr]
    mov rsi, n
    call mergesort
    ; arr ตอนนี้: 3, 9, 10, 27, 38, 43, 82
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 8. Searching Algorithms (อัลกอริทึมการค้นหา)

```nasm
; ไฟล์: searching.asm
; Linear Search และ Binary Search ใน NASM
; คอมไพล์: nasm -f elf64 searching.asm -o searching.o

section .data
    ; sorted array สำหรับ binary search
    sorted_arr  dq 2, 5, 8, 12, 16, 23, 38, 56, 72, 91
    arr_size    equ 10

section .text
    global _start

; ฟังก์ชัน: linear_search
; ค้นหาแบบ linear (ไม่ต้อง sorted)
; Input:  rdi = pointer ไปยัง array
;         rsi = จำนวน elements
;         rdx = ค่าที่ต้องการค้นหา
; Output: rax = index ที่พบ (หรือ -1 ถ้าไม่พบ)
linear_search:
    push rbp
    mov rbp, rsp
    
    xor rcx, rcx                            ; index = 0
    
.search_loop:
    cmp rcx, rsi
    jge .not_found
    
    cmp [rdi + rcx*8], rdx                  ; เปรียบเทียบ array[i] กับ target
    je .found
    
    inc rcx
    jmp .search_loop
    
.found:
    mov rax, rcx                            ; return index
    pop rbp
    ret
    
.not_found:
    mov rax, -1
    pop rbp
    ret

; ฟังก์ชัน: binary_search
; ค้นหาแบบ binary (ต้องเป็น sorted array)
; Input:  rdi = pointer ไปยัง sorted array
;         rsi = จำนวน elements
;         rdx = ค่าที่ต้องการค้นหา
; Output: rax = index ที่พบ (หรือ -1 ถ้าไม่พบ)
binary_search:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    
    mov r12, rdi                            ; base address
    mov r13, rdx                            ; target
    
    xor rbx, rbx                            ; low = 0
    lea rcx, [rsi - 1]                      ; high = n - 1
    
.bs_loop:
    cmp rbx, rcx
    jg .bs_not_found
    
    ; mid = (low + high) / 2
    mov rax, rbx
    add rax, rcx
    shr rax, 1
    
    ; เปรียบเทียบ array[mid] กับ target
    mov rdx, [r12 + rax*8]
    
    cmp rdx, r13
    je .bs_found
    jl .go_right
    
    ; target อยู่ทาง left
    lea rcx, [rax - 1]                      ; high = mid - 1
    jmp .bs_loop
    
.go_right:
    lea rbx, [rax + 1]                      ; low = mid + 1
    jmp .bs_loop
    
.bs_found:
    ; rax ยังคงเก็บ mid index ที่พบ
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret
    
.bs_not_found:
    mov rax, -1
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

_start:
    ; Linear search หา 23 ใน array
    lea rdi, [sorted_arr]
    mov rsi, arr_size
    mov rdx, 23
    call linear_search
    ; rax = 5 (index 5)
    
    ; Binary search หา 56
    lea rdi, [sorted_arr]
    mov rsi, arr_size
    mov rdx, 56
    call binary_search
    ; rax = 7 (index 7)
    
    ; Binary search หาค่าที่ไม่มี (50)
    lea rdi, [sorted_arr]
    mov rsi, arr_size
    mov rdx, 50
    call binary_search
    ; rax = -1 (ไม่พบ)
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 9. String Operations (การดำเนินการกับ String)

```nasm
; ไฟล์: string_ops.asm
; การดำเนินการกับ String ใน NASM
; คอมไพล์: nasm -f elf64 string_ops.asm -o string_ops.o
; ลิงก์:   ld string_ops.o -o string_ops

section .data
    test_str1   db "Hello, Assembly!", 0
    test_str2   db "racecar", 0
    test_str3   db "A man a plan a canal Panama", 0
    
    msg_palindrome  db " is a palindrome", 10, 0
    msg_not_palindrome db " is NOT a palindrome", 10, 0
    msg_reversed    db "Reversed: ", 0
    msg_length      db "Length: ", 0
    msg_nl          db 10, 0

section .bss
    result_buf  resb 256
    print_buf   resb 32

section .text
    global _start

; ฟังก์ชัน: str_length
; คำนวณความยาวของ string
; Input:  rdi = pointer ไปยัง string (null-terminated)
; Output: rax = ความยาว (ไม่นับ null terminator)
str_length:
    xor rax, rax
.len_loop:
    cmp byte [rdi + rax], 0
    je .len_done
    inc rax
    jmp .len_loop
.len_done:
    ret

; ฟังก์ชัน: str_reverse
; กลับลำดับ string (in-place)
; Input:  rdi = pointer ไปยัง string
str_reverse:
    push rbp
    mov rbp, rsp
    push rbx
    push rcx
    
    ; หาความยาว
    push rdi
    call str_length
    pop rdi
    
    ; two-pointer approach
    mov rbx, rdi                            ; left pointer
    lea rcx, [rdi + rax - 1]               ; right pointer
    
.reverse_loop:
    cmp rbx, rcx
    jge .reverse_done
    
    ; swap *left กับ *right
    mov al, [rbx]
    mov dl, [rcx]
    mov [rbx], dl
    mov [rcx], al
    
    inc rbx
    dec rcx
    jmp .reverse_loop
    
.reverse_done:
    pop rcx
    pop rbx
    pop rbp
    ret

; ฟังก์ชัน: str_copy
; คัดลอก string
; Input:  rdi = destination buffer
;         rsi = source string
str_copy:
    push rbp
    mov rbp, rsp
    
.copy_loop:
    mov al, [rsi]
    mov [rdi], al
    inc rsi
    inc rdi
    test al, al                             ; ตรวจสอบ null terminator
    jnz .copy_loop
    
    pop rbp
    ret

; ฟังก์ชัน: is_palindrome
; ตรวจสอบว่า string เป็น palindrome หรือไม่
; Input:  rdi = pointer ไปยัง string
; Output: rax = 1 ถ้าเป็น palindrome, 0 ถ้าไม่ใช่
is_palindrome:
    push rbp
    mov rbp, rsp
    push rbx
    push rcx
    
    push rdi
    call str_length
    pop rdi
    
    test rax, rax
    jz .is_palindrome_true                  ; string ว่าง = palindrome
    
    mov rbx, rdi                            ; left pointer
    lea rcx, [rdi + rax - 1]               ; right pointer
    
.palindrome_loop:
    cmp rbx, rcx
    jge .is_palindrome_true                 ; left >= right = ผ่านการตรวจสอบ
    
    ; เปรียบเทียบตัวอักษรจากสองด้าน (case-insensitive)
    movzx rax, byte [rbx]
    movzx rdx, byte [rcx]
    
    ; แปลงเป็น lowercase สำหรับการเปรียบเทียบ
    cmp al, 'A'
    jl .left_done
    cmp al, 'Z'
    jg .left_done
    or al, 0x20                             ; แปลงเป็น lowercase
.left_done:
    cmp dl, 'A'
    jl .right_done
    cmp dl, 'Z'
    jg .right_done
    or dl, 0x20
.right_done:
    
    cmp al, dl
    jne .not_palindrome
    
    inc rbx
    dec rcx
    jmp .palindrome_loop
    
.is_palindrome_true:
    mov rax, 1
    jmp .palindrome_done
    
.not_palindrome:
    xor rax, rax
    
.palindrome_done:
    pop rcx
    pop rbx
    pop rbp
    ret

; ฟังก์ชัน: str_compare
; เปรียบเทียบ 2 strings
; Input:  rdi = string 1
;         rsi = string 2
; Output: rax = 0 ถ้าเท่ากัน, <0 ถ้า s1 < s2, >0 ถ้า s1 > s2
str_compare:
.cmp_loop:
    movzx rax, byte [rdi]
    movzx rcx, byte [rsi]
    
    sub rax, rcx
    jnz .cmp_done
    
    test cl, cl                             ; ถึง null terminator?
    jz .cmp_done
    
    inc rdi
    inc rsi
    jmp .cmp_loop
    
.cmp_done:
    ret

; ฟังก์ชัน: str_find_char
; ค้นหาตัวอักษรใน string
; Input:  rdi = string
;         rsi = ตัวอักษรที่ต้องการค้นหา (byte)
; Output: rax = pointer ไปยังตัวอักษรแรกที่พบ (หรือ 0 ถ้าไม่พบ)
str_find_char:
    mov rax, rdi
.find_loop:
    movzx rcx, byte [rax]
    test cl, cl
    jz .find_notfound
    cmp cl, sil
    je .find_found
    inc rax
    jmp .find_loop
.find_found:
    ret
.find_notfound:
    xor rax, rax
    ret

; ฟังก์ชัน: str_count_char
; นับจำนวนการปรากฏของตัวอักษรใน string
; Input:  rdi = string
;         rsi = ตัวอักษรที่ต้องการนับ
; Output: rax = จำนวนครั้งที่พบ
str_count_char:
    xor rax, rax                            ; count = 0
.count_loop:
    movzx rcx, byte [rdi]
    test cl, cl
    jz .count_done
    cmp cl, sil
    jne .count_next
    inc rax
.count_next:
    inc rdi
    jmp .count_loop
.count_done:
    ret

; ฟังก์ชัน: str_to_upper
; แปลง string เป็น uppercase (in-place)
; Input:  rdi = string
str_to_upper:
.upper_loop:
    movzx rax, byte [rdi]
    test al, al
    jz .upper_done
    
    cmp al, 'a'
    jl .skip_upper
    cmp al, 'z'
    jg .skip_upper
    and al, 0xDF                            ; แปลงเป็น uppercase (ล้าง bit 5)
    mov [rdi], al
    
.skip_upper:
    inc rdi
    jmp .upper_loop
.upper_done:
    ret

; print_string helper
print_string:
    push rsi
    push rdx
    push rax
    push rdi
    mov rsi, rdi
    xor rdx, rdx
.plen:
    cmp byte [rsi+rdx], 0
    je .pprint
    inc rdx
    jmp .plen
.pprint:
    mov rax, 1
    mov rdi, 1
    syscall
    pop rdi
    pop rax
    pop rdx
    pop rsi
    ret

_start:
    ; ทดสอบ str_length
    lea rdi, [test_str1]
    call str_length
    ; rax = 16 (ความยาวของ "Hello, Assembly!")
    
    ; ทดสอบ is_palindrome กับ "racecar"
    lea rdi, [test_str2]
    call is_palindrome
    ; rax = 1 (เป็น palindrome)
    
    ; ทดสอบ is_palindrome กับ "Hello, Assembly!"
    lea rdi, [test_str1]
    call is_palindrome
    ; rax = 0 (ไม่ใช่ palindrome)
    
    ; คัดลอก string แล้ว reverse
    lea rdi, [result_buf]
    lea rsi, [test_str1]
    call str_copy
    
    lea rdi, [result_buf]
    call str_reverse
    ; result_buf = "!ylbmessA ,olleH"
    
    ; แสดงผล reversed string
    lea rdi, [msg_reversed]
    call print_string
    lea rdi, [result_buf]
    call print_string
    lea rdi, [msg_nl]
    call print_string
    
    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 10. Simple Memory Allocator (Bump Allocator)

```nasm
; ไฟล์: bump_allocator.asm
; Simple Bump Allocator ใน NASM
; คอมไพล์: nasm -f elf64 bump_allocator.asm -o bump_allocator.o
; ลิงก์:   ld bump_allocator.o -o bump_allocator

; Bump Allocator ทำงานโดยเก็บ pointer ไปยังหน่วยความจำที่ว่างอยู่
; และเลื่อน pointer ไปข้างหน้าทุกครั้งที่มีการจัดสรร
; ไม่รองรับการคืนหน่วยความจำแบบ individual (ต้อง reset ทั้งหมด)

HEAP_SIZE   equ 65536                       ; 64KB heap

section .bss
    heap_start  resb HEAP_SIZE              ; heap memory
    heap_ptr    resq 1                      ; current pointer (bump pointer)
    heap_end    resq 1                      ; end of heap

section .data
    msg_alloc   db "Allocated at: 0x", 0
    msg_nl      db 10, 0
    msg_oom     db "Out of memory!", 10, 0

section .text
    global _start

; ฟังก์ชัน: allocator_init
; เริ่มต้น bump allocator
allocator_init:
    ; ตั้ง heap_ptr ไปที่จุดเริ่มต้น
    lea rax, [heap_start]
    mov [heap_ptr], rax
    
    ; ตั้ง heap_end
    lea rax, [heap_start + HEAP_SIZE]
    mov [heap_end], rax
    ret

; ฟังก์ชัน: bump_alloc
; จัดสรรหน่วยความจำด้วย bump allocator
; Input:  rdi = จำนวน bytes ที่ต้องการ
; Output: rax = pointer ไปยังหน่วยความจำที่จัดสรร (หรือ 0 ถ้าหมด)
bump_alloc:
    push rbp
    mov rbp, rsp
    
    ; Align ขนาดเป็น 8 bytes (สำหรับ alignment)
    add rdi, 7
    and rdi, ~7                             ; round up to multiple of 8
    
    ; ตรวจสอบว่ามีพื้นที่เพียงพอหรือไม่
    mov rax, [heap_ptr]
    mov rcx, rax
    add rcx, rdi
    
    cmp rcx, [heap_end]
    jg .oom
    
    ; อัพเดท bump pointer
    mov [heap_ptr], rcx
    
    ; ล้างหน่วยความจำที่จัดสรร (zero out)
    push rdi
    push rax
    
    mov rdi, rax                            ; destination
    xor eax, eax                            ; value = 0
    ; rsi ยังคงเป็น size จาก stack
    pop rdi                                 ; restore allocated address
    pop rcx                                 ; size
    push rdi
    
    ; zero memory loop
    xor rdx, rdx
.zero_loop:
    cmp rdx, rcx
    jge .zero_done
    mov byte [rdi + rdx], 0
    inc rdx
    jmp .zero_loop
.zero_done:
    
    pop rax                                 ; return allocated address
    
    pop rbp
    ret
    
.oom:
    ; แสดงข้อความ out of memory
    push rdi
    mov rax, 1
    mov rdi, 1
    lea rsi, [msg_oom]
    mov rdx, 14
    syscall
    pop rdi
    
    xor rax, rax                            ; return NULL
    pop rbp
    ret

; ฟังก์ชัน: allocator_reset
; รีเซ็ต allocator (คืนหน่วยความจำทั้งหมด)
allocator_reset:
    lea rax, [heap_start]
    mov [heap_ptr], rax
    ret

; ฟังก์ชัน: allocator_used
; คืนค่าจำนวน bytes ที่ถูกใช้งาน
; Output: rax = จำนวน bytes ที่ใช้
allocator_used:
    mov rax, [heap_ptr]
    sub rax, heap_start
    ret

; ฟังก์ชัน: allocator_remaining
; คืนค่าจำนวน bytes ที่เหลือ
; Output: rax = จำนวน bytes ที่เหลือ
allocator_remaining:
    mov rax, [heap_end]
    sub rax, [heap_ptr]
    ret

_start:
    ; เริ่มต้น allocator
    call allocator_init
    
    ; จัดสรร 16 bytes
    mov rdi, 16
    call bump_alloc
    ; rax = address ของหน่วยความจำ 16 bytes
    push rax                                ; เก็บ address ไว้
    
    ; ใช้หน่วยความจำที่จัดสรร
    pop rbx
    mov qword [rbx], 0xDEADBEEF            ; เขียนค่าลงไป
    
    ; จัดสรร 32 bytes
    mov rdi, 32
    call bump_alloc
    ; rax = address ถัดมา (หลังจาก 16 bytes แรก)
    
    ; ตรวจสอบว่าใช้ไปเท่าไร
    call allocator_used
    ; rax = 48 (16 + 32, พร้อม alignment)
    
    ; จัดสรร array ขนาด 10 int64
    mov rdi, 10 * 8                         ; 10 * 8 bytes
    call bump_alloc
    ; rax = pointer ไปยัง array
    push rax
    
    ; เขียนค่าลง array
    pop rbx
    mov qword [rbx + 0], 100
    mov qword [rbx + 8], 200
    mov qword [rbx + 16], 300
    
    ; รีเซ็ต allocator (คืนหน่วยความจำทั้งหมด)
    call allocator_reset
    
    ; จัดสรรใหม่ (เริ่มจากต้น)
    mov rdi, 8
    call bump_alloc
    ; rax = heap_start (เริ่มจากต้นอีกครั้ง)
    
    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 11. Dynamic Memory ด้วย mmap

```nasm
; ไฟล์: mmap_alloc.asm
; การใช้ mmap สำหรับ Dynamic Memory Allocation ใน NASM
; คอมไพล์: nasm -f elf64 mmap_alloc.asm -o mmap_alloc.o
; ลิงก์:   ld mmap_alloc.o -o mmap_alloc

; Constants สำหรับ mmap
PROT_READ   equ 1
PROT_WRITE  equ 2
MAP_PRIVATE equ 2
MAP_ANON    equ 0x20

section .data
    page_size   dq 4096                     ; ขนาดของ page (4KB)
    
section .text
    global _start

; ฟังก์ชัน: mmap_alloc
; จัดสรรหน่วยความจำด้วย mmap
; Input:  rdi = จำนวน bytes ที่ต้องการ
; Output: rax = pointer ไปยังหน่วยความจำ (หรือ -1 ถ้าล้มเหลว)
mmap_alloc:
    push rbp
    mov rbp, rsp
    
    ; ปัดขนาดขึ้นให้เป็น multiple ของ page size
    ; (rdi + 4095) & ~4095
    add rdi, 4095
    and rdi, ~4095
    
    ; เรียก mmap
    mov rax, 9                              ; sys_mmap
    xor rsi, rsi                            ; addr = NULL
    ; rdi = ขนาด (ตั้งค่าไว้แล้ว)
    mov rdx, PROT_READ | PROT_WRITE
    mov r10, MAP_PRIVATE | MAP_ANON
    mov r8, -1                              ; fd = -1 (anonymous)
    xor r9, r9                              ; offset = 0
    ; ต้องย้าย rdi ไป rsi ก่อน
    mov rsi, rdi
    xor rdi, rdi                            ; addr = NULL
    syscall
    
    ; ตรวจสอบข้อผิดพลาด
    cmp rax, -1
    je .mmap_fail
    
    pop rbp
    ret
    
.mmap_fail:
    mov rax, -1
    pop rbp
    ret

; ฟังก์ชัน: mmap_free
; คืนหน่วยความจำที่จัดสรรด้วย mmap
; Input:  rdi = pointer ไปยังหน่วยความจำ
;         rsi = ขนาด (bytes)
; Output: rax = 0 ถ้าสำเร็จ, -1 ถ้าล้มเหลว
mmap_free:
    push rbp
    mov rbp, rsp
    
    ; ปัดขนาดขึ้น
    add rsi, 4095
    and rsi, ~4095
    
    ; เรียก munmap
    mov rax, 11                             ; sys_munmap
    syscall
    
    pop rbp
    ret

_start:
    ; จัดสรร 1 page (4096 bytes)
    mov rdi, 4096
    call mmap_alloc
    cmp rax, -1
    je .alloc_failed
    
    mov rbx, rax                            ; เก็บ address
    
    ; ใช้หน่วยความจำ
    mov qword [rbx], 0x1234567890ABCDEF
    mov qword [rbx + 8], 42
    
    ; คืนหน่วยความจำ
    mov rdi, rbx
    mov rsi, 4096
    call mmap_free
    
.alloc_failed:
    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 12. Selection Sort

```nasm
; ไฟล์: selection_sort.asm
; Selection Sort ใน NASM
; คอมไพล์: nasm -f elf64 selection_sort.asm -o selection_sort.o

section .data
    arr     dq 64, 25, 12, 22, 11
    n       equ 5

section .text
    global _start

; ฟังก์ชัน: selection_sort
; Input:  rdi = pointer ไปยัง array
;         rsi = จำนวน elements
selection_sort:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    push r15
    
    mov r12, rdi                            ; base
    mov r13, rsi                            ; n
    
    xor rbx, rbx                            ; i = 0
    
.outer_sel:
    lea rax, [r13 - 1]
    cmp rbx, rax
    jge .sel_done
    
    ; หา minimum ใน subarray [i, n-1]
    mov r14, rbx                            ; min_idx = i
    lea r15, [rbx + 1]                      ; j = i + 1
    
.inner_sel:
    cmp r15, r13
    jge .swap_min
    
    ; เปรียบเทียบ array[j] กับ array[min_idx]
    mov rax, [r12 + r15*8]
    mov rcx, [r12 + r14*8]
    
    cmp rax, rcx
    jge .next_sel
    mov r14, r15                            ; min_idx = j
    
.next_sel:
    inc r15
    jmp .inner_sel
    
.swap_min:
    ; สลับ array[i] กับ array[min_idx]
    cmp r14, rbx
    je .next_outer_sel                      ; ถ้า min_idx == i ไม่ต้องสลับ
    
    mov rax, [r12 + rbx*8]
    mov rcx, [r12 + r14*8]
    mov [r12 + rbx*8], rcx
    mov [r12 + r14*8], rax
    
.next_outer_sel:
    inc rbx
    jmp .outer_sel
    
.sel_done:
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

_start:
    lea rdi, [arr]
    mov rsi, n
    call selection_sort
    ; arr ตอนนี้: 11, 12, 22, 25, 64
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## คำสั่งคอมไพล์และรัน

```bash
# คอมไพล์ทุกไฟล์
nasm -f elf64 static_array.asm -o static_array.o && ld static_array.o -o static_array
nasm -f elf64 dynamic_array.asm -o dynamic_array.o && ld dynamic_array.o -o dynamic_array
nasm -f elf64 2d_array.asm -o 2d_array.o && ld 2d_array.o -o 2d_array
nasm -f elf64 linked_list.asm -o linked_list.o && ld linked_list.o -o linked_list
nasm -f elf64 stack_array.asm -o stack_array.o && ld stack_array.o -o stack_array
nasm -f elf64 circular_queue.asm -o circular_queue.o && ld circular_queue.o -o circular_queue
nasm -f elf64 hash_table.asm -o hash_table.o && ld hash_table.o -o hash_table
nasm -f elf64 binary_tree.asm -o binary_tree.o && ld binary_tree.o -o binary_tree
nasm -f elf64 bubble_sort.asm -o bubble_sort.o && ld bubble_sort.o -o bubble_sort
nasm -f elf64 insertion_sort.asm -o insertion_sort.o && ld insertion_sort.o -o insertion_sort
nasm -f elf64 quicksort.asm -o quicksort.o && ld quicksort.o -o quicksort
nasm -f elf64 mergesort.asm -o mergesort.o && ld mergesort.o -o mergesort
nasm -f elf64 searching.asm -o searching.o && ld searching.o -o searching
nasm -f elf64 string_ops.asm -o string_ops.o && ld string_ops.o -o string_ops
nasm -f elf64 bump_allocator.asm -o bump_allocator.o && ld bump_allocator.o -o bump_allocator
nasm -f elf64 selection_sort.asm -o selection_sort.o && ld selection_sort.o -o selection_sort

# Debug ด้วย GDB
gdb ./quicksort
(gdb) break _start
(gdb) run
(gdb) x/7gd arr     # ดู array ก่อน sort
(gdb) next
(gdb) x/7gd arr     # ดู array หลัง sort
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Doubly Linked List
สร้าง Doubly Linked List ที่รองรับการ traverse ทั้งสองทิศทาง
- Node มี: data, prev pointer, next pointer
- ฟังก์ชัน: push_front, push_back, pop_front, pop_back, traverse_forward, traverse_backward

### แบบฝึกหัดที่ 2: Stack-based Expression Evaluator
สร้าง evaluator สำหรับ postfix expression (Reverse Polish Notation)
- รับ input: "5 3 + 2 *" = (5+3)*2 = 16
- ใช้ stack สำหรับการคำนวณ
- รองรับ: +, -, *, /

### แบบฝึกหัดที่ 3: Circular Queue ที่ expandable
ขยาย circular queue ให้สามารถเพิ่มขนาดได้อัตโนมัติเมื่อเต็ม
- เมื่อ queue เต็ม ให้ขยายขนาดเป็น 2 เท่า
- ใช้ mmap และ mremap

### แบบฝึกหัดที่ 4: Hash Table with Chaining
สร้าง hash table แบบ chaining (linked list ที่แต่ละ bucket)
- แก้ปัญหา collision ด้วยการเชื่อม node ใน linked list
- ฟังก์ชัน: insert, lookup, delete, resize (rehash)

### แบบฝึกหัดที่ 5: Heapsort
สร้าง Heapsort algorithm
- Build max-heap จาก array
- Extract elements ทีละตัวจาก heap
- ความซับซ้อน: O(n log n)

### แบบฝึกหัดที่ 6: String Interning
สร้าง string intern pool
- เก็บ unique strings ใน hash table
- ฟังก์ชัน intern(str) คืน pointer ไปยัง canonical copy
- ประหยัดหน่วยความจำสำหรับ strings ที่ซ้ำกัน

---

## สรุป Big-O Complexity ของ Data Structures

| Data Structure   | Access | Search | Insert | Delete |
|-----------------|--------|--------|--------|--------|
| Array (static)  | O(1)   | O(n)   | O(n)   | O(n)   |
| Linked List     | O(n)   | O(n)   | O(1)*  | O(1)*  |
| Stack (array)   | O(1)   | O(n)   | O(1)   | O(1)   |
| Queue (circular)| O(1)   | O(n)   | O(1)   | O(1)   |
| Hash Table      | O(1)   | O(1)   | O(1)   | O(1)   |
| BST (balanced)  | O(log n)| O(log n)| O(log n)| O(log n)|

\* สำหรับ linked list: O(1) เมื่อรู้ pointer ของ node ก่อนหน้า

---

## สรุป Sorting Algorithm Complexity

| Algorithm      | Best       | Average    | Worst      | Space  |
|---------------|------------|------------|------------|--------|
| Bubble Sort   | O(n)       | O(n²)      | O(n²)      | O(1)   |
| Selection Sort| O(n²)      | O(n²)      | O(n²)      | O(1)   |
| Insertion Sort| O(n)       | O(n²)      | O(n²)      | O(1)   |
| Quicksort     | O(n log n) | O(n log n) | O(n²)      | O(log n)|
| Merge Sort    | O(n log n) | O(n log n) | O(n log n) | O(n)   |

---

*Part 028 - Data Structures ใน Assembly | Assembly Programming Course*
