# Part 029: Structures และ Records ใน NASM

## ภาพรวม (Overview)

ใน Assembly การจัดกลุ่มข้อมูลที่เกี่ยวข้องกันเข้าด้วยกันเป็นสิ่งสำคัญมาก NASM มี directive พิเศษที่ชื่อว่า `STRUC` และ `ENDSTRUC` ที่ช่วยให้เราสามารถกำหนดโครงสร้างข้อมูล (structures) ได้คล้ายกับ `struct` ใน C/C++ ซึ่งทำให้โค้ดอ่านง่ายขึ้นและบำรุงรักษาได้ดีขึ้น

---

## 1. STRUC/ENDSTRUC Directive พื้นฐาน

### 1.1 ไวยากรณ์ (Syntax)

```nasm
struc ชื่อโครงสร้าง
    .ชื่อฟิลด์:  resb/resw/resd/resq  จำนวน
endstruc
```

### 1.2 ตัวอย่างแรก: Point Structure

```nasm
; ไฟล์: point_struct.asm
; คอมไพล์: nasm -f elf64 point_struct.asm -o point_struct.o
;          ld point_struct.o -o point_struct
; รัน:     ./point_struct

section .data
    ; สร้าง label สำหรับข้อความ
    msg_point   db "Point: (%d, %d)", 10, 0
    fmt_str     db "%s%d%s%d%s", 10, 0

; ====================================================
; กำหนด structure สำหรับจุดพิกัด 2D
; ====================================================
struc Point2D
    .x:     resd 1      ; 4 bytes สำหรับ x coordinate (int32)
    .y:     resd 1      ; 4 bytes สำหรับ y coordinate (int32)
endstruc
; Point2D_size = 8 bytes

; ====================================================
; กำหนด structure สำหรับจุดพิกัด 3D
; ====================================================
struc Point3D
    .x:     resd 1      ; 4 bytes สำหรับ x
    .y:     resd 1      ; 4 bytes สำหรับ y
    .z:     resd 1      ; 4 bytes สำหรับ z
endstruc
; Point3D_size = 12 bytes

section .bss
    ; จองพื้นที่ใน .bss สำหรับ Point2D
    myPoint:    resb Point2D_size

section .text
    global _start

_start:
    ; กำหนดค่า x = 10
    mov dword [myPoint + Point2D.x], 10
    
    ; กำหนดค่า y = 20
    mov dword [myPoint + Point2D.y], 20
    
    ; อ่านค่า x กลับมา
    mov eax, [myPoint + Point2D.x]
    ; eax = 10
    
    ; อ่านค่า y กลับมา
    mov ebx, [myPoint + Point2D.y]
    ; ebx = 20
    
    ; จบโปรแกรม
    mov eax, 60
    xor edi, edi
    syscall
```

---

## 2. ISTRUC/IEND สำหรับการสร้าง Instance

`ISTRUC` และ `IEND` ใช้สำหรับสร้าง initialized instance ของ structure ใน section `.data`

### 2.1 ตัวอย่าง ISTRUC พื้นฐาน

```nasm
; ไฟล์: istruc_demo.asm
; คอมไพล์: nasm -f elf64 istruc_demo.asm -o istruc_demo.o
;          ld istruc_demo.o -o istruc_demo

; ====================================================
; กำหนด Person structure
; ====================================================
struc Person
    .age:       resb 1      ; 1 byte อายุ (0-255)
    .gender:    resb 1      ; 1 byte เพศ (0=ชาย, 1=หญิง)
    .height:    resw 1      ; 2 bytes ส่วนสูง (cm)
    .weight:    resw 1      ; 2 bytes น้ำหนัก (กรัม x 100)
    .id:        resd 1      ; 4 bytes รหัสประจำตัว
endstruc
; Person_size = 10 bytes

section .data
    ; ====================================================
    ; สร้าง instance ของ Person ด้วย ISTRUC
    ; ====================================================
    alice:
        istruc Person
            at Person.age,    db  25        ; อายุ 25 ปี
            at Person.gender, db  1         ; เพศหญิง
            at Person.height, dw  165       ; ส่วนสูง 165 cm
            at Person.weight, dw  5500      ; น้ำหนัก 55.00 kg
            at Person.id,     dd  1001      ; รหัส 1001
        iend

    bob:
        istruc Person
            at Person.age,    db  30        ; อายุ 30 ปี
            at Person.gender, db  0         ; เพศชาย
            at Person.height, dw  175       ; ส่วนสูง 175 cm
            at Person.weight, dw  7000      ; น้ำหนัก 70.00 kg
            at Person.id,     dd  1002      ; รหัส 1002
        iend

section .text
    global _start

_start:
    ; อ่านข้อมูลของ alice
    movzx eax, byte [alice + Person.age]    ; eax = 25
    movzx ebx, word [alice + Person.height] ; ebx = 165
    mov   ecx, [alice + Person.id]          ; ecx = 1001
    
    ; อ่านข้อมูลของ bob
    movzx edx, byte [bob + Person.age]      ; edx = 30
    
    ; เปรียบเทียบอายุ alice vs bob
    cmp eax, edx
    jl  alice_younger   ; ถ้า alice อายุน้อยกว่า
    jmp done

alice_younger:
    ; alice อายุน้อยกว่า bob
    nop

done:
    mov eax, 60
    xor edi, edi
    syscall
```

---

## 3. การเข้าถึง Struct Members

### 3.1 การเข้าถึงด้วย Base Register

```nasm
; ไฟล์: struct_access.asm
; คอมไพล์: nasm -f elf64 struct_access.asm -o struct_access.o
;          gcc -nostartfiles struct_access.o -o struct_access

; ====================================================
; Rectangle structure
; ====================================================
struc Rectangle
    .x:         resd 1      ; ตำแหน่ง x
    .y:         resd 1      ; ตำแหน่ง y
    .width:     resd 1      ; ความกว้าง
    .height:    resd 1      ; ความสูง
endstruc
; Rectangle_size = 16 bytes

section .data
    rect1:
        istruc Rectangle
            at Rectangle.x,      dd  10
            at Rectangle.y,      dd  20
            at Rectangle.width,  dd  100
            at Rectangle.height, dd  50
        iend

section .text
    global _start

; ====================================================
; ฟังก์ชัน: คำนวณพื้นที่สี่เหลี่ยม
; Parameter: rdi = pointer ไปยัง Rectangle struct
; Return:    rax = พื้นที่ (width * height)
; ====================================================
calc_area:
    push rbp
    mov rbp, rsp
    
    ; rdi มี pointer ไปยัง Rectangle
    ; เข้าถึง width และ height ด้วย offset
    mov eax, [rdi + Rectangle.width]    ; eax = width
    mov ecx, [rdi + Rectangle.height]   ; ecx = height
    imul eax, ecx                        ; eax = width * height
    
    pop rbp
    ret

; ====================================================
; ฟังก์ชัน: ย้ายสี่เหลี่ยม (translate)
; Parameter: rdi = pointer ไปยัง Rectangle
;            esi = delta_x
;            edx = delta_y
; ====================================================
translate_rect:
    push rbp
    mov rbp, rsp
    
    ; เพิ่มค่า x
    add [rdi + Rectangle.x], esi
    
    ; เพิ่มค่า y
    add [rdi + Rectangle.y], edx
    
    pop rbp
    ret

; ====================================================
; ฟังก์ชัน: ตรวจสอบว่าจุดอยู่ใน Rectangle หรือไม่
; Parameter: rdi = pointer ไปยัง Rectangle
;            esi = point_x
;            edx = point_y
; Return:    rax = 1 ถ้าอยู่ใน, 0 ถ้าไม่อยู่ใน
; ====================================================
point_in_rect:
    push rbp
    mov rbp, rsp
    
    xor eax, eax    ; ตั้งค่าเริ่มต้น = false
    
    ; ตรวจสอบ x: rect.x <= point_x < rect.x + rect.width
    mov ecx, [rdi + Rectangle.x]
    cmp esi, ecx
    jl  .not_in         ; ถ้า point_x < rect.x → ไม่อยู่ใน
    
    mov r8d, ecx
    add r8d, [rdi + Rectangle.width]
    cmp esi, r8d
    jge .not_in         ; ถ้า point_x >= rect.x + width → ไม่อยู่ใน
    
    ; ตรวจสอบ y: rect.y <= point_y < rect.y + rect.height
    mov ecx, [rdi + Rectangle.y]
    cmp edx, ecx
    jl  .not_in         ; ถ้า point_y < rect.y → ไม่อยู่ใน
    
    mov r8d, ecx
    add r8d, [rdi + Rectangle.height]
    cmp edx, r8d
    jge .not_in         ; ถ้า point_y >= rect.y + height → ไม่อยู่ใน
    
    mov eax, 1          ; อยู่ใน rectangle
    jmp .done

.not_in:
    xor eax, eax        ; ไม่อยู่ใน rectangle

.done:
    pop rbp
    ret

_start:
    ; ทดสอบ calc_area
    lea rdi, [rect1]        ; โหลด address ของ rect1
    call calc_area          ; rax = 100 * 50 = 5000
    
    ; ทดสอบ translate_rect
    lea rdi, [rect1]
    mov esi, 5              ; delta_x = 5
    mov edx, 10             ; delta_y = 10
    call translate_rect
    ; ตอนนี้ rect1.x = 15, rect1.y = 30
    
    ; ทดสอบ point_in_rect
    lea rdi, [rect1]
    mov esi, 20             ; point_x = 20
    mov edx, 40             ; point_y = 40
    call point_in_rect      ; rax ควรเป็น 1 (อยู่ใน)
    
    mov eax, 60
    xor edi, edi
    syscall
```

---

## 4. Nested Structures (โครงสร้างซ้อนกัน)

```nasm
; ไฟล์: nested_struct.asm
; คอมไพล์: nasm -f elf64 nested_struct.asm -o nested_struct.o
;          ld nested_struct.o -o nested_struct

; ====================================================
; กำหนด structures ซ้อนกัน
; ====================================================

; Address structure
struc Address
    .street_num:    resd 1      ; เลขที่บ้าน (4 bytes)
    .zip_code:      resd 1      ; รหัสไปรษณีย์ (4 bytes)
    .city_id:       resw 1      ; รหัสเมือง (2 bytes)
    .country_id:    resw 1      ; รหัสประเทศ (2 bytes)
endstruc
; Address_size = 12 bytes

; Employee structure ที่มี Address ซ้อนอยู่
struc Employee
    .emp_id:        resd 1              ; รหัสพนักงาน (4 bytes)
    .department:    resw 1              ; แผนก (2 bytes)
    .salary:        resq 1              ; เงินเดือน (8 bytes)
    .address:       resb Address_size  ; ที่อยู่ (12 bytes)
    .years:         resb 1              ; อายุงาน (1 byte)
endstruc
; Employee_size = 4+2+8+12+1 = 27 bytes (ก่อน align)

section .data
    emp1:
        istruc Employee
            at Employee.emp_id,         dd  2001
            at Employee.department,     dw  5          ; แผนก 5
            at Employee.salary,         dq  50000      ; 50,000 บาท
            ; กรอกข้อมูล address ใช้ offset manual
            at Employee.address,        dd  123, 10500, 1, 66
            at Employee.years,          db  3
        iend

section .text
    global _start

; ====================================================
; เข้าถึง nested struct member
; rdi = pointer ไปยัง Employee
; ====================================================
get_emp_zip:
    ; เข้าถึง address.zip_code ใน Employee
    ; offset = Employee.address + Address.zip_code
    mov eax, [rdi + Employee.address + Address.zip_code]
    ret

get_emp_salary:
    ; เข้าถึง salary ของ employee
    mov rax, [rdi + Employee.salary]
    ret

_start:
    lea rdi, [emp1]
    
    ; ดึงรหัสไปรษณีย์
    call get_emp_zip     ; rax = 10500
    
    ; ดึงเงินเดือน
    call get_emp_salary  ; rax = 50000
    
    ; เข้าถึง nested struct โดยตรง
    mov eax, [emp1 + Employee.address + Address.street_num]  ; = 123
    
    mov eax, 60
    xor edi, edi
    syscall
```

---

## 5. Struct Padding และ Alignment

### 5.1 ทำความเข้าใจ Alignment

```nasm
; ไฟล์: struct_padding.asm
; แสดงให้เห็นว่า struct padding ทำงานอย่างไร

; ====================================================
; Struct ที่มีปัญหา alignment (ไม่ efficient)
; ====================================================
struc BadStruct
    .flag:      resb 1      ; byte (offset 0)
    ; 3 bytes padding ถูก add โดย compiler ใน C
    ; แต่ใน NASM เราต้องจัดการเอง
    .value:     resd 1      ; dword ควรอยู่ที่ offset 4 (ไม่ใช่ 1)
    .data:      resw 1      ; word ควรอยู่ที่ offset 8
    ; 2 bytes padding
    .bignum:    resq 1      ; qword ควรอยู่ที่ offset 16
endstruc
; BadStruct_size = ขึ้นอยู่กับ NASM ไม่ใส่ padding อัตโนมัติ

; ====================================================
; Struct ที่ทำ alignment ถูกต้องด้วยมือ
; ====================================================
struc GoodStruct
    .flag:      resb 1      ; byte ที่ offset 0
    .pad1:      resb 3      ; padding 3 bytes เพื่อ align .value
    .value:     resd 1      ; dword ที่ offset 4 (aligned ที่ 4)
    .data:      resw 1      ; word ที่ offset 8 (aligned ที่ 2)
    .pad2:      resw 1      ; padding 2 bytes เพื่อ align .bignum
    .bignum:    resq 1      ; qword ที่ offset 12 → ควรเป็น 16
endstruc

; ====================================================
; Struct ที่ align ถูกต้องสมบูรณ์
; ====================================================
struc AlignedStruct
    .bignum:    resq 1      ; 8 bytes ที่ offset 0 (aligned ที่ 8)
    .value:     resd 1      ; 4 bytes ที่ offset 8 (aligned ที่ 4)
    .data:      resw 1      ; 2 bytes ที่ offset 12 (aligned ที่ 2)
    .flag:      resb 1      ; 1 byte ที่ offset 14
    .pad:       resb 1      ; 1 byte padding เพื่อให้ size เป็น 16
endstruc
; AlignedStruct_size = 16 bytes (optimal)

section .data
    ; ตัวอย่างการใช้งาน aligned struct
    good_data:
        istruc AlignedStruct
            at AlignedStruct.bignum,  dq  0x123456789ABCDEF0
            at AlignedStruct.value,   dd  42
            at AlignedStruct.data,    dw  1000
            at AlignedStruct.flag,    db  1
            at AlignedStruct.pad,     db  0
        iend

section .text
    global _start

_start:
    ; การเข้าถึง aligned struct มีประสิทธิภาพสูงสุด
    mov rax, [good_data + AlignedStruct.bignum]   ; aligned access
    mov ecx, [good_data + AlignedStruct.value]    ; aligned access
    movzx edx, word [good_data + AlignedStruct.data]  ; aligned access
    
    mov eax, 60
    xor edi, edi
    syscall
```

---

## 6. Packed Structs

```nasm
; ไฟล์: packed_struct.asm
; Packed struct ไม่มี padding เลย ประหยัดพื้นที่แต่อาจช้ากว่า

; ====================================================
; Packed Network Packet Header
; ใช้ทุก bit อย่างมีประสิทธิภาพ
; ====================================================
struc PackedHeader
    ; Version (4 bits) + IHL (4 bits) รวมกันใน 1 byte
    .ver_ihl:   resb 1      ; byte 0
    ; DSCP (6 bits) + ECN (2 bits) รวมกันใน 1 byte
    .dscp_ecn:  resb 1      ; byte 1
    ; Total Length (16 bits)
    .tot_len:   resw 1      ; bytes 2-3
    ; Identification (16 bits)
    .id:        resw 1      ; bytes 4-5
    ; Flags (3 bits) + Fragment Offset (13 bits) รวมกันใน word
    .frag_off:  resw 1      ; bytes 6-7
    ; TTL (8 bits)
    .ttl:       resb 1      ; byte 8
    ; Protocol (8 bits)
    .protocol:  resb 1      ; byte 9
    ; Header Checksum (16 bits)
    .check:     resw 1      ; bytes 10-11
    ; Source IP (32 bits)
    .saddr:     resd 1      ; bytes 12-15
    ; Destination IP (32 bits)
    .daddr:     resd 1      ; bytes 16-19
endstruc
; PackedHeader_size = 20 bytes (เหมือน IPv4 header จริงๆ)

section .data
    ; สร้าง packet header ตัวอย่าง
    sample_pkt:
        istruc PackedHeader
            at PackedHeader.ver_ihl,  db  0x45        ; Version=4, IHL=5
            at PackedHeader.dscp_ecn, db  0x00        ; Default DSCP/ECN
            at PackedHeader.tot_len,  dw  0x003C      ; 60 bytes total
            at PackedHeader.id,       dw  0x1234      ; ID = 0x1234
            at PackedHeader.frag_off, dw  0x4000      ; Don't Fragment bit
            at PackedHeader.ttl,      db  64          ; TTL = 64
            at PackedHeader.protocol, db  6           ; TCP = 6
            at PackedHeader.check,    dw  0           ; Checksum (คำนวณทีหลัง)
            at PackedHeader.saddr,    dd  0xC0A80001  ; 192.168.0.1
            at PackedHeader.daddr,    dd  0xC0A80002  ; 192.168.0.2
        iend

section .text
    global _start

; ====================================================
; ฟังก์ชัน: ดึง Version จาก ver_ihl field
; rdi = pointer ไปยัง PackedHeader
; return: rax = IP version (4 หรือ 6)
; ====================================================
get_ip_version:
    movzx eax, byte [rdi + PackedHeader.ver_ihl]
    shr eax, 4          ; เลื่อนขวา 4 bits เพื่อได้ version
    and eax, 0x0F       ; mask เอาแค่ 4 bits
    ret

; ====================================================
; ฟังก์ชัน: ดึง IHL จาก ver_ihl field
; rdi = pointer ไปยัง PackedHeader
; return: rax = IHL (จำนวน 32-bit words ใน header)
; ====================================================
get_ihl:
    movzx eax, byte [rdi + PackedHeader.ver_ihl]
    and eax, 0x0F       ; mask เอาแค่ 4 bits ล่าง
    ret

; ====================================================
; ฟังก์ชัน: ตรวจสอบ Don't Fragment bit
; rdi = pointer ไปยัง PackedHeader
; return: rax = 1 ถ้า DF set, 0 ถ้าไม่
; ====================================================
check_dont_fragment:
    movzx eax, word [rdi + PackedHeader.frag_off]
    xchg al, ah         ; swap bytes (big-endian)
    shr eax, 14         ; DF bit อยู่ที่ bit 14
    and eax, 1          ; เอาแค่ 1 bit
    ret

_start:
    lea rdi, [sample_pkt]
    
    call get_ip_version     ; rax = 4 (IPv4)
    call get_ihl            ; rax = 5 (5 * 4 = 20 bytes)
    call check_dont_fragment ; rax = 1 (DF set)
    
    mov eax, 60
    xor edi, edi
    syscall
```

---

## 7. Arrays of Structs

```nasm
; ไฟล์: struct_array.asm
; คอมไพล์: nasm -f elf64 struct_array.asm -o struct_array.o
;          ld struct_array.o -o struct_array

; ====================================================
; Student structure
; ====================================================
struc Student
    .id:        resd 1      ; รหัสนักศึกษา (4 bytes)
    .grade:     resb 1      ; เกรด A-F (1 byte)
    .year:      resb 1      ; ปีการศึกษา 1-4 (1 byte)
    .score:     resw 1      ; คะแนน 0-1000 (2 bytes)
endstruc
; Student_size = 8 bytes

MAX_STUDENTS    equ 10  ; จำนวนนักศึกษาสูงสุด

section .data
    ; อาร์เรย์ของ students
    students:
        ; Student 0
        istruc Student
            at Student.id,    dd  65001
            at Student.grade, db  'A'
            at Student.year,  db  2
            at Student.score, dw  950
        iend
        ; Student 1
        istruc Student
            at Student.id,    dd  65002
            at Student.grade, db  'B'
            at Student.year,  db  1
            at Student.score, dw  820
        iend
        ; Student 2
        istruc Student
            at Student.id,    dd  65003
            at Student.grade, db  'A'
            at Student.year,  db  3
            at Student.score, dw  980
        iend
        ; Student 3
        istruc Student
            at Student.id,    dd  65004
            at Student.grade, db  'C'
            at Student.year,  db  4
            at Student.score, dw  720
        iend
        ; Student 4
        istruc Student
            at Student.id,    dd  65005
            at Student.grade, db  'B'
            at Student.year,  db  2
            at Student.score, dw  850
        iend

    num_students    dd  5   ; จำนวนนักศึกษาจริง

section .bss
    max_score_idx   resd 1  ; index ของนักศึกษาที่ได้คะแนนสูงสุด

section .text
    global _start

; ====================================================
; ฟังก์ชัน: หา index ของนักศึกษาที่ได้คะแนนสูงสุด
; rdi = pointer ไปยัง array ของ Student
; esi = จำนวนนักศึกษา
; return: rax = index ของนักศึกษาที่ได้คะแนนสูงสุด
; ====================================================
find_max_score:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    
    xor rbx, rbx        ; rbx = current index (i = 0)
    xor r12, r12        ; r12 = best_idx = 0
    movzx r13d, word [rdi + Student.score]  ; r13 = max_score = students[0].score
    
    inc rbx             ; เริ่มจาก index 1
    
.loop:
    cmp ebx, esi
    jge .done           ; ถ้า i >= num_students → จบ
    
    ; คำนวณ offset: rdi + rbx * Student_size
    mov rax, rbx
    imul rax, Student_size  ; rax = index * struct_size
    
    ; อ่านคะแนนของนักศึกษา i
    movzx ecx, word [rdi + rax + Student.score]
    
    ; เปรียบเทียบกับ max_score
    cmp ecx, r13d
    jle .next           ; ถ้าไม่มากกว่า → ข้ามไป
    
    ; อัปเดต max
    mov r13d, ecx
    mov r12, rbx        ; best_idx = i

.next:
    inc rbx
    jmp .loop

.done:
    mov rax, r12        ; return best_idx
    
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

; ====================================================
; ฟังก์ชัน: นับจำนวนนักศึกษาเกรด A
; rdi = pointer ไปยัง array ของ Student
; esi = จำนวนนักศึกษา
; return: rax = จำนวนนักศึกษาเกรด A
; ====================================================
count_grade_a:
    push rbp
    mov rbp, rsp
    push rbx
    
    xor rbx, rbx        ; i = 0
    xor ecx, ecx        ; count = 0
    
.loop:
    cmp ebx, esi
    jge .done
    
    mov rax, rbx
    imul rax, Student_size
    
    movzx edx, byte [rdi + rax + Student.grade]
    cmp dl, 'A'
    jne .next
    
    inc ecx             ; count++

.next:
    inc rbx
    jmp .loop

.done:
    mov eax, ecx
    pop rbx
    pop rbp
    ret

_start:
    ; หานักศึกษาที่ได้คะแนนสูงสุด
    lea rdi, [students]
    mov esi, [num_students]
    call find_max_score
    ; rax = 2 (student index 2 ได้คะแนน 980)
    
    mov [max_score_idx], eax
    
    ; นับจำนวนนักศึกษาเกรด A
    lea rdi, [students]
    mov esi, [num_students]
    call count_grade_a
    ; rax = 2 (students[0] และ students[2])
    
    mov eax, 60
    xor edi, edi
    syscall
```

---

## 8. Struct Pointers

```nasm
; ไฟล์: struct_pointer.asm
; การใช้ pointer เพื่อชี้ไปยัง struct ต่างๆ

struc Shape
    .type:      resb 1      ; 0=circle, 1=rect, 2=triangle
    .color:     resb 1      ; สี (index)
    .pad:       resw 1      ; padding
    .area:      resq 1      ; พื้นที่ (float ในรูป bits)
endstruc

struc Circle
    .base:      resb Shape_size  ; embed Shape base
    .cx:        resd 1      ; center x
    .cy:        resd 1      ; center y
    .radius:    resd 1      ; รัศมี
endstruc

struc Rect
    .base:      resb Shape_size  ; embed Shape base
    .x:         resd 1
    .y:         resd 1
    .w:         resd 1
    .h:         resd 1
endstruc

section .data
    ; สร้าง shapes
    circle1:
        istruc Circle
            at Circle.base,   db 0, 2, 0, 0    ; type=circle, color=2, pad
                              dq 0             ; area (จะคำนวณทีหลัง)
            at Circle.cx,     dd 50
            at Circle.cy,     dd 50
            at Circle.radius, dd 30
        iend

    rect1:
        istruc Rect
            at Rect.base,     db 1, 5, 0, 0    ; type=rect, color=5, pad
                              dq 0             ; area
            at Rect.x,        dd 10
            at Rect.y,        dd 10
            at Rect.w,        dd 80
            at Rect.h,        dd 40
        iend

    ; Array ของ pointers ไปยัง shapes
    shapes_ptr:
        dq  circle1
        dq  rect1

    num_shapes  dq  2

section .text
    global _start

; ====================================================
; ฟังก์ชัน: ดึง shape type ผ่าน pointer
; rdi = pointer ไปยัง Shape (หรือ subtype)
; return: rax = shape type
; ====================================================
get_shape_type:
    movzx eax, byte [rdi + Shape.type]
    ret

; ====================================================
; ฟังก์ชัน: iterate ผ่าน array ของ pointers
; rdi = pointer ไปยัง array ของ pointers
; esi = จำนวน shapes
; ====================================================
print_shape_types:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    
    mov r12d, esi       ; r12 = count
    xor ebx, ebx        ; i = 0

.loop:
    cmp ebx, r12d
    jge .done
    
    ; โหลด pointer จาก array (แต่ละ entry เป็น qword)
    mov rax, rbx
    imul rax, 8             ; offset = i * 8 (pointer size)
    mov rsi, [rdi + rax]    ; rsi = shapes_ptr[i]
    
    ; เรียก get_shape_type ด้วย shape pointer
    push rdi
    push rbx
    mov rdi, rsi
    call get_shape_type
    pop rbx
    pop rdi
    ; rax = shape type
    
    inc ebx
    jmp .loop

.done:
    pop r12
    pop rbx
    pop rbp
    ret

_start:
    lea rdi, [shapes_ptr]
    mov esi, 2
    call print_shape_types
    
    mov eax, 60
    xor edi, edi
    syscall
```

---

## 9. Linked List ด้วย STRUC

```nasm
; ไฟล์: linked_list_struct.asm
; คอมไพล์: nasm -f elf64 linked_list_struct.asm -o linked_list_struct.o
;          ld linked_list_struct.o -o linked_list_struct

; ====================================================
; Node structure สำหรับ Singly Linked List
; ====================================================
struc ListNode
    .data:  resd 1      ; ข้อมูล (4 bytes)
    .pad:   resd 1      ; padding เพื่อ align .next
    .next:  resq 1      ; pointer ไปยัง node ถัดไป (8 bytes)
endstruc
; ListNode_size = 16 bytes

NULL    equ 0   ; Null pointer

section .data
    ; สร้าง linked list: 10 → 20 → 30 → 40 → NULL
    node4:
        istruc ListNode
            at ListNode.data, dd 40
            at ListNode.pad,  dd 0
            at ListNode.next, dq NULL       ; tail node
        iend

    node3:
        istruc ListNode
            at ListNode.data, dd 30
            at ListNode.pad,  dd 0
            at ListNode.next, dq node4
        iend

    node2:
        istruc ListNode
            at ListNode.data, dd 20
            at ListNode.pad,  dd 0
            at ListNode.next, dq node3
        iend

    node1:
        istruc ListNode
            at ListNode.data, dd 10
            at ListNode.pad,  dd 0
            at ListNode.next, dq node2
        iend

    head    dq node1    ; pointer ไปยัง head ของ list

section .bss
    sum     resd 1      ; ผลรวมของ list

section .text
    global _start

; ====================================================
; ฟังก์ชัน: หาผลรวมของ linked list
; rdi = pointer ไปยัง head node (หรือ NULL)
; return: rax = ผลรวม
; ====================================================
list_sum:
    push rbp
    mov rbp, rsp
    
    xor eax, eax        ; sum = 0
    
.loop:
    test rdi, rdi       ; ตรวจสอบว่า node เป็น NULL หรือไม่
    jz  .done           ; ถ้า NULL → จบ
    
    add eax, [rdi + ListNode.data]      ; sum += node->data
    mov rdi, [rdi + ListNode.next]      ; node = node->next
    jmp .loop

.done:
    pop rbp
    ret

; ====================================================
; ฟังก์ชัน: หาค่าสูงสุดใน linked list
; rdi = pointer ไปยัง head node (ต้องไม่ใช่ NULL)
; return: rax = ค่าสูงสุด
; ====================================================
list_max:
    push rbp
    mov rbp, rsp
    
    ; เริ่มต้นด้วย head->data
    mov eax, [rdi + ListNode.data]
    mov rdi, [rdi + ListNode.next]  ; ไปยัง node ถัดไป

.loop:
    test rdi, rdi
    jz  .done
    
    mov ecx, [rdi + ListNode.data]
    cmp ecx, eax
    jle .next           ; ถ้า node->data <= max → ข้าม
    mov eax, ecx        ; อัปเดต max

.next:
    mov rdi, [rdi + ListNode.next]
    jmp .loop

.done:
    pop rbp
    ret

; ====================================================
; ฟังก์ชัน: นับจำนวน nodes ใน linked list
; rdi = head pointer
; return: rax = จำนวน nodes
; ====================================================
list_count:
    xor eax, eax

.loop:
    test rdi, rdi
    jz .done
    inc eax
    mov rdi, [rdi + ListNode.next]
    jmp .loop

.done:
    ret

; ====================================================
; ฟังก์ชัน: ค้นหาค่าใน linked list
; rdi = head pointer
; esi = ค่าที่ค้นหา
; return: rax = pointer ไปยัง node ที่พบ, หรือ 0 ถ้าไม่พบ
; ====================================================
list_find:
.loop:
    test rdi, rdi
    jz  .not_found
    
    cmp [rdi + ListNode.data], esi
    je  .found
    
    mov rdi, [rdi + ListNode.next]
    jmp .loop

.found:
    mov rax, rdi        ; คืน pointer ของ node ที่พบ
    ret

.not_found:
    xor eax, eax        ; คืน NULL
    ret

_start:
    ; หาผลรวม: 10+20+30+40 = 100
    mov rdi, [head]
    call list_sum
    ; rax = 100
    mov [sum], eax
    
    ; หาค่าสูงสุด
    mov rdi, [head]
    call list_max
    ; rax = 40
    
    ; นับจำนวน nodes
    mov rdi, [head]
    call list_count
    ; rax = 4
    
    ; ค้นหาค่า 30
    mov rdi, [head]
    mov esi, 30
    call list_find
    ; rax = address ของ node3
    
    mov eax, 60
    xor edi, edi
    syscall
```

---

## 10. Binary Tree ด้วย STRUC

```nasm
; ไฟล์: binary_tree_struct.asm
; คอมไพล์: nasm -f elf64 binary_tree_struct.asm -o binary_tree_struct.o
;          ld binary_tree_struct.o -o binary_tree_struct

; ====================================================
; TreeNode structure สำหรับ Binary Search Tree
; ====================================================
struc TreeNode
    .key:   resd 1      ; key (4 bytes)
    .pad:   resd 1      ; padding
    .left:  resq 1      ; pointer ไปยัง left child (8 bytes)
    .right: resq 1      ; pointer ไปยัง right child (8 bytes)
endstruc
; TreeNode_size = 24 bytes

NULL equ 0

section .data
    ;
    ;         50
    ;        /  \
    ;       30   70
    ;      / \  / \
    ;    20  40 60  80
    ;
    ; BST ที่สร้างไว้แล้ว

    node20:
        istruc TreeNode
            at TreeNode.key,   dd 20
            at TreeNode.pad,   dd 0
            at TreeNode.left,  dq NULL
            at TreeNode.right, dq NULL
        iend

    node40:
        istruc TreeNode
            at TreeNode.key,   dd 40
            at TreeNode.pad,   dd 0
            at TreeNode.left,  dq NULL
            at TreeNode.right, dq NULL
        iend

    node60:
        istruc TreeNode
            at TreeNode.key,   dd 60
            at TreeNode.pad,   dd 0
            at TreeNode.left,  dq NULL
            at TreeNode.right, dq NULL
        iend

    node80:
        istruc TreeNode
            at TreeNode.key,   dd 80
            at TreeNode.pad,   dd 0
            at TreeNode.left,  dq NULL
            at TreeNode.right, dq NULL
        iend

    node30:
        istruc TreeNode
            at TreeNode.key,   dd 30
            at TreeNode.pad,   dd 0
            at TreeNode.left,  dq node20
            at TreeNode.right, dq node40
        iend

    node70:
        istruc TreeNode
            at TreeNode.key,   dd 70
            at TreeNode.pad,   dd 0
            at TreeNode.left,  dq node60
            at TreeNode.right, dq node80
        iend

    root:
        istruc TreeNode
            at TreeNode.key,   dd 50
            at TreeNode.pad,   dd 0
            at TreeNode.left,  dq node30
            at TreeNode.right, dq node70
        iend

    tree_root   dq root     ; pointer ไปยัง root

section .bss
    inorder_buf resb 100    ; buffer สำหรับเก็บ inorder traversal

section .text
    global _start

; ====================================================
; ฟังก์ชัน: ค้นหาใน BST
; rdi = root pointer
; esi = ค่าที่ค้นหา
; return: rax = pointer ไปยัง node (หรือ 0 ถ้าไม่พบ)
; ====================================================
bst_search:
.loop:
    test rdi, rdi       ; ตรวจสอบ NULL
    jz  .not_found
    
    cmp esi, [rdi + TreeNode.key]
    je  .found
    jl  .go_left
    
.go_right:
    mov rdi, [rdi + TreeNode.right]
    jmp .loop

.go_left:
    mov rdi, [rdi + TreeNode.left]
    jmp .loop

.found:
    mov rax, rdi
    ret

.not_found:
    xor eax, eax
    ret

; ====================================================
; ฟังก์ชัน: หาค่าน้อยสุดใน BST
; rdi = root pointer (ต้องไม่ใช่ NULL)
; return: rax = ค่าน้อยสุด
; ====================================================
bst_min:
.loop:
    mov rax, rdi                    ; เก็บ current node
    mov rdi, [rdi + TreeNode.left]  ; ไปซ้าย
    test rdi, rdi
    jnz .loop                       ; ถ้าไม่ NULL → ไปต่อ
    
    mov eax, [rax + TreeNode.key]   ; คืนค่า key ของ leftmost node
    ret

; ====================================================
; ฟังก์ชัน: หาค่ามากสุดใน BST
; rdi = root pointer (ต้องไม่ใช่ NULL)
; return: rax = ค่ามากสุด
; ====================================================
bst_max:
.loop:
    mov rax, rdi
    mov rdi, [rdi + TreeNode.right]
    test rdi, rdi
    jnz .loop
    
    mov eax, [rax + TreeNode.key]
    ret

; ====================================================
; ฟังก์ชัน: คำนวณ height ของ BST (recursive)
; rdi = node pointer (อาจเป็น NULL)
; return: rax = height (0 ถ้า NULL)
; ====================================================
bst_height:
    test rdi, rdi
    jz  .null_case
    
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    
    mov rbx, rdi        ; เก็บ current node
    
    ; คำนวณ left height
    mov rdi, [rbx + TreeNode.left]
    call bst_height
    mov r12, rax        ; r12 = left_height
    
    ; คำนวณ right height
    mov rdi, [rbx + TreeNode.right]
    call bst_height
    ; rax = right_height
    
    ; height = max(left_height, right_height) + 1
    cmp r12, rax
    jge .use_left
    
.use_right:
    inc rax             ; height = right_height + 1
    jmp .done

.use_left:
    mov rax, r12
    inc rax             ; height = left_height + 1

.done:
    pop r12
    pop rbx
    pop rbp
    ret

.null_case:
    xor eax, eax        ; height ของ NULL tree = 0
    ret

_start:
    ; ค้นหาค่า 60 ใน BST
    mov rdi, [tree_root]
    mov esi, 60
    call bst_search
    ; rax = address ของ node60
    
    ; หาค่าน้อยสุด
    mov rdi, [tree_root]
    call bst_min
    ; rax = 20
    
    ; หาค่ามากสุด
    mov rdi, [tree_root]
    call bst_max
    ; rax = 80
    
    ; คำนวณ height
    mov rdi, [tree_root]
    call bst_height
    ; rax = 3 (root → 30 → 20)
    
    mov eax, 60
    xor edi, edi
    syscall
```

---

## 11. Union Equivalent ใน NASM

```nasm
; ไฟล์: union_nasm.asm
; NASM ไม่มี union keyword แต่เราจำลองได้ด้วย resb

; ====================================================
; Union simulation ใน NASM
; Union คือหลาย fields ใช้ memory เดียวกัน
; ====================================================

; ใน C:
; union Data {
;     uint8_t  b;      // 1 byte
;     uint16_t w;      // 2 bytes
;     uint32_t d;      // 4 bytes
;     uint64_t q;      // 8 bytes
;     float    f;      // 4 bytes
;     double   df;     // 8 bytes
; };

struc UnionData
    .storage:   resb 8      ; จอง 8 bytes (ขนาดของ member ใหญ่สุด)
endstruc

; Accessor macros (กำหนด offset ให้เป็น 0 ทั้งหมด)
%define UnionData.b     UnionData.storage + 0
%define UnionData.w     UnionData.storage + 0
%define UnionData.d     UnionData.storage + 0
%define UnionData.q     UnionData.storage + 0
; Note: ใน NASM ทั้งหมดชี้ไปยัง offset 0 ซึ่งคือจุดเริ่มต้น

; ====================================================
; IP Address Union: ใช้ทั้ง 4 bytes แยกหรือ dword รวม
; ====================================================
struc IPAddr
    .as_dword:  resd 1      ; ทั้ง 4 bytes รวมกัน (เหมือน uint32_t)
endstruc
; เข้าถึง byte แต่ละตัวด้วย offset เพิ่ม

section .data
    ; IP Address: 192.168.1.100
    ip_addr:
        istruc IPAddr
            at IPAddr.as_dword, dd  0xC0A80164  ; 192.168.1.100
        iend

    union_test:
        istruc UnionData
            at UnionData.storage, dq 0x0102030405060708
        iend

section .text
    global _start

; ====================================================
; แสดงการใช้ Union สำหรับ IP Address
; ====================================================
get_ip_byte:
    ; rdi = pointer ไปยัง IPAddr
    ; esi = byte index (0-3)
    movzx eax, byte [rdi + IPAddr.as_dword + rsi]
    ret

_start:
    ; อ่านค่า union ในรูปแบบต่างๆ
    movzx eax, byte  [union_test + UnionData.storage]   ; byte = 0x08
    movzx eax, word  [union_test + UnionData.storage]   ; word = 0x0708
    mov   eax, dword [union_test + UnionData.storage]   ; dword = 0x05060708
    mov   rax, qword [union_test + UnionData.storage]   ; qword = ทั้งหมด
    
    ; อ่าน IP address ทีละ byte
    lea rdi, [ip_addr]
    mov esi, 0
    call get_ip_byte    ; rax = 100 (byte 0 = 0x64)
    
    mov esi, 1
    call get_ip_byte    ; rax = 1 (byte 1 = 0x01)
    
    mov esi, 2
    call get_ip_byte    ; rax = 168 (byte 2 = 0xA8)
    
    mov esi, 3
    call get_ip_byte    ; rax = 192 (byte 3 = 0xC0)
    
    mov eax, 60
    xor edi, edi
    syscall
```

---

## 12. Struct Size Calculation

```nasm
; ไฟล์: struct_size.asm
; การคำนวณขนาดของ struct และการใช้ size constants

; ====================================================
; กำหนด structs หลายขนาด
; ====================================================
struc Tiny
    .a: resb 1
    .b: resb 1
endstruc
; Tiny_size = 2

struc Small
    .x: resw 1
    .y: resw 1
endstruc
; Small_size = 4

struc Medium
    .id:    resd 1
    .name:  resb 32
    .score: resw 1
    .grade: resb 1
    .pad:   resb 1
endstruc
; Medium_size = 4 + 32 + 2 + 1 + 1 = 40

struc Large
    .header:    resq 2      ; 16 bytes
    .data:      resb 64     ; 64 bytes
    .checksum:  resd 1      ; 4 bytes
    .flags:     resd 1      ; 4 bytes
endstruc
; Large_size = 88 bytes

section .data
    ; แสดงขนาดของ structs (ใช้สำหรับ debug)
    size_tiny   dd  Tiny_size       ; = 2
    size_small  dd  Small_size      ; = 4
    size_medium dd  Medium_size     ; = 40
    size_large  dd  Large_size      ; = 88

section .bss
    ; จอง array โดยใช้ struct size
    tiny_arr    resb (Tiny_size * 10)   ; array ของ 10 Tiny structs
    medium_arr  resb (Medium_size * 5)  ; array ของ 5 Medium structs

section .text
    global _start

; ====================================================
; ฟังก์ชัน: เข้าถึง element ที่ index ใน array
; rdi = base address ของ array
; esi = index
; rdx = struct size
; return: rax = pointer ไปยัง element
; ====================================================
array_element:
    mov rax, rsi            ; rax = index
    imul rax, rdx           ; rax = index * struct_size
    add rax, rdi            ; rax = base + offset
    ret

_start:
    ; เข้าถึง medium_arr[2]
    lea rdi, [medium_arr]
    mov esi, 2              ; index = 2
    mov edx, Medium_size    ; struct_size = 40
    call array_element
    ; rax = &medium_arr[2] = medium_arr + 80
    
    ; กำหนดค่า id ของ medium_arr[2]
    mov dword [rax + Medium.id], 12345
    
    ; เข้าถึง medium_arr[0]
    lea rdi, [medium_arr]
    mov esi, 0
    mov edx, Medium_size
    call array_element
    mov dword [rax + Medium.id], 99
    
    mov eax, 60
    xor edi, edi
    syscall
```

---

## 13. Struct เป็น Function Parameter

```nasm
; ไฟล์: struct_param.asm
; คอมไพล์: nasm -f elf64 struct_param.asm -o struct_param.o
;          ld struct_param.o -o struct_param
; การส่ง struct เป็น argument ให้ฟังก์ชัน (ส่ง pointer)

; ====================================================
; Vector3D structure
; ====================================================
struc Vector3D
    .x: resq 1      ; double x (8 bytes)
    .y: resq 1      ; double y
    .z: resq 1      ; double z
endstruc
; Vector3D_size = 24 bytes

section .data
    ; ตัวอย่าง vectors
    vec_a:
        istruc Vector3D
            at Vector3D.x, dq 1.0   ; x = 1.0
            at Vector3D.y, dq 2.0   ; y = 2.0
            at Vector3D.z, dq 3.0   ; z = 3.0
        iend

    vec_b:
        istruc Vector3D
            at Vector3D.x, dq 4.0   ; x = 4.0
            at Vector3D.y, dq 5.0   ; y = 5.0
            at Vector3D.z, dq 6.0   ; z = 6.0
        iend

section .bss
    vec_result: resb Vector3D_size  ; ผลลัพธ์

section .text
    global _start

; ====================================================
; ฟังก์ชัน: บวก Vector3D สองตัว
; rdi = pointer ไปยัง Vector A
; rsi = pointer ไปยัง Vector B
; rdx = pointer ไปยัง Vector ผลลัพธ์
; ====================================================
vec3d_add:
    ; โหลดค่า x
    movsd xmm0, [rdi + Vector3D.x]
    addsd xmm0, [rsi + Vector3D.x]
    movsd [rdx + Vector3D.x], xmm0
    
    ; โหลดค่า y
    movsd xmm1, [rdi + Vector3D.y]
    addsd xmm1, [rsi + Vector3D.y]
    movsd [rdx + Vector3D.y], xmm1
    
    ; โหลดค่า z
    movsd xmm2, [rdi + Vector3D.z]
    addsd xmm2, [rsi + Vector3D.z]
    movsd [rdx + Vector3D.z], xmm2
    ret

; ====================================================
; ฟังก์ชัน: คูณ Vector3D ด้วย scalar
; rdi = pointer ไปยัง Vector (จะถูกแก้ไข in-place)
; xmm0 = scalar
; ====================================================
vec3d_scale:
    mulsd xmm0, [rdi + Vector3D.x]
    movsd [rdi + Vector3D.x], xmm0
    
    movsd xmm1, [rdi + Vector3D.y]
    mulsd xmm1, xmm0                ; xmm1 = y * (x_result) → ผิด!
    ; ต้อง reload scalar
    ret

; ====================================================
; ฟังก์ชัน: คูณ Vector3D ด้วย scalar (ถูกต้อง)
; rdi = pointer ไปยัง Vector (in-place)
; xmm0 = scalar
; ====================================================
vec3d_scale_correct:
    movsd xmm1, xmm0    ; เก็บ scalar ไว้

    movsd xmm2, [rdi + Vector3D.x]
    mulsd xmm2, xmm1
    movsd [rdi + Vector3D.x], xmm2

    movsd xmm2, [rdi + Vector3D.y]
    mulsd xmm2, xmm1
    movsd [rdi + Vector3D.y], xmm2

    movsd xmm2, [rdi + Vector3D.z]
    mulsd xmm2, xmm1
    movsd [rdi + Vector3D.z], xmm2
    ret

; ====================================================
; ฟังก์ชัน: Dot Product ของสอง Vector3D
; rdi = pointer ไปยัง Vector A
; rsi = pointer ไปยัง Vector B
; return: xmm0 = dot product (A.x*B.x + A.y*B.y + A.z*B.z)
; ====================================================
vec3d_dot:
    movsd xmm0, [rdi + Vector3D.x]
    mulsd xmm0, [rsi + Vector3D.x]     ; xmm0 = ax * bx
    
    movsd xmm1, [rdi + Vector3D.y]
    mulsd xmm1, [rsi + Vector3D.y]     ; xmm1 = ay * by
    
    movsd xmm2, [rdi + Vector3D.z]
    mulsd xmm2, [rsi + Vector3D.z]     ; xmm2 = az * bz
    
    addsd xmm0, xmm1    ; xmm0 = ax*bx + ay*by
    addsd xmm0, xmm2    ; xmm0 = ax*bx + ay*by + az*bz
    ret

_start:
    ; บวก vec_a + vec_b → vec_result
    lea rdi, [vec_a]
    lea rsi, [vec_b]
    lea rdx, [vec_result]
    call vec3d_add
    ; vec_result = (5.0, 7.0, 9.0)
    
    ; คำนวณ dot product
    lea rdi, [vec_a]
    lea rsi, [vec_b]
    call vec3d_dot
    ; xmm0 = 1*4 + 2*5 + 3*6 = 4 + 10 + 18 = 32.0
    
    mov eax, 60
    xor edi, edi
    syscall
```

---

## 14. Dynamic Struct Allocation ด้วย mmap

```nasm
; ไฟล์: dynamic_struct.asm
; คอมไพล์: nasm -f elf64 dynamic_struct.asm -o dynamic_struct.o
;          ld dynamic_struct.o -o dynamic_struct

; ====================================================
; Node สำหรับ dynamic linked list
; ====================================================
struc DynNode
    .value: resd 1      ; ค่า (4 bytes)
    .pad:   resd 1      ; padding
    .next:  resq 1      ; next pointer (8 bytes)
endstruc
; DynNode_size = 16 bytes

; mmap system call constants
PROT_READ       equ 0x1
PROT_WRITE      equ 0x2
MAP_PRIVATE     equ 0x2
MAP_ANONYMOUS   equ 0x20
SYS_MMAP        equ 9
SYS_MUNMAP      equ 11
SYS_BRKINIT     equ 12

section .bss
    heap_ptr    resq 1      ; pointer ไปยัง heap memory ที่ allocate ไว้
    heap_used   resq 1      ; จำนวน bytes ที่ใช้ไปแล้ว
    heap_size   resq 1      ; ขนาดรวมของ heap

    list_head   resq 1      ; head ของ dynamic list

section .text
    global _start

; ====================================================
; ฟังก์ชัน: สร้าง simple heap ด้วย mmap
; edi = ขนาดที่ต้องการ (bytes)
; return: rax = pointer ไปยัง memory (หรือ -1 ถ้า error)
; ====================================================
init_heap:
    push rbp
    mov rbp, rsp
    
    ; mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)
    xor rdi, rdi                    ; addr = NULL
    movsx rsi, edi                  ; length = ขนาดที่ต้องการ
    mov rdx, PROT_READ | PROT_WRITE ; protection
    mov r10, MAP_PRIVATE | MAP_ANONYMOUS  ; flags
    mov r8, -1                      ; fd = -1
    xor r9, r9                      ; offset = 0
    mov rax, SYS_MMAP
    syscall
    
    ; ตรวจสอบ error (rax เป็น negative number ใหญ่)
    cmp rax, -4096
    ja  .error
    
    ; เก็บ heap info
    mov [heap_ptr], rax
    mov [heap_used], qword 0
    movsx rcx, edi
    mov [heap_size], rcx
    
    pop rbp
    ret

.error:
    mov rax, -1
    pop rbp
    ret

; ====================================================
; ฟังก์ชัน: allocate memory จาก heap
; rdi = ขนาดที่ต้องการ (bytes)
; return: rax = pointer (หรือ 0 ถ้า out of memory)
; ====================================================
heap_alloc:
    ; ตรวจสอบว่าเหลือพอ
    mov rax, [heap_used]
    add rax, rdi
    cmp rax, [heap_size]
    jg  .out_of_mem
    
    ; คืน pointer ปัจจุบัน
    mov rax, [heap_ptr]
    add rax, [heap_used]
    
    ; อัปเดต heap_used
    add [heap_used], rdi
    
    ret

.out_of_mem:
    xor eax, eax
    ret

; ====================================================
; ฟังก์ชัน: สร้าง node ใหม่
; edi = ค่าของ node
; return: rax = pointer ไปยัง DynNode ใหม่ (หรือ 0)
; ====================================================
create_node:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    
    mov r12d, edi           ; เก็บค่าที่จะใส่
    
    ; allocate memory สำหรับ node
    mov rdi, DynNode_size
    call heap_alloc
    
    test rax, rax
    jz  .done               ; ถ้า allocation ล้มเหลว
    
    mov rbx, rax            ; rbx = pointer ไปยัง node ใหม่
    
    ; กำหนดค่า
    mov [rbx + DynNode.value], r12d
    mov dword [rbx + DynNode.pad], 0
    mov qword [rbx + DynNode.next], 0   ; next = NULL

.done:
    mov rax, rbx
    pop r12
    pop rbx
    pop rbp
    ret

; ====================================================
; ฟังก์ชัน: เพิ่ม node เข้า list ด้านหน้า (prepend)
; rdi = ค่าที่จะเพิ่ม
; ====================================================
list_prepend:
    push rbp
    mov rbp, rsp
    push rbx
    
    ; สร้าง node ใหม่
    call create_node
    test rax, rax
    jz  .done
    
    mov rbx, rax
    ; node->next = list_head
    mov rax, [list_head]
    mov [rbx + DynNode.next], rax
    ; list_head = node
    mov [list_head], rbx

.done:
    pop rbx
    pop rbp
    ret

_start:
    ; สร้าง heap ขนาด 4096 bytes
    mov edi, 4096
    call init_heap
    
    ; สร้าง linked list: 10, 20, 30, 40
    mov edi, 10
    call list_prepend   ; list: 10
    
    mov edi, 20
    call list_prepend   ; list: 20 → 10
    
    mov edi, 30
    call list_prepend   ; list: 30 → 20 → 10
    
    mov edi, 40
    call list_prepend   ; list: 40 → 30 → 20 → 10
    
    ; traverse list และนับ
    mov rdi, [list_head]
    xor ecx, ecx
.count_loop:
    test rdi, rdi
    jz .done
    inc ecx
    mov rdi, [rdi + DynNode.next]
    jmp .count_loop

.done:
    ; ecx = 4
    
    ; munmap
    mov rdi, [heap_ptr]
    mov rsi, [heap_size]
    mov rax, SYS_MUNMAP
    syscall
    
    mov eax, 60
    xor edi, edi
    syscall
```

---

## 15. Struct Serialization/Deserialization

```nasm
; ไฟล์: struct_serial.asm
; คอมไพล์: nasm -f elf64 struct_serial.asm -o struct_serial.o
;          ld struct_serial.o -o struct_serial

; ====================================================
; Config structure ที่จะ serialize/deserialize
; ====================================================
struc Config
    .magic:     resd 1      ; magic number สำหรับตรวจสอบ (4 bytes)
    .version:   resw 1      ; version (2 bytes)
    .flags:     resw 1      ; flags (2 bytes)
    .max_conn:  resd 1      ; จำนวน connections สูงสุด (4 bytes)
    .timeout:   resd 1      ; timeout (milliseconds) (4 bytes)
    .port:      resw 1      ; port number (2 bytes)
    .pad:       resw 1      ; padding (2 bytes)
endstruc
; Config_size = 20 bytes

MAGIC_NUMBER    equ 0xDEADBEEF  ; magic number สำหรับตรวจสอบ
CONFIG_VERSION  equ 1

section .data
    ; default config
    default_config:
        istruc Config
            at Config.magic,    dd  MAGIC_NUMBER
            at Config.version,  dw  CONFIG_VERSION
            at Config.flags,    dw  0x0003          ; flags = 3
            at Config.max_conn, dd  100
            at Config.timeout,  dd  5000            ; 5 seconds
            at Config.port,     dw  8080
            at Config.pad,      dw  0
        iend

section .bss
    ; buffer สำหรับ serialized data
    serial_buf  resb 256
    ; config ที่ deserialized
    loaded_cfg  resb Config_size

section .text
    global _start

; ====================================================
; ฟังก์ชัน: Serialize Config ไปยัง byte buffer
; rdi = pointer ไปยัง Config source
; rsi = pointer ไปยัง buffer destination
; return: rax = จำนวน bytes ที่เขียน
; ====================================================
serialize_config:
    push rbp
    mov rbp, rsp
    push rcx
    
    ; Copy struct ทั้งหมดไปยัง buffer (memcpy)
    mov ecx, Config_size
    
.copy_loop:
    dec ecx
    movzx eax, byte [rdi + rcx]
    mov [rsi + rcx], al
    test ecx, ecx
    jnz .copy_loop
    
    mov eax, Config_size    ; คืนจำนวน bytes
    pop rcx
    pop rbp
    ret

; ====================================================
; ฟังก์ชัน: Deserialize byte buffer ไปยัง Config
; rdi = pointer ไปยัง buffer source
; rsi = pointer ไปยัง Config destination
; return: rax = 1 ถ้าสำเร็จ, 0 ถ้า invalid magic
; ====================================================
deserialize_config:
    push rbp
    mov rbp, rsp
    push rcx
    
    ; ตรวจสอบ magic number ก่อน
    mov eax, [rdi + Config.magic]
    cmp eax, MAGIC_NUMBER
    jne .invalid
    
    ; Copy buffer ไปยัง Config struct
    mov ecx, Config_size

.copy_loop:
    dec ecx
    movzx eax, byte [rdi + rcx]
    mov [rsi + rcx], al
    test ecx, ecx
    jnz .copy_loop
    
    mov eax, 1          ; success
    jmp .done

.invalid:
    xor eax, eax        ; failure

.done:
    pop rcx
    pop rbp
    ret

; ====================================================
; ฟังก์ชัน: ตรวจสอบความถูกต้องของ Config
; rdi = pointer ไปยัง Config
; return: rax = 1 ถ้า valid, 0 ถ้า invalid
; ====================================================
validate_config:
    ; ตรวจสอบ magic
    cmp dword [rdi + Config.magic], MAGIC_NUMBER
    jne .invalid
    
    ; ตรวจสอบ version
    cmp word [rdi + Config.version], CONFIG_VERSION
    jne .invalid
    
    ; ตรวจสอบ port (ต้องอยู่ระหว่าง 1-65535)
    movzx eax, word [rdi + Config.port]
    test eax, eax
    jz  .invalid        ; port = 0 ไม่ valid
    cmp eax, 65535
    jg  .invalid
    
    ; ตรวจสอบ max_conn (ต้องมากกว่า 0)
    cmp dword [rdi + Config.max_conn], 0
    jle .invalid
    
    mov eax, 1          ; valid
    ret

.invalid:
    xor eax, eax
    ret

_start:
    ; Serialize default_config → serial_buf
    lea rdi, [default_config]
    lea rsi, [serial_buf]
    call serialize_config
    ; rax = 20 (Config_size)
    
    ; Deserialize serial_buf → loaded_cfg
    lea rdi, [serial_buf]
    lea rsi, [loaded_cfg]
    call deserialize_config
    ; rax = 1 (success)
    
    ; Validate loaded config
    lea rdi, [loaded_cfg]
    call validate_config
    ; rax = 1 (valid)
    
    ; อ่านค่าจาก loaded_cfg
    mov eax, [loaded_cfg + Config.max_conn]     ; eax = 100
    movzx ecx, word [loaded_cfg + Config.port]  ; ecx = 8080
    
    mov eax, 60
    xor edi, edi
    syscall
```

---

## 16. ตัวอย่างสมบูรณ์: Student Management System

```nasm
; ไฟล์: student_mgmt.asm
; คอมไพล์: nasm -f elf64 student_mgmt.asm -o student_mgmt.o
;          ld student_mgmt.o -o student_mgmt
; โปรแกรมบริหารจัดการข้อมูลนักศึกษาโดยใช้ struct

; ====================================================
; Constants
; ====================================================
MAX_NAME_LEN    equ 32
MAX_COURSES     equ 5

; ====================================================
; Course structure
; ====================================================
struc Course
    .code:      resw 1          ; รหัสวิชา (2 bytes)
    .credit:    resb 1          ; หน่วยกิต (1 byte)
    .grade:     resb 1          ; เกรด A=4, B=3, C=2, D=1, F=0
endstruc
; Course_size = 4 bytes

; ====================================================
; Student structure (สมบูรณ์)
; ====================================================
struc StudentFull
    .id:        resd 1                      ; รหัสนักศึกษา 8 หลัก (4 bytes)
    .year:      resb 1                      ; ชั้นปี 1-4 (1 byte)
    .sex:       resb 1                      ; เพศ 0=ชาย 1=หญิง (1 byte)
    .n_courses: resb 1                      ; จำนวนวิชาที่ลงทะเบียน (1 byte)
    .pad:       resb 1                      ; padding
    .name:      resb MAX_NAME_LEN           ; ชื่อ (32 bytes)
    .courses:   resb (Course_size * MAX_COURSES)  ; วิชา (20 bytes)
endstruc
; StudentFull_size = 4+1+1+1+1+32+20 = 60 bytes

section .data
    ; ====================================================
    ; ข้อมูลนักศึกษา
    ; ====================================================
    stu1:
        istruc StudentFull
            at StudentFull.id,        dd  65010001
            at StudentFull.year,      db  2
            at StudentFull.sex,       db  0          ; ชาย
            at StudentFull.n_courses, db  3
            at StudentFull.pad,       db  0
            at StudentFull.name,      db  "Somchai Jaidee", 0
                                      times (MAX_NAME_LEN-15) db 0
            at StudentFull.courses,   ; Course 0: CS101, 3 credits, B
                                      dw 101, db 3, db 3,
                                      ; Course 1: CS102, 3 credits, A
                                      dw 102, db 3, db 4,
                                      ; Course 2: MATH101, 4 credits, C
                                      dw 201, db 4, db 2,
                                      ; padding courses
                                      times (Course_size * (MAX_COURSES-3)) db 0
        iend

    num_students    dd  1

section .bss
    gpa_result  resq 1      ; GPA ผล (เก็บเป็น int * 100)

section .text
    global _start

; ====================================================
; ฟังก์ชัน: คำนวณ GPA
; rdi = pointer ไปยัง StudentFull
; return: eax = GPA * 100 (เพื่อหลีกเลี่ยง float)
;         (เช่น GPA 3.50 → return 350)
; ====================================================
calculate_gpa:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    
    movzx ecx, byte [rdi + StudentFull.n_courses]
    test ecx, ecx
    jz  .zero_gpa
    
    xor ebx, ebx    ; total_grade_points = 0
    xor r12d, r12d  ; total_credits = 0
    xor r13d, r13d  ; i = 0

.loop:
    cmp r13d, ecx
    jge .calc
    
    ; เข้าถึง courses[i]
    mov rax, r13
    imul rax, Course_size
    lea rdx, [rdi + StudentFull.courses]
    
    ; grade_points = credit * grade
    movzx r8d, byte [rdx + rax + Course.credit]
    movzx r9d, byte [rdx + rax + Course.grade]
    
    imul r8d, r9d           ; grade_points = credit * grade
    add ebx, r8d            ; total_grade_points += grade_points
    
    movzx r8d, byte [rdx + rax + Course.credit]
    add r12d, r8d           ; total_credits += credit
    
    inc r13d
    jmp .loop

.calc:
    ; GPA = total_grade_points / total_credits
    ; คำนวณ GPA * 100 เพื่อความแม่นยำ
    imul ebx, 100           ; total_grade_points * 100
    xor edx, edx
    div r12d                ; eax = (total_grade_points * 100) / total_credits
    jmp .done

.zero_gpa:
    xor eax, eax

.done:
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

_start:
    ; คำนวณ GPA ของ stu1
    lea rdi, [stu1]
    call calculate_gpa
    ; stu1 มี 3 วิชา: CS101(3cr,B=3), CS102(3cr,A=4), MATH101(4cr,C=2)
    ; total_gp = 3*3 + 3*4 + 4*2 = 9 + 12 + 8 = 29
    ; total_cr = 3 + 3 + 4 = 10
    ; GPA = 29/10 = 2.9 → return 290
    
    mov [gpa_result], eax
    
    mov eax, 60
    xor edi, edi
    syscall
```

---

## 17. สรุปคำสั่ง (Quick Reference)

### Struct Definition
```nasm
struc MyStruct
    .field1: resb 1     ; 1 byte
    .field2: resw 1     ; 2 bytes (word)
    .field3: resd 1     ; 4 bytes (dword)
    .field4: resq 1     ; 8 bytes (qword)
    .array:  resb 16    ; 16 bytes array
endstruc
; MyStruct_size ถูกสร้างขึ้นอัตโนมัติ
```

### Creating Instance
```nasm
; ใน .data section
myvar:
    istruc MyStruct
        at MyStruct.field1, db 10
        at MyStruct.field2, dw 200
        at MyStruct.field3, dd 3000
        at MyStruct.field4, dq 40000
        at MyStruct.array,  times 16 db 0
    iend

; ใน .bss section
mybuf: resb MyStruct_size
```

### Accessing Members
```nasm
; Direct access
mov eax, [myvar + MyStruct.field3]

; Via pointer (rdi = pointer)
mov eax, [rdi + MyStruct.field3]

; Array access (rdx = array base, rcx = index)
mov rax, rcx
imul rax, MyStruct_size
mov ebx, [rdx + rax + MyStruct.field1]
```

---

## 18. แบบฝึกหัด (Exercises)

### Exercise 1: Stack Structure
สร้าง Stack โดยใช้ STRUC:
```
- กำหนด Stack struct ที่มี: capacity, top, data array
- เขียนฟังก์ชัน push, pop, peek, is_empty, is_full
- ทดสอบด้วย values: 5, 10, 15, 20
```

### Exercise 2: Matrix Structure
```
- สร้าง Matrix 3x3 struct
- เขียนฟังก์ชัน matrix_add, matrix_multiply
- ทดสอบการคูณ identity matrix
```

### Exercise 3: File Header
```
- สร้าง BMP_Header struct ตาม BMP file format
- เขียนฟังก์ชัน validate_bmp ที่ตรวจ magic bytes "BM"
- เขียนฟังก์ชัน get_image_size ที่คืน width * height
```

### Exercise 4: Priority Queue
```
- สร้าง PriorityNode struct ที่มี data และ priority
- สร้าง PriorityQueue ด้วย max-heap ใช้ array
- เขียนฟังก์ชัน enqueue, dequeue ที่ถูกต้อง
```

### Exercise 5: JSON-like Structure
```
- สร้าง JSONValue struct ที่มี type (int/string/bool/null) และ union value
- เขียน serialize ไปยัง text format
- ทดสอบกับ object: {"name": "test", "value": 42, "active": true}
```

---

## 19. Tips และ Best Practices

### 1. เรียงฟิลด์จากใหญ่ไปเล็กเพื่อ Alignment
```nasm
; แบบดี - ไม่มี padding ที่ไม่จำเป็น
struc GoodLayout
    .q: resq 1      ; 8 bytes ก่อน
    .d: resd 1      ; 4 bytes ถัดมา
    .w: resw 1      ; 2 bytes
    .b: resb 1      ; 1 byte
    .p: resb 1      ; 1 byte padding เพื่อให้ size เป็น 16
endstruc

; แบบไม่ดี - มี padding เยอะ
struc BadLayout
    .b: resb 1      ; 1 byte แต่ต้องมี 7 byte padding ก่อน .q
    .q: resq 1      ; กระทบ alignment
    .w: resw 1
    .d: resd 1
endstruc
```

### 2. ใช้ Constants สำหรับ Magic Numbers
```nasm
PACKET_MAGIC    equ 0xCAFEBABE
CONFIG_VERSION  equ 2

; ตรวจสอบ
cmp dword [rdi + Packet.magic], PACKET_MAGIC
jne .invalid_packet
```

### 3. ใช้ Macros สำหรับ Common Operations
```nasm
; Macro สำหรับ copy struct
%macro COPY_STRUCT 3        ; %1=dst, %2=src, %3=struct_name
    push rsi
    push rdi
    push rcx
    lea rdi, [%1]
    lea rsi, [%2]
    mov ecx, %3_size
    rep movsb
    pop rcx
    pop rdi
    pop rsi
%endmacro
```

### 4. Document ขนาด Struct ในโค้ด
```nasm
struc Config
    .a: resb 1      ; offset 0, size 1
    .b: resb 3      ; offset 1, size 3 (padding)
    .c: resd 1      ; offset 4, size 4
    .d: resq 1      ; offset 8, size 8
endstruc
; Total: 16 bytes, aligned to 8
```

---

## 20. คำสั่ง Compilation

```bash
# ELF64 (Linux x86-64)
nasm -f elf64 program.asm -o program.o
ld program.o -o program

# ELF64 กับ C standard library
nasm -f elf64 program.asm -o program.o
gcc program.o -o program -no-pie

# Debug build
nasm -f elf64 -g -F dwarf program.asm -o program.o
ld program.o -o program

# ดู struct sizes ด้วย nasm preprocessing
nasm -f elf64 -e program.asm | grep "size"

# Disassemble เพื่อตรวจสอบ
objdump -d program
```

---

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:

1. **STRUC/ENDSTRUC** - การกำหนด structure ใน NASM
2. **ISTRUC/IEND** - การสร้าง initialized instances
3. **Member Access** - การเข้าถึง fields ด้วย base pointer + offset
4. **Nested Structs** - การซ้อน structure ภายใน structure
5. **Alignment** - การจัดเรียงฟิลด์ให้ถูกต้องเพื่อประสิทธิภาพ
6. **Packed Structs** - ใช้ทุก bit อย่างมีประสิทธิภาพ
7. **Arrays of Structs** - การคำนวณ offset ด้วย `index * struct_size`
8. **Struct Pointers** - การส่ง pointer ไปยังฟังก์ชัน
9. **Linked List** - การสร้าง linked list ด้วย STRUC
10. **Binary Tree** - BST operations ด้วย recursive functions
11. **Union** - การจำลอง union ด้วย shared memory
12. **Dynamic Allocation** - การใช้ mmap สำหรับ dynamic structs
13. **Serialization** - การแปลง struct ไป/กลับ byte stream

---

**ต่อไป**: Part 030 - Macros ขั้นสูงและ Conditional Assembly
