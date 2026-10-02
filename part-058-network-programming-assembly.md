# Part 058: Network Programming ใน Assembly

## บทนำ (Introduction)

Network programming ใน assembly เป็นหัวข้อที่ท้าทายแต่ให้ความเข้าใจลึกซึ้งที่สุดเกี่ยวกับการทำงานของเครือข่าย เราจะเรียกใช้ Linux system calls โดยตรง โดยไม่ผ่าน C library wrapper ใดๆ

ใน Linux x86-64, network system calls ทำงานผ่าน `syscall` instruction โดยใช้ register convention:
- `rax` = syscall number
- `rdi` = argument 1
- `rsi` = argument 2  
- `rdx` = argument 3
- `r10` = argument 4
- `r8`  = argument 5
- `r9`  = argument 6

---

## 1. Socket Syscall และ Address Families

### 1.1 syscall numbers สำหรับ network

```nasm
; Linux x86-64 network syscall numbers
SYS_SOCKET      equ 41      ; socket(domain, type, protocol)
SYS_BIND        equ 49      ; bind(sockfd, addr, addrlen)
SYS_LISTEN      equ 50      ; listen(sockfd, backlog)
SYS_ACCEPT      equ 43      ; accept(sockfd, addr, addrlen)
SYS_ACCEPT4     equ 288     ; accept4(sockfd, addr, addrlen, flags)
SYS_CONNECT     equ 42      ; connect(sockfd, addr, addrlen)
SYS_SEND        equ 44      ; send(sockfd, buf, len, flags) - alias sendto
SYS_RECV        equ 45      ; recv(sockfd, buf, len, flags) - alias recvfrom
SYS_SENDTO      equ 44      ; sendto(sockfd, buf, len, flags, dest_addr, addrlen)
SYS_RECVFROM    equ 45      ; recvfrom(sockfd, buf, len, flags, src_addr, addrlen)
SYS_SETSOCKOPT  equ 54      ; setsockopt(sockfd, level, optname, optval, optlen)
SYS_GETSOCKOPT  equ 55      ; getsockopt(sockfd, level, optname, optval, optlen)
SYS_GETSOCKNAME equ 51      ; getsockname(sockfd, addr, addrlen)
SYS_GETPEERNAME equ 52      ; getpeername(sockfd, addr, addrlen)
SYS_SHUTDOWN    equ 48      ; shutdown(sockfd, how)
SYS_CLOSE       equ 3       ; close(fd)
SYS_FCNTL       equ 72      ; fcntl(fd, cmd, arg)
SYS_SELECT      equ 23      ; select(nfds, readfds, writefds, exceptfds, timeout)
SYS_POLL        equ 7       ; poll(fds, nfds, timeout)
SYS_EPOLL_CREATE1 equ 291   ; epoll_create1(flags)
SYS_EPOLL_CTL   equ 233     ; epoll_ctl(epfd, op, fd, event)
SYS_EPOLL_WAIT  equ 232     ; epoll_wait(epfd, events, maxevents, timeout)
SYS_READ        equ 0
SYS_WRITE       equ 1
SYS_EXIT        equ 60
```

### 1.2 Address Families (AF_*)

```nasm
; Address Family constants
AF_UNSPEC   equ 0       ; Unspecified
AF_UNIX     equ 1       ; Unix domain sockets
AF_INET     equ 2       ; IPv4 Internet protocols
AF_INET6    equ 10      ; IPv6 Internet protocols
AF_PACKET   equ 17      ; Low level packet interface (raw sockets)
```

### 1.3 Socket Types (SOCK_*)

```nasm
; Socket type constants
SOCK_STREAM     equ 1   ; Sequenced, reliable, two-way connection (TCP)
SOCK_DGRAM      equ 2   ; Connectionless, unreliable (UDP)
SOCK_RAW        equ 3   ; Raw network protocol access
SOCK_NONBLOCK   equ 2048    ; O_NONBLOCK flag (0x800)
SOCK_CLOEXEC    equ 524288  ; O_CLOEXEC flag (0x80000)
```

### 1.4 Protocol constants

```nasm
; Protocol numbers (from /etc/protocols)
IPPROTO_IP      equ 0   ; Dummy protocol for TCP
IPPROTO_TCP     equ 6   ; Transmission Control Protocol
IPPROTO_UDP     equ 17  ; User Datagram Protocol
IPPROTO_RAW     equ 255 ; Raw IP packets
```

---

## 2. sockaddr_in Structure Layout

### 2.1 IPv4 sockaddr_in

```
struct sockaddr_in {
    sa_family_t    sin_family;   // 2 bytes: AF_INET = 2
    in_port_t      sin_port;     // 2 bytes: port in network byte order
    struct in_addr sin_addr;     // 4 bytes: IPv4 address
    char           sin_zero[8];  // 8 bytes: padding (must be zero)
};
// Total: 16 bytes
```

```nasm
; sockaddr_in structure offsets
struc sockaddr_in
    .sin_family:    resw 1  ; offset 0, 2 bytes
    .sin_port:      resw 1  ; offset 2, 2 bytes  
    .sin_addr:      resd 1  ; offset 4, 4 bytes
    .sin_zero:      resb 8  ; offset 8, 8 bytes padding
endstruc
; SOCKADDR_IN_SIZE = 16

; INADDR constants
INADDR_ANY       equ 0x00000000  ; 0.0.0.0 - bind to all interfaces
INADDR_LOOPBACK  equ 0x7f000001  ; 127.0.0.1 (network byte order)
INADDR_BROADCAST equ 0xFFFFFFFF  ; 255.255.255.255
```

### 2.2 IPv6 sockaddr_in6

```
struct sockaddr_in6 {
    sa_family_t     sin6_family;   // 2 bytes: AF_INET6 = 10
    in_port_t       sin6_port;     // 2 bytes: port (network byte order)
    uint32_t        sin6_flowinfo; // 4 bytes: IPv6 flow info
    struct in6_addr sin6_addr;     // 16 bytes: IPv6 address
    uint32_t        sin6_scope_id; // 4 bytes: scope ID
};
// Total: 28 bytes
```

```nasm
struc sockaddr_in6
    .sin6_family:   resw 1  ; offset 0
    .sin6_port:     resw 1  ; offset 2
    .sin6_flowinfo: resd 1  ; offset 4
    .sin6_addr:     resb 16 ; offset 8  (16 bytes for IPv6 address)
    .sin6_scope_id: resd 1  ; offset 24
endstruc
; SOCKADDR_IN6_SIZE = 28
```

---

## 3. htons/htonl ใน Assembly (Byte Order Conversion)

### 3.1 ความเข้าใจ Byte Order

Network byte order = Big-endian (MSB first)
x86 = Little-endian (LSB first)

ตัวอย่าง port 8080 = 0x1F90:
- Little-endian (x86): 0x90 0x1F ในหน่วยความจำ
- Big-endian (network): 0x1F 0x90 ในหน่วยความจำ

### 3.2 htons (host to network short - 16-bit)

```nasm
; htons ใน assembly - แปลง 16-bit จาก host order เป็น network order
; Input: ax = value in host byte order
; Output: ax = value in network byte order

htons_macro:
    ; วิธีที่ 1: ใช้ XCHG
    xchg    al, ah          ; สลับ byte สูงและต่ำ
    
    ; วิธีที่ 2: ใน 32-bit register ใช้ ROR
    ror     ax, 8           ; หมุนขวา 8 bits
    
    ; วิธีที่ 3: คำนวณ manual
    movzx   eax, ax         ; zero extend
    movzx   ecx, al         ; เก็บ low byte
    shr     eax, 8          ; เลื่อน high byte มา low
    shl     ecx, 8          ; เลื่อน low byte ไป high
    or      eax, ecx        ; รวมกัน

; ตัวอย่างการใช้งาน: port 8080 = 0x1F90
; mov ax, 8080
; xchg al, ah  ; ax = 0x901F = 0x1F90 ใน little-endian memory
```

### 3.3 htonl (host to network long - 32-bit)

```nasm
; htonl ใน assembly - แปลง 32-bit จาก host order เป็น network order
; Input: eax = value in host byte order
; Output: eax = value in network byte order

htonl_func:
    ; วิธีที่ 1: ใช้ BSWAP (เร็วที่สุด)
    bswap   eax             ; สลับทุก byte ใน 32-bit register
    ret
    ; BSWAP ab cd ef 12 -> 12 ef cd ab
    
    ; วิธีที่ 2: ใช้ ROR
    ror     eax, 16         ; สลับ 2 words
    xchg    al, ah          ; สลับ bytes ใน low word
    ror     eax, 16
    xchg    al, ah
    
    ; วิธีที่ 3: ใช้ PSHUFB (SSE3 - เร็วมากสำหรับ bulk)
    ; movd xmm0, eax
    ; pshufb xmm0, [bswap_mask]  ; mask = [3,2,1,0, ...]
    ; movd eax, xmm0

; ตัวอย่าง:
; mov eax, 0x01020304    ; host order
; bswap eax              ; eax = 0x04030201 (network order)

; สำหรับ IP address 127.0.0.1 = 0x7F000001
; ใน network byte order (big-endian): 0x7F 0x00 0x00 0x01
; ใน little-endian memory: 01 00 00 7F
; htonl(0x0100007F) = 0x7F000001 (เก็บในหน่วยความจำ = 01 00 00 7F)
```

### 3.4 ntohl/ntohs (network to host)

```nasm
; ntohl และ ntohs เหมือน htonl/htons เพราะการ swap สองทิศทางเหมือนกัน
; ntohl = htonl, ntohs = htons (symmetric operation)

ntohl_func:
    bswap   eax     ; เหมือนกันเลย
    ret

ntohs_func:
    xchg    al, ah  ; เหมือนกันเลย
    ret
```

---

## 4. bind Syscall

```nasm
; bind(sockfd, addr, addrlen)
; rdi = sockfd
; rsi = pointer to sockaddr structure
; rdx = size of sockaddr structure
; Returns: 0 on success, -1 on error (errno set)

; ตัวอย่างการ bind ที่ port 8080 บน 0.0.0.0
section .bss
    server_addr resb 16     ; sockaddr_in structure (16 bytes)

section .text
bind_example:
    ; สร้าง sockaddr_in structure
    lea     rdi, [server_addr]
    
    ; sin_family = AF_INET (2)
    mov     word [rdi], AF_INET
    
    ; sin_port = htons(8080) = 0x901F
    mov     ax, 8080
    xchg    al, ah          ; byte swap
    mov     word [rdi + 2], ax
    
    ; sin_addr = INADDR_ANY (0)
    mov     dword [rdi + 4], 0
    
    ; sin_zero = 0 (8 bytes padding)
    mov     qword [rdi + 8], 0
    
    ; เรียก bind syscall
    ; rdi ยังชี้ไปที่ server_addr อยู่
    mov     rsi, rdi        ; rsi = addr
    mov     rdi, [sockfd]   ; rdi = sockfd
    mov     rdx, 16         ; rdx = sizeof(sockaddr_in)
    mov     rax, SYS_BIND
    syscall
    
    test    rax, rax
    js      bind_error      ; jump if sign (negative = error)
    ret

bind_error:
    ; rax contains -errno
    neg     rax             ; แปลงเป็น positive errno
    ret
```

---

## 5. listen Syscall

```nasm
; listen(sockfd, backlog)
; rdi = sockfd
; rsi = backlog (จำนวน connections ที่ queue ได้)
; Returns: 0 on success, -1 on error

LISTEN_BACKLOG  equ 128     ; จำนวน pending connections ที่ยอมรับได้

listen_example:
    mov     rdi, [sockfd]
    mov     rsi, LISTEN_BACKLOG
    mov     rax, SYS_LISTEN
    syscall
    
    test    rax, rax
    js      listen_error
    ret
```

---

## 6. accept/accept4 Syscall

```nasm
; accept(sockfd, addr, addrlen)
; rdi = sockfd (listening socket)
; rsi = pointer to sockaddr (client address, can be NULL)
; rdx = pointer to addrlen (size of addr buffer, updated with actual size)
; Returns: new fd on success, -1 on error

; accept4(sockfd, addr, addrlen, flags)
; r10 = flags (SOCK_NONBLOCK, SOCK_CLOEXEC, etc.)

section .bss
    client_addr     resb 16     ; client sockaddr_in
    client_addrlen  resd 1      ; client address length

accept_example:
    ; ตั้งค่า addrlen ก่อนเรียก accept
    mov     dword [client_addrlen], 16
    
    mov     rdi, [server_sockfd]
    lea     rsi, [client_addr]
    lea     rdx, [client_addrlen]
    mov     rax, SYS_ACCEPT
    syscall
    
    test    rax, rax
    js      accept_error
    mov     [client_sockfd], rax    ; เก็บ client fd
    ret

; accept4 - ดีกว่า accept เพราะสามารถตั้ง flags ได้ทันที
accept4_example:
    mov     dword [client_addrlen], 16
    
    mov     rdi, [server_sockfd]
    lea     rsi, [client_addr]
    lea     rdx, [client_addrlen]
    mov     r10, SOCK_CLOEXEC       ; ตั้ง CLOEXEC flag
    mov     rax, SYS_ACCEPT4
    syscall
    ret
```

---

## 7. connect Syscall

```nasm
; connect(sockfd, addr, addrlen)
; rdi = sockfd
; rsi = pointer to server sockaddr
; rdx = size of sockaddr

connect_to_server:
    ; สร้าง server address structure
    lea     rbx, [server_addr]
    
    ; AF_INET
    mov     word [rbx], AF_INET
    
    ; port 80 -> htons(80) = 0x5000
    mov     ax, 80
    xchg    al, ah
    mov     word [rbx + 2], ax
    
    ; 127.0.0.1 = 0x7F000001 ใน network byte order
    ; ใน little-endian memory: 01 00 00 7F
    mov     dword [rbx + 4], 0x0100007F
    
    ; zero padding
    mov     qword [rbx + 8], 0
    
    ; เรียก connect
    mov     rdi, [client_sockfd]
    mov     rsi, rbx
    mov     rdx, 16
    mov     rax, SYS_CONNECT
    syscall
    
    test    rax, rax
    js      connect_error
    ret
```

---

## 8. send/recv Syscalls

### 8.1 send

```nasm
; send(sockfd, buf, len, flags)
; rdi = sockfd
; rsi = buffer pointer
; rdx = buffer length
; r10 = flags (0 = no flags, MSG_DONTWAIT = 0x40, MSG_NOSIGNAL = 0x4000)

MSG_DONTWAIT    equ 0x40
MSG_NOSIGNAL    equ 0x4000
MSG_WAITALL     equ 0x100

send_data:
    mov     rdi, [client_sockfd]
    mov     rsi, [data_ptr]         ; pointer to data
    mov     rdx, [data_len]         ; length of data
    mov     r10, MSG_NOSIGNAL       ; ไม่ให้ SIGPIPE ถ้า connection ถูกปิด
    mov     rax, SYS_SEND
    syscall
    ; rax = bytes sent, หรือ -1 ถ้า error

send_all:
    ; วนลูปส่งข้อมูลจนครบ
    xor     r12, r12        ; bytes sent so far = 0
.loop:
    mov     rdi, [sockfd]
    lea     rsi, [buffer + r12]     ; pointer + offset
    mov     rdx, [total_len]
    sub     rdx, r12                ; remaining bytes
    jz      .done                   ; ส่งครบแล้ว
    
    mov     r10, MSG_NOSIGNAL
    mov     rax, SYS_SEND
    syscall
    
    test    rax, rax
    js      .error          ; error
    jz      .peer_closed    ; connection closed
    
    add     r12, rax        ; เพิ่ม bytes ที่ส่งแล้ว
    jmp     .loop
.done:
    mov     rax, r12        ; return total bytes sent
    ret
.error:
.peer_closed:
    ret
```

### 8.2 recv

```nasm
; recv(sockfd, buf, len, flags)
; rdi = sockfd
; rsi = buffer
; rdx = buffer length
; r10 = flags

recv_data:
    mov     rdi, [client_sockfd]
    lea     rsi, [recv_buffer]
    mov     rdx, RECV_BUFFER_SIZE
    mov     r10, 0          ; no flags
    mov     rax, SYS_RECV
    syscall
    ; rax = bytes received, 0 = connection closed, -1 = error
    
    cmp     rax, 0
    je      .connection_closed
    js      .error
    ; rax = number of bytes received
    ret
```

---

## 9. sendto/recvfrom สำหรับ UDP

### 9.1 sendto

```nasm
; sendto(sockfd, buf, len, flags, dest_addr, addrlen)
; rdi = sockfd
; rsi = buf
; rdx = len
; r10 = flags
; r8  = dest_addr (pointer to sockaddr)
; r9  = addrlen

udp_send:
    mov     rdi, [udp_sockfd]
    lea     rsi, [send_buffer]
    mov     rdx, [send_len]
    mov     r10, 0                      ; no flags
    lea     r8,  [dest_addr]            ; destination address
    mov     r9,  16                     ; sizeof(sockaddr_in)
    mov     rax, SYS_SENDTO
    syscall
    ret
```

### 9.2 recvfrom

```nasm
; recvfrom(sockfd, buf, len, flags, src_addr, addrlen)
; rdi = sockfd
; rsi = buf
; rdx = len
; r10 = flags
; r8  = src_addr (filled in by kernel)
; r9  = addrlen (pointer to int, must be initialized)

section .bss
    udp_from_addr    resb 16
    udp_from_addrlen resd 1

udp_recv:
    mov     dword [udp_from_addrlen], 16
    
    mov     rdi, [udp_sockfd]
    lea     rsi, [recv_buffer]
    mov     rdx, BUFFER_SIZE
    mov     r10, 0
    lea     r8,  [udp_from_addr]
    lea     r9,  [udp_from_addrlen]
    mov     rax, SYS_RECVFROM
    syscall
    ; rax = bytes received
    ; udp_from_addr = sender's address
    ret
```

---

## 10. setsockopt

### 10.1 Socket Option Constants

```nasm
; setsockopt(sockfd, level, optname, optval, optlen)
; rdi = sockfd
; rsi = level
; rdx = optname
; r10 = optval (pointer to value)
; r8  = optlen (size of optval)

; Level constants
SOL_SOCKET      equ 1       ; Socket-level options
IPPROTO_TCP     equ 6       ; TCP-level options

; SOL_SOCKET option names
SO_REUSEADDR    equ 2       ; Allow reuse of local addresses
SO_REUSEPORT    equ 15      ; Allow reuse of port
SO_KEEPALIVE    equ 9       ; Keep connections alive
SO_SNDBUF       equ 7       ; Send buffer size
SO_RCVBUF       equ 8       ; Receive buffer size
SO_LINGER       equ 13      ; Linger on close
SO_RCVTIMEO     equ 20      ; Receive timeout
SO_SNDTIMEO     equ 21      ; Send timeout

; IPPROTO_TCP option names  
TCP_NODELAY     equ 1       ; Disable Nagle's algorithm
TCP_MAXSEG      equ 2       ; Maximum segment size
TCP_KEEPIDLE    equ 4       ; Idle time before keepalive probe
TCP_KEEPINTVL   equ 5       ; Interval between keepalive probes
TCP_KEEPCNT     equ 6       ; Number of keepalive probes
```

### 10.2 SO_REUSEADDR

```nasm
; ตั้ง SO_REUSEADDR เพื่อให้ bind ใหม่ได้หลัง server restart
section .data
    opt_val_1   dd 1    ; integer value = 1 (enable)

set_reuseaddr:
    mov     rdi, [sockfd]
    mov     rsi, SOL_SOCKET     ; level = SOL_SOCKET
    mov     rdx, SO_REUSEADDR   ; optname
    lea     r10, [opt_val_1]    ; optval = pointer to 1
    mov     r8,  4              ; optlen = sizeof(int)
    mov     rax, SYS_SETSOCKOPT
    syscall
    test    rax, rax
    js      .error
    ret
```

### 10.3 SO_REUSEPORT

```nasm
; SO_REUSEPORT ทำให้หลาย processes bind ที่ port เดียวกันได้ (load balancing)
set_reuseport:
    mov     rdi, [sockfd]
    mov     rsi, SOL_SOCKET
    mov     rdx, SO_REUSEPORT
    lea     r10, [opt_val_1]
    mov     r8,  4
    mov     rax, SYS_SETSOCKOPT
    syscall
    ret
```

### 10.4 TCP_NODELAY

```nasm
; TCP_NODELAY ปิด Nagle's algorithm - ส่ง data ทันที ไม่รอ buffer เต็ม
; เหมาะสำหรับ real-time applications (gaming, financial trading)
set_tcp_nodelay:
    mov     rdi, [sockfd]
    mov     rsi, IPPROTO_TCP    ; level = IPPROTO_TCP
    mov     rdx, TCP_NODELAY    ; optname
    lea     r10, [opt_val_1]
    mov     r8,  4
    mov     rax, SYS_SETSOCKOPT
    syscall
    ret
```

---

## 11. getsockname/getpeername

```nasm
; getsockname - ดึง local address ของ socket
; getpeername - ดึง remote address ของ connected socket

; getsockname(sockfd, addr, addrlen)
section .bss
    my_addr     resb 16
    my_addrlen  resd 1

get_local_addr:
    mov     dword [my_addrlen], 16
    
    mov     rdi, [sockfd]
    lea     rsi, [my_addr]
    lea     rdx, [my_addrlen]
    mov     rax, SYS_GETSOCKNAME
    syscall
    ; my_addr ถูก fill ด้วย local address
    ret

; getpeername(sockfd, addr, addrlen)
section .bss
    peer_addr       resb 16
    peer_addrlen    resd 1

get_peer_addr:
    mov     dword [peer_addrlen], 16
    
    mov     rdi, [client_sockfd]
    lea     rsi, [peer_addr]
    lea     rdx, [peer_addrlen]
    mov     rax, SYS_GETPEERNAME
    syscall
    ret

; ดึง IP และ port จาก sockaddr_in
extract_ip_port:
    ; peer_addr อยู่ใน rbx
    lea     rbx, [peer_addr]
    
    ; ดึง port (network byte order -> host byte order)
    movzx   eax, word [rbx + 2]  ; sin_port
    xchg    al, ah               ; ntohs
    ; eax = port number
    
    ; ดึง IP address (network byte order)
    mov     eax, dword [rbx + 4]  ; sin_addr
    bswap   eax                   ; ntohl
    ; eax = IP address ใน host byte order (192.168.1.1 = 0xC0A80101)
    ret
```

---

## 12. shutdown Syscall

```nasm
; shutdown(sockfd, how)
; rdi = sockfd
; rsi = how: SHUT_RD=0, SHUT_WR=1, SHUT_RDWR=2

SHUT_RD     equ 0   ; ปิดรับข้อมูล
SHUT_WR     equ 1   ; ปิดส่งข้อมูล (ส่ง FIN)
SHUT_RDWR   equ 2   ; ปิดทั้งสองทาง

graceful_close:
    ; ส่ง FIN แต่ยังรับข้อมูลได้
    mov     rdi, [sockfd]
    mov     rsi, SHUT_WR
    mov     rax, SYS_SHUTDOWN
    syscall
    
    ; รอรับข้อมูลที่เหลือ (drain)
.drain_loop:
    mov     rdi, [sockfd]
    lea     rsi, [drain_buf]
    mov     rdx, 1024
    mov     r10, 0
    mov     rax, SYS_RECV
    syscall
    test    rax, rax
    jg      .drain_loop     ; ยังมีข้อมูล, รับต่อ
    
    ; ปิด socket
    mov     rdi, [sockfd]
    mov     rax, SYS_CLOSE
    syscall
    ret
```

---

## 13. Non-blocking: fcntl(O_NONBLOCK)

```nasm
; fcntl(fd, cmd, arg)
; rdi = fd
; rsi = cmd
; rdx = arg

; fcntl commands
F_GETFL     equ 3   ; Get file status flags
F_SETFL     equ 4   ; Set file status flags
O_NONBLOCK  equ 2048    ; Non-blocking I/O (0x800)

set_nonblocking:
    ; ขั้นตอนที่ 1: ดึง current flags ก่อน
    mov     rdi, [sockfd]
    mov     rsi, F_GETFL
    xor     rdx, rdx        ; arg ไม่ใช้
    mov     rax, SYS_FCNTL
    syscall
    
    test    rax, rax
    js      .error
    
    ; ขั้นตอนที่ 2: เพิ่ม O_NONBLOCK flag
    mov     rdx, rax
    or      rdx, O_NONBLOCK
    
    mov     rdi, [sockfd]
    mov     rsi, F_SETFL
    ; rdx = flags | O_NONBLOCK
    mov     rax, SYS_FCNTL
    syscall
    
    test    rax, rax
    js      .error
    ret

.error:
    ret

; เมื่อ socket เป็น non-blocking:
; - accept() จะ return -EAGAIN ถ้าไม่มี connection
; - recv() จะ return -EAGAIN ถ้าไม่มีข้อมูล
; - send() จะ return -EAGAIN ถ้า buffer เต็ม
EAGAIN  equ 11  ; errno: Try again
```

---

## 14. select Syscall: fd_set Manipulation

### 14.1 fd_set Structure

```nasm
; fd_set = bitmap ของ file descriptors
; FD_SETSIZE = 1024 bits = 128 bytes
FD_SETSIZE  equ 1024

; fd_set operations ใน assembly:
; FD_ZERO(set)  - clear all bits
; FD_SET(fd, set) - set bit fd
; FD_CLR(fd, set) - clear bit fd
; FD_ISSET(fd, set) - test bit fd

; fd_set เป็นแค่ array of unsigned long (8 bytes each)
; สำหรับ FD_SETSIZE=1024: 1024/64 = 16 longs = 128 bytes

section .bss
    read_fds    resb 128    ; fd_set สำหรับ read
    write_fds   resb 128    ; fd_set สำหรับ write  
    except_fds  resb 128    ; fd_set สำหรับ exceptions

; FD_ZERO macro
%macro FD_ZERO 1
    ; %1 = pointer to fd_set
    push    rdi
    push    rcx
    lea     rdi, [%1]
    mov     rcx, 16         ; 128 bytes / 8 bytes per qword = 16
    xor     eax, eax
    rep stosq               ; fill with zeros
    pop     rcx
    pop     rdi
%endmacro

; FD_SET macro
%macro FD_SET 2
    ; %1 = fd, %2 = pointer to fd_set
    push    rax
    push    rcx
    push    rdx
    mov     eax, %1
    mov     ecx, eax
    shr     ecx, 6          ; index = fd / 64 (offset ใน array)
    and     eax, 63         ; bit = fd % 64
    bts     qword [%2 + rcx*8], rax  ; set bit
    pop     rdx
    pop     rcx
    pop     rax
%endmacro

; FD_ISSET macro - sets ZF if not set, clears ZF if set
%macro FD_ISSET 2
    ; %1 = fd, %2 = pointer to fd_set
    push    rcx
    mov     eax, %1
    mov     ecx, eax
    shr     ecx, 6
    and     eax, 63
    bt      qword [%2 + rcx*8], rax  ; test bit -> CF
    pop     rcx
    ; CF = 1 ถ้า fd อยู่ใน set
%endmacro
```

### 14.2 select syscall

```nasm
; select(nfds, readfds, writefds, exceptfds, timeout)
; rdi = nfds (max fd + 1)
; rsi = readfds pointer (or NULL)
; rdx = writefds pointer (or NULL)
; r10 = exceptfds pointer (or NULL)
; r8  = timeout pointer (or NULL for infinite)

struc timeval
    .tv_sec:    resq 1      ; seconds
    .tv_usec:   resq 1      ; microseconds
endstruc

section .bss
    select_timeout  resb 16     ; struct timeval

select_example:
    ; ตั้ง timeout = 5 วินาที
    mov     qword [select_timeout + timeval.tv_sec],  5
    mov     qword [select_timeout + timeval.tv_usec], 0
    
    ; Zero out read fd_set
    lea     rdi, [read_fds]
    mov     ecx, 16
    xor     eax, eax
    rep stosq
    
    ; FD_SET(server_fd, &read_fds)
    mov     eax, [server_fd]
    mov     ecx, eax
    shr     ecx, 6
    and     eax, 63
    bts     qword [read_fds + rcx*8], rax
    
    ; เรียก select
    mov     rdi, [server_fd]
    inc     rdi                 ; nfds = max_fd + 1
    lea     rsi, [read_fds]
    xor     rdx, rdx            ; writefds = NULL
    xor     r10, r10            ; exceptfds = NULL
    lea     r8,  [select_timeout]
    mov     rax, SYS_SELECT
    syscall
    
    ; rax = number of fds ready, 0 = timeout, -1 = error
    test    rax, rax
    js      .error
    jz      .timeout
    
    ; ตรวจสอบว่า server_fd พร้อมหรือไม่
    mov     eax, [server_fd]
    mov     ecx, eax
    shr     ecx, 6
    and     eax, 63
    bt      qword [read_fds + rcx*8], rax
    jnc     .not_ready          ; CF = 0 = not set
    
    ; server_fd พร้อม! รับ connection
    ; ... เรียก accept
    
.error:
.timeout:
.not_ready:
    ret
```

---

## 15. poll Syscall: pollfd Structure

### 15.1 pollfd Structure

```nasm
; struct pollfd {
;     int   fd;        // 4 bytes: file descriptor
;     short events;    // 2 bytes: events to watch
;     short revents;   // 2 bytes: events that occurred
; };

struc pollfd
    .fd:      resd 1    ; offset 0
    .events:  resw 1    ; offset 4
    .revents: resw 1    ; offset 6
endstruc
; POLLFD_SIZE = 8

; Event flags
POLLIN      equ 0x0001  ; Data may be read
POLLOUT     equ 0x0004  ; Writing is possible
POLLERR     equ 0x0008  ; Error condition (output only)
POLLHUP     equ 0x0010  ; Hang up (output only)
POLLNVAL    equ 0x0020  ; Invalid request: fd not open
POLLRDHUP   equ 0x2000  ; Peer closed/shutdown writing
```

### 15.2 poll syscall

```nasm
; poll(fds, nfds, timeout)
; rdi = pointer to array of pollfd
; rsi = number of pollfd structures
; rdx = timeout in milliseconds (-1 = infinite)

MAX_FDS     equ 10

section .bss
    poll_fds    resb MAX_FDS * 8    ; array of pollfd

poll_example:
    ; ตั้งค่า pollfd array
    lea     rbx, [poll_fds]
    
    ; pollfd[0]: server socket, รอรับ connections
    mov     eax, [server_fd]
    mov     dword [rbx + pollfd.fd],     eax
    mov     word  [rbx + pollfd.events], POLLIN
    mov     word  [rbx + pollfd.revents], 0
    
    ; เรียก poll
    lea     rdi, [poll_fds]
    mov     rsi, 1          ; nfds = 1
    mov     rdx, 5000       ; timeout = 5000ms = 5 seconds
    mov     rax, SYS_POLL
    syscall
    
    ; rax = number of fds ready, 0 = timeout, -1 = error
    test    rax, rax
    js      .error
    jz      .timeout
    
    ; ตรวจสอบ revents
    lea     rbx, [poll_fds]
    movzx   eax, word [rbx + pollfd.revents]
    test    eax, POLLIN
    jz      .no_data
    
    ; มีข้อมูล! เรียก accept
    
.error:
.timeout:
.no_data:
    ret
```

---

## 16. epoll: epoll_create1/epoll_ctl/epoll_wait

### 16.1 epoll_event Structure

```nasm
; struct epoll_event {
;     uint32_t events;     // 4 bytes: event flags
;     epoll_data_t data;   // 8 bytes: user data union
; };
; Note: เนื่องจาก alignment, structure อาจเป็น 12 bytes
; แต่ใน x86-64 Linux kernel ใช้ packed = 12 bytes

struc epoll_event
    .events:  resd 1    ; offset 0: 4 bytes
    .data:    resq 1    ; offset 4: 8 bytes (union: fd, ptr, u32, u64)
endstruc
; EPOLL_EVENT_SIZE = 12 (packed)

; Epoll event flags
EPOLLIN     equ 0x001   ; Available for read
EPOLLOUT    equ 0x004   ; Available for write
EPOLLERR    equ 0x008   ; Error condition
EPOLLHUP    equ 0x010   ; Hang up
EPOLLET     equ 1 << 31 ; Edge-triggered (vs level-triggered)
EPOLLONESHOT equ 1 << 30 ; One-shot notification
EPOLLRDHUP  equ 0x2000  ; Peer closed connection

; epoll_ctl operations
EPOLL_CTL_ADD   equ 1   ; Add a fd to the interest list
EPOLL_CTL_DEL   equ 2   ; Remove a fd
EPOLL_CTL_MOD   equ 3   ; Change event mask

; epoll_create1 flags
EPOLL_CLOEXEC   equ 524288  ; O_CLOEXEC (0x80000)
```

### 16.2 epoll_create1

```nasm
; epoll_create1(flags)
; rdi = flags (0 or EPOLL_CLOEXEC)
; Returns: epfd (epoll file descriptor), -1 on error

create_epoll:
    mov     rdi, EPOLL_CLOEXEC  ; หรือ 0
    mov     rax, SYS_EPOLL_CREATE1
    syscall
    
    test    rax, rax
    js      .error
    mov     [epfd], rax     ; เก็บ epoll fd
    ret
```

### 16.3 epoll_ctl

```nasm
; epoll_ctl(epfd, op, fd, event)
; rdi = epfd
; rsi = op (ADD/DEL/MOD)
; rdx = fd to watch
; r10 = pointer to epoll_event

section .bss
    ep_event    resb 12     ; epoll_event (12 bytes packed)

add_to_epoll:
    ; ตั้งค่า event
    lea     rbx, [ep_event]
    mov     dword [rbx + epoll_event.events], EPOLLIN | EPOLLET  ; edge-triggered read
    mov     qword [rbx + epoll_event.data],   [server_fd]       ; user data = fd

    mov     rdi, [epfd]
    mov     rsi, EPOLL_CTL_ADD
    mov     rdx, [server_fd]
    lea     r10, [ep_event]
    mov     rax, SYS_EPOLL_CTL
    syscall
    ret

modify_epoll:
    ; เปลี่ยน events ที่ต้องการ monitor
    lea     rbx, [ep_event]
    mov     dword [rbx + epoll_event.events], EPOLLIN | EPOLLOUT
    mov     qword [rbx + epoll_event.data],   [client_fd]

    mov     rdi, [epfd]
    mov     rsi, EPOLL_CTL_MOD
    mov     rdx, [client_fd]
    lea     r10, [ep_event]
    mov     rax, SYS_EPOLL_CTL
    syscall
    ret

remove_from_epoll:
    mov     rdi, [epfd]
    mov     rsi, EPOLL_CTL_DEL
    mov     rdx, [client_fd]
    xor     r10, r10        ; event = NULL (ignored for DEL)
    mov     rax, SYS_EPOLL_CTL
    syscall
    ret
```

### 16.4 epoll_wait

```nasm
; epoll_wait(epfd, events, maxevents, timeout)
; rdi = epfd
; rsi = array of epoll_event to fill
; rdx = maxevents
; r10 = timeout (-1 = infinite, 0 = return immediately)

MAX_EVENTS  equ 64

section .bss
    events_buf  resb MAX_EVENTS * 12    ; array of epoll_event

epoll_event_loop:
.loop:
    mov     rdi, [epfd]
    lea     rsi, [events_buf]
    mov     rdx, MAX_EVENTS
    mov     r10, -1             ; timeout = infinite
    mov     rax, SYS_EPOLL_WAIT
    syscall
    
    ; rax = number of events, -1 on error
    test    rax, rax
    js      .error
    
    ; วนประมวลผลแต่ละ event
    mov     rcx, rax            ; event count
    xor     r12, r12            ; index = 0
    lea     r13, [events_buf]

.process_events:
    test    rcx, rcx
    jz      .loop               ; ประมวลผลครบ, รอ events ใหม่
    
    ; ดึง event
    mov     eax, dword [r13 + r12 + epoll_event.events]
    mov     rbx, qword [r13 + r12 + epoll_event.data]  ; fd
    
    ; ตรวจสอบว่าเป็น server socket หรือ client socket
    cmp     rbx, [server_fd]
    je      .new_connection
    
    ; client socket - ตรวจสอบ event type
    test    eax, EPOLLIN
    jnz     .read_data
    
    test    eax, EPOLLHUP
    jnz     .close_connection
    
    test    eax, EPOLLERR
    jnz     .close_connection
    
    jmp     .next_event

.new_connection:
    ; เรียก accept และเพิ่ม fd ใหม่ใน epoll
    ; ... accept4 ...
    jmp     .next_event

.read_data:
    ; รับข้อมูลจาก client
    ; rbx = client fd
    jmp     .next_event

.close_connection:
    ; ปิด connection
    ; rbx = fd to close
    jmp     .next_event

.next_event:
    add     r12, 12     ; sizeof(epoll_event) = 12
    dec     rcx
    jmp     .process_events

.error:
    ret
```

---

## 17. Program: TCP Echo Server

โปรแกรม TCP echo server ที่รับข้อมูลและส่งกลับ

```nasm
; tcp_echo_server.asm
; TCP Echo Server - รับข้อมูลและส่งกลับ
; ใช้งาน: ./tcp_echo_server (รอรับที่ port 12345)

global _start

section .data
    ; ข้อความแสดงสถานะ
    msg_start   db "TCP Echo Server เริ่มทำงาน...", 10, 0
    msg_start_len equ $ - msg_start
    msg_listen  db "กำลัง listen ที่ port 12345", 10, 0
    msg_listen_len equ $ - msg_listen
    msg_connect db "Client เชื่อมต่อแล้ว", 10, 0
    msg_connect_len equ $ - msg_connect
    msg_close   db "Client ตัดการเชื่อมต่อ", 10, 0
    msg_close_len equ $ - msg_close
    msg_error   db "ERROR เกิดขึ้น!", 10, 0
    msg_error_len equ $ - msg_error

; Syscall numbers
SYS_READ        equ 0
SYS_WRITE       equ 1
SYS_CLOSE       equ 3
SYS_EXIT        equ 60
SYS_SOCKET      equ 41
SYS_ACCEPT      equ 43
SYS_BIND        equ 49
SYS_LISTEN      equ 50
SYS_SETSOCKOPT  equ 54

; Socket constants
AF_INET         equ 2
SOCK_STREAM     equ 1
SOL_SOCKET      equ 1
SO_REUSEADDR    equ 2
IPPROTO_TCP     equ 6

STDOUT          equ 1
PORT            equ 12345
BACKLOG         equ 5
BUFFER_SIZE     equ 4096

section .bss
    server_fd   resq 1
    client_fd   resq 1
    server_addr resb 16
    client_addr resb 16
    client_addrlen resd 1
    buffer      resb BUFFER_SIZE
    opt_val     resd 1

section .text
_start:
    ; แสดงข้อความเริ่มต้น
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel msg_start]
    mov     rdx, msg_start_len
    syscall

    ;--- ขั้นตอนที่ 1: สร้าง socket ---
    mov     rdi, AF_INET        ; domain = IPv4
    mov     rsi, SOCK_STREAM    ; type = TCP
    xor     rdx, rdx            ; protocol = 0 (auto)
    mov     rax, SYS_SOCKET
    syscall
    
    test    rax, rax
    js      .fatal_error
    mov     [rel server_fd], rax

    ;--- ขั้นตอนที่ 2: ตั้ง SO_REUSEADDR ---
    mov     dword [rel opt_val], 1
    mov     rdi, [rel server_fd]
    mov     rsi, SOL_SOCKET
    mov     rdx, SO_REUSEADDR
    lea     r10, [rel opt_val]
    mov     r8,  4
    mov     rax, SYS_SETSOCKOPT
    syscall

    ;--- ขั้นตอนที่ 3: ตั้ง server address ---
    ; server_addr = { AF_INET, htons(12345), INADDR_ANY, 0 }
    lea     rbx, [rel server_addr]
    mov     word  [rbx],      AF_INET     ; sin_family
    
    ; htons(12345): 12345 = 0x3039, htons = 0x3930
    mov     ax, PORT
    xchg    al, ah
    mov     word  [rbx + 2],  ax          ; sin_port
    
    mov     dword [rbx + 4],  0           ; sin_addr = INADDR_ANY
    mov     qword [rbx + 8],  0           ; sin_zero

    ;--- ขั้นตอนที่ 4: bind ---
    mov     rdi, [rel server_fd]
    lea     rsi, [rel server_addr]
    mov     rdx, 16
    mov     rax, SYS_BIND
    syscall
    
    test    rax, rax
    js      .fatal_error

    ;--- ขั้นตอนที่ 5: listen ---
    mov     rdi, [rel server_fd]
    mov     rsi, BACKLOG
    mov     rax, SYS_LISTEN
    syscall
    
    test    rax, rax
    js      .fatal_error

    ; แสดงข้อความ listen
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel msg_listen]
    mov     rdx, msg_listen_len
    syscall

    ;--- Main Loop: รอรับ connections ---
.accept_loop:
    ; ตั้งค่า addrlen
    mov     dword [rel client_addrlen], 16
    
    ; accept client connection
    mov     rdi, [rel server_fd]
    lea     rsi, [rel client_addr]
    lea     rdx, [rel client_addrlen]
    mov     rax, SYS_ACCEPT
    syscall
    
    test    rax, rax
    js      .accept_loop    ; ถ้า error ให้ลองใหม่ (อาจ EINTR)
    
    mov     [rel client_fd], rax

    ; แสดงข้อความ client เชื่อมต่อ
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel msg_connect]
    mov     rdx, msg_connect_len
    syscall

    ;--- Echo Loop: รับข้อมูลและส่งกลับ ---
.echo_loop:
    ; รับข้อมูล
    mov     rdi, [rel client_fd]
    lea     rsi, [rel buffer]
    mov     rdx, BUFFER_SIZE
    xor     r10, r10
    mov     rax, 45         ; SYS_RECVFROM
    syscall
    
    test    rax, rax
    jle     .client_closed  ; 0 = closed, negative = error
    
    mov     rbx, rax        ; เก็บ bytes ที่รับได้
    
    ; ส่งข้อมูลกลับ (echo)
    mov     rdi, [rel client_fd]
    lea     rsi, [rel buffer]
    mov     rdx, rbx        ; จำนวน bytes ที่รับมา
    xor     r10, r10
    mov     rax, 44         ; SYS_SENDTO
    syscall
    
    jmp     .echo_loop

.client_closed:
    ; แสดงข้อความ client ตัดการเชื่อมต่อ
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel msg_close]
    mov     rdx, msg_close_len
    syscall
    
    ; ปิด client socket
    mov     rdi, [rel client_fd]
    mov     rax, SYS_CLOSE
    syscall
    
    ; รอ client ใหม่
    jmp     .accept_loop

.fatal_error:
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel msg_error]
    mov     rdx, msg_error_len
    syscall
    
    mov     rax, SYS_EXIT
    mov     rdi, 1
    syscall

; Build & Run:
; nasm -f elf64 tcp_echo_server.asm -o tcp_echo_server.o
; ld tcp_echo_server.o -o tcp_echo_server
; ./tcp_echo_server
; 
; ทดสอบด้วย: nc localhost 12345
```

---

## 18. Program: TCP Echo Client

```nasm
; tcp_echo_client.asm
; TCP Echo Client - ส่งข้อมูลไปยัง echo server และรับกลับมา

global _start

section .data
    ; Server info
    server_port     equ 12345
    
    ; ข้อความส่ง
    test_msg        db "Hello from Assembly Client!", 10
    test_msg_len    equ $ - test_msg
    
    msg_sent        db "ส่งข้อมูลแล้ว: "
    msg_sent_len    equ $ - msg_sent
    msg_received    db "ได้รับข้อมูลกลับ: "
    msg_received_len equ $ - msg_received
    newline         db 10
    
    msg_connected   db "เชื่อมต่อสำเร็จ!", 10
    msg_connected_len equ $ - msg_connected
    msg_conn_error  db "ไม่สามารถเชื่อมต่อได้!", 10
    msg_conn_error_len equ $ - msg_conn_error

; Syscall numbers
SYS_READ        equ 0
SYS_WRITE       equ 1
SYS_CLOSE       equ 3
SYS_SOCKET      equ 41
SYS_CONNECT     equ 42
SYS_EXIT        equ 60

AF_INET         equ 2
SOCK_STREAM     equ 1
STDOUT          equ 1
BUFFER_SIZE     equ 1024

section .bss
    sockfd      resq 1
    server_addr resb 16
    recv_buf    resb BUFFER_SIZE

section .text
_start:
    ;--- สร้าง socket ---
    mov     rdi, AF_INET
    mov     rsi, SOCK_STREAM
    xor     rdx, rdx
    mov     rax, SYS_SOCKET
    syscall
    
    test    rax, rax
    js      .error
    mov     [rel sockfd], rax

    ;--- ตั้งค่า server address ---
    lea     rbx, [rel server_addr]
    mov     word  [rbx],     AF_INET
    
    ; htons(12345) = 0x3930
    mov     ax, server_port
    xchg    al, ah
    mov     word [rbx + 2],  ax
    
    ; 127.0.0.1 ใน network byte order
    ; = 0x7F000001 big-endian
    ; stored little-endian = 01 00 00 7F
    mov     dword [rbx + 4], 0x0100007F
    mov     qword [rbx + 8], 0

    ;--- เชื่อมต่อ ---
    mov     rdi, [rel sockfd]
    lea     rsi, [rel server_addr]
    mov     rdx, 16
    mov     rax, SYS_CONNECT
    syscall
    
    test    rax, rax
    js      .conn_error

    ; แสดงข้อความเชื่อมต่อสำเร็จ
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel msg_connected]
    mov     rdx, msg_connected_len
    syscall

    ;--- ส่งข้อมูล ---
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel msg_sent]
    mov     rdx, msg_sent_len
    syscall
    
    ; แสดงข้อความที่จะส่ง
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel test_msg]
    mov     rdx, test_msg_len
    syscall
    
    ; ส่งข้อมูลผ่าน socket
    mov     rdi, [rel sockfd]
    lea     rsi, [rel test_msg]
    mov     rdx, test_msg_len
    xor     r10, r10        ; flags = 0
    mov     rax, 44         ; SYS_SENDTO
    syscall

    ;--- รับข้อมูลกลับ ---
    mov     rdi, [rel sockfd]
    lea     rsi, [rel recv_buf]
    mov     rdx, BUFFER_SIZE
    xor     r10, r10
    mov     rax, 45         ; SYS_RECVFROM
    syscall
    
    test    rax, rax
    jle     .done
    
    mov     rbx, rax        ; bytes received
    
    ; แสดงข้อมูลที่รับมา
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel msg_received]
    mov     rdx, msg_received_len
    syscall
    
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel recv_buf]
    mov     rdx, rbx
    syscall

.done:
    ; ปิด socket
    mov     rdi, [rel sockfd]
    mov     rax, SYS_CLOSE
    syscall
    
    mov     rax, SYS_EXIT
    xor     rdi, rdi
    syscall

.conn_error:
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel msg_conn_error]
    mov     rdx, msg_conn_error_len
    syscall
    jmp     .exit_error

.error:
.exit_error:
    mov     rax, SYS_EXIT
    mov     rdi, 1
    syscall
```

---

## 19. Program: UDP Server

```nasm
; udp_server.asm
; UDP Echo Server - รับ datagram และส่งกลับ
; ไม่มี connection, แต่ละ packet อิสระ

global _start

section .data
    msg_start   db "UDP Server เริ่มทำงานที่ port 54321...", 10
    msg_start_len equ $ - msg_start
    msg_recv    db "ได้รับ datagram จาก client", 10
    msg_recv_len equ $ - msg_recv
    msg_sent    db "ส่ง echo กลับแล้ว", 10
    msg_sent_len equ $ - msg_sent

SYS_WRITE       equ 1
SYS_CLOSE       equ 3
SYS_SOCKET      equ 41
SYS_BIND        equ 49
SYS_SENDTO      equ 44
SYS_RECVFROM    equ 45
SYS_SETSOCKOPT  equ 54
SYS_EXIT        equ 60

AF_INET         equ 2
SOCK_DGRAM      equ 2       ; UDP!
SOL_SOCKET      equ 1
SO_REUSEADDR    equ 2
STDOUT          equ 1
UDP_PORT        equ 54321
BUFFER_SIZE     equ 65535   ; UDP max payload

section .bss
    udp_fd          resq 1
    server_addr     resb 16
    client_addr     resb 16     ; ที่อยู่ของ client ที่ส่งมา
    client_addrlen  resd 1
    buffer          resb BUFFER_SIZE
    opt_val         resd 1

section .text
_start:
    ; แสดงข้อความเริ่มต้น
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel msg_start]
    mov     rdx, msg_start_len
    syscall

    ;--- สร้าง UDP socket ---
    mov     rdi, AF_INET
    mov     rsi, SOCK_DGRAM     ; UDP
    xor     rdx, rdx
    mov     rax, SYS_SOCKET
    syscall
    
    test    rax, rax
    js      .error
    mov     [rel udp_fd], rax

    ;--- SO_REUSEADDR ---
    mov     dword [rel opt_val], 1
    mov     rdi, [rel udp_fd]
    mov     rsi, SOL_SOCKET
    mov     rdx, SO_REUSEADDR
    lea     r10, [rel opt_val]
    mov     r8,  4
    mov     rax, SYS_SETSOCKOPT
    syscall

    ;--- bind ---
    lea     rbx, [rel server_addr]
    mov     word  [rbx],     AF_INET
    mov     ax, UDP_PORT
    xchg    al, ah
    mov     word  [rbx + 2], ax
    mov     dword [rbx + 4], 0      ; INADDR_ANY
    mov     qword [rbx + 8], 0
    
    mov     rdi, [rel udp_fd]
    lea     rsi, [rel server_addr]
    mov     rdx, 16
    mov     rax, SYS_BIND
    syscall
    
    test    rax, rax
    js      .error

    ;--- Main Loop: รับ datagrams ---
.recv_loop:
    ; ตั้งค่า client_addrlen
    mov     dword [rel client_addrlen], 16
    
    ; recvfrom - รับ datagram พร้อม client address
    mov     rdi, [rel udp_fd]
    lea     rsi, [rel buffer]
    mov     rdx, BUFFER_SIZE
    xor     r10, r10            ; flags = 0
    lea     r8,  [rel client_addr]      ; จะถูก fill ด้วย client address
    lea     r9,  [rel client_addrlen]
    mov     rax, SYS_RECVFROM
    syscall
    
    test    rax, rax
    js      .recv_loop          ; error, ลองใหม่
    jz      .recv_loop          ; 0 bytes?
    
    mov     rbx, rax            ; bytes received
    
    ; แสดงข้อความ
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel msg_recv]
    mov     rdx, msg_recv_len
    syscall
    
    ; ส่ง echo กลับไปยัง client
    ; ใช้ client_addr ที่ recvfrom fill ให้เรา
    mov     rdi, [rel udp_fd]
    lea     rsi, [rel buffer]
    mov     rdx, rbx            ; same number of bytes
    xor     r10, r10
    lea     r8,  [rel client_addr]      ; ส่งกลับหาคนที่ส่งมา
    mov     r9,  dword [rel client_addrlen]  ; addrlen (not pointer for sendto!)
    mov     rax, SYS_SENDTO
    syscall
    
    ; แสดงข้อความ
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel msg_sent]
    mov     rdx, msg_sent_len
    syscall
    
    jmp     .recv_loop

.error:
    mov     rdi, [rel udp_fd]
    mov     rax, SYS_CLOSE
    syscall
    
    mov     rax, SYS_EXIT
    mov     rdi, 1
    syscall
```

---

## 20. Program: HTTP/1.0 Server (Serve Static Files)

```nasm
; http_server.asm
; HTTP/1.0 Server - serve static HTML files
; รับ GET request และส่ง file กลับ

global _start

section .data
    ; Server config
    HTTP_PORT   equ 8080
    BACKLOG     equ 10
    
    ; HTTP response headers
    http_ok         db "HTTP/1.0 200 OK", 13, 10
                    db "Content-Type: text/html", 13, 10
                    db "Connection: close", 13, 10, 13, 10
    http_ok_len     equ $ - http_ok
    
    http_404        db "HTTP/1.0 404 Not Found", 13, 10
                    db "Content-Type: text/html", 13, 10
                    db "Connection: close", 13, 10, 13, 10
                    db "<html><body><h1>404 Not Found</h1></body></html>", 13, 10
    http_404_len    equ $ - http_404
    
    http_400        db "HTTP/1.0 400 Bad Request", 13, 10
                    db "Content-Type: text/html", 13, 10
                    db "Connection: close", 13, 10, 13, 10
                    db "<html><body><h1>400 Bad Request</h1></body></html>"
    http_400_len    equ $ - http_400
    
    ; Default page content
    index_html      db "<!DOCTYPE html>", 10
                    db "<html>", 10
                    db "<head><title>Assembly HTTP Server</title></head>", 10
                    db "<body>", 10
                    db "<h1>Hello from Assembly!</h1>", 10
                    db "<p>This page is served by a pure Assembly HTTP server</p>", 10
                    db "</body>", 10
                    db "</html>", 10
    index_html_len  equ $ - index_html
    
    ; GET method string สำหรับ compare
    get_str     db "GET "
    get_str_len equ $ - get_str
    
    ; root path
    root_path   db "/ "
    
    ; Log messages
    msg_server_start db "HTTP Server เริ่มที่ port 8080...", 10
    msg_server_start_len equ $ - msg_server_start
    msg_request  db "ได้รับ HTTP request", 10
    msg_request_len equ $ - msg_request

; Syscall numbers
SYS_READ        equ 0
SYS_WRITE       equ 1
SYS_OPEN        equ 2
SYS_CLOSE       equ 3
SYS_SOCKET      equ 41
SYS_ACCEPT      equ 43
SYS_BIND        equ 49
SYS_LISTEN      equ 50
SYS_SENDTO      equ 44
SYS_RECVFROM    equ 45
SYS_SETSOCKOPT  equ 54
SYS_EXIT        equ 60
SYS_FORK        equ 57

AF_INET         equ 2
SOCK_STREAM     equ 1
SOL_SOCKET      equ 1
SO_REUSEADDR    equ 2
STDOUT          equ 1

O_RDONLY        equ 0

REQUEST_SIZE    equ 4096
FILE_BUF_SIZE   equ 8192

section .bss
    server_fd       resq 1
    client_fd       resq 1
    server_addr     resb 16
    client_addr     resb 16
    client_addrlen  resd 1
    opt_val         resd 1
    request_buf     resb REQUEST_SIZE
    file_buf        resb FILE_BUF_SIZE
    path_buf        resb 256            ; extracted path
    
section .text
_start:
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel msg_server_start]
    mov     rdx, msg_server_start_len
    syscall

    ;--- สร้าง socket ---
    mov     rdi, AF_INET
    mov     rsi, SOCK_STREAM
    xor     rdx, rdx
    mov     rax, SYS_SOCKET
    syscall
    test    rax, rax
    js      .fatal
    mov     [rel server_fd], rax

    ;--- SO_REUSEADDR ---
    mov     dword [rel opt_val], 1
    mov     rdi, [rel server_fd]
    mov     rsi, SOL_SOCKET
    mov     rdx, SO_REUSEADDR
    lea     r10, [rel opt_val]
    mov     r8,  4
    mov     rax, SYS_SETSOCKOPT
    syscall

    ;--- bind port 8080 ---
    lea     rbx, [rel server_addr]
    mov     word  [rbx],     AF_INET
    mov     ax, HTTP_PORT
    xchg    al, ah
    mov     word  [rbx + 2], ax
    mov     dword [rbx + 4], 0
    mov     qword [rbx + 8], 0
    
    mov     rdi, [rel server_fd]
    lea     rsi, [rel server_addr]
    mov     rdx, 16
    mov     rax, SYS_BIND
    syscall
    test    rax, rax
    js      .fatal

    ;--- listen ---
    mov     rdi, [rel server_fd]
    mov     rsi, BACKLOG
    mov     rax, SYS_LISTEN
    syscall
    test    rax, rax
    js      .fatal

    ;--- Main accept loop ---
.accept_loop:
    mov     dword [rel client_addrlen], 16
    
    mov     rdi, [rel server_fd]
    lea     rsi, [rel client_addr]
    lea     rdx, [rel client_addrlen]
    mov     rax, SYS_ACCEPT
    syscall
    
    test    rax, rax
    js      .accept_loop
    mov     [rel client_fd], rax

    ; log request
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel msg_request]
    mov     rdx, msg_request_len
    syscall

    ; handle this client
    call    handle_client
    
    ; ปิด client
    mov     rdi, [rel client_fd]
    mov     rax, SYS_CLOSE
    syscall
    
    jmp     .accept_loop

.fatal:
    mov     rax, SYS_EXIT
    mov     rdi, 1
    syscall

;--- handle_client: อ่าน request และส่ง response ---
handle_client:
    ; รับ HTTP request
    mov     rdi, [rel client_fd]
    lea     rsi, [rel request_buf]
    mov     rdx, REQUEST_SIZE - 1
    xor     r10, r10
    mov     rax, SYS_RECVFROM
    syscall
    
    test    rax, rax
    jle     .done
    
    ; null-terminate
    mov     byte [rel request_buf + rax], 0
    
    ; ตรวจสอบว่าเป็น GET request
    lea     rsi, [rel request_buf]
    ; ตรวจ "GET "
    cmp     dword [rsi], 0x20544547  ; "GET " in little-endian
    jne     .bad_request
    
    ; ตรวจสอบว่าเป็น path "/"
    ; request: "GET / HTTP/1.0\r\n..."
    ; offset 4 = start of path
    cmp     byte [rsi + 4], '/'
    jne     .not_found
    
    ; path "/" -> ส่ง index.html
    cmp     byte [rsi + 5], ' '
    je      .send_index
    
    ; path อื่นๆ -> 404
    jmp     .not_found

.send_index:
    ; ส่ง 200 OK header
    mov     rdi, [rel client_fd]
    lea     rsi, [rel http_ok]
    mov     rdx, http_ok_len
    xor     r10, r10
    mov     rax, SYS_SENDTO
    syscall
    
    ; ส่ง HTML content
    mov     rdi, [rel client_fd]
    lea     rsi, [rel index_html]
    mov     rdx, index_html_len
    xor     r10, r10
    mov     rax, SYS_SENDTO
    syscall
    jmp     .done

.not_found:
    ; ส่ง 404
    mov     rdi, [rel client_fd]
    lea     rsi, [rel http_404]
    mov     rdx, http_404_len
    xor     r10, r10
    mov     rax, SYS_SENDTO
    syscall
    jmp     .done

.bad_request:
    ; ส่ง 400
    mov     rdi, [rel client_fd]
    lea     rsi, [rel http_400]
    mov     rdx, http_400_len
    xor     r10, r10
    mov     rax, SYS_SENDTO
    syscall

.done:
    ret

; Build & Test:
; nasm -f elf64 http_server.asm -o http_server.o
; ld http_server.o -o http_server
; ./http_server
; curl http://localhost:8080/
```

---

## 21. Program: Port Scanner

```nasm
; port_scanner.asm
; TCP Port Scanner - สแกน port ที่เปิดอยู่บน target host
; ใช้ non-blocking connect เพื่อความเร็ว

global _start

section .data
    msg_scanning    db "กำลังสแกน ports...", 10
    msg_scanning_len equ $ - msg_scanning
    msg_open        db "OPEN: port "
    msg_open_len    equ $ - msg_open
    msg_newline     db 10
    msg_done        db "การสแกนเสร็จสิ้น", 10
    msg_done_len    equ $ - msg_done
    
    ; Target: localhost (127.0.0.1)
    TARGET_IP   equ 0x0100007F  ; 127.0.0.1 little-endian

; Syscall numbers
SYS_WRITE       equ 1
SYS_CLOSE       equ 3
SYS_SOCKET      equ 41
SYS_CONNECT     equ 42
SYS_EXIT        equ 60
SYS_FCNTL       equ 72
SYS_SELECT      equ 23

AF_INET         equ 2
SOCK_STREAM     equ 1
F_GETFL         equ 3
F_SETFL         equ 4
O_NONBLOCK      equ 2048
STDOUT          equ 1
EINPROGRESS     equ 115     ; errno for non-blocking connect in progress

START_PORT      equ 1
END_PORT        equ 1024    ; สแกน ports 1-1024

section .bss
    scan_sockfd     resq 1
    target_addr     resb 16
    fd_set_write    resb 128    ; fd_set สำหรับ select
    fd_set_except   resb 128
    timeout         resq 2      ; struct timeval
    port_str        resb 8      ; port number string

section .text
_start:
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel msg_scanning]
    mov     rdx, msg_scanning_len
    syscall

    ; r15 = current port
    mov     r15, START_PORT

.scan_loop:
    cmp     r15, END_PORT
    jg      .scan_done

    ;--- สร้าง socket ใหม่สำหรับแต่ละ port ---
    mov     rdi, AF_INET
    mov     rsi, SOCK_STREAM
    xor     rdx, rdx
    mov     rax, SYS_SOCKET
    syscall
    
    test    rax, rax
    js      .next_port      ; ถ้าสร้างไม่ได้ ข้ามไป
    mov     [rel scan_sockfd], rax

    ;--- ตั้ง non-blocking ---
    mov     rdi, [rel scan_sockfd]
    mov     rsi, F_GETFL
    xor     rdx, rdx
    mov     rax, SYS_FCNTL
    syscall
    
    or      rax, O_NONBLOCK
    mov     rdx, rax
    
    mov     rdi, [rel scan_sockfd]
    mov     rsi, F_SETFL
    mov     rax, SYS_FCNTL
    syscall

    ;--- ตั้งค่า target address ---
    lea     rbx, [rel target_addr]
    mov     word  [rbx],     AF_INET
    
    ; htons(port)
    mov     ax, r15w
    xchg    al, ah
    mov     word  [rbx + 2], ax
    
    mov     dword [rbx + 4], TARGET_IP
    mov     qword [rbx + 8], 0

    ;--- non-blocking connect ---
    mov     rdi, [rel scan_sockfd]
    lea     rsi, [rel target_addr]
    mov     rdx, 16
    mov     rax, SYS_CONNECT
    syscall
    
    ; Non-blocking connect จะ return -EINPROGRESS
    ; ต้องใช้ select เพื่อรอ
    neg     rax
    cmp     rax, EINPROGRESS
    jne     .close_port     ; ถ้าไม่ใช่ EINPROGRESS = error = port closed

    ;--- ใช้ select รอ connect สำเร็จ ---
    ; Zero out fd_sets
    lea     rdi, [rel fd_set_write]
    mov     ecx, 16
    xor     eax, eax
    rep stosq
    
    lea     rdi, [rel fd_set_except]
    mov     ecx, 16
    xor     eax, eax
    rep stosq
    
    ; FD_SET(scan_sockfd, &fd_set_write)
    mov     eax, dword [rel scan_sockfd]
    mov     ecx, eax
    shr     ecx, 6
    and     eax, 63
    bts     qword [rel fd_set_write + rcx*8], rax
    
    ; FD_SET(scan_sockfd, &fd_set_except)
    mov     eax, dword [rel scan_sockfd]
    mov     ecx, eax
    shr     ecx, 6
    and     eax, 63
    bts     qword [rel fd_set_except + rcx*8], rax
    
    ; timeout = 100ms = 0 sec, 100000 usec
    mov     qword [rel timeout],     0
    mov     qword [rel timeout + 8], 100000
    
    ; select
    mov     rdi, [rel scan_sockfd]
    inc     rdi                 ; nfds = fd + 1
    xor     rsi, rsi            ; readfds = NULL
    lea     rdx, [rel fd_set_write]
    lea     r10, [rel fd_set_except]
    lea     r8,  [rel timeout]
    mov     rax, SYS_SELECT
    syscall
    
    test    rax, rax
    jle     .close_port     ; timeout หรือ error = port closed
    
    ; ตรวจสอบว่า except set หรือไม่ (error = port closed/refused)
    mov     eax, dword [rel scan_sockfd]
    mov     ecx, eax
    shr     ecx, 6
    and     eax, 63
    bt      qword [rel fd_set_except + rcx*8], rax
    jc      .close_port     ; CF=1 = exception = port closed
    
    ; ตรวจสอบ write set (success = port open)
    mov     eax, dword [rel scan_sockfd]
    mov     ecx, eax
    shr     ecx, 6
    and     eax, 63
    bt      qword [rel fd_set_write + rcx*8], rax
    jnc     .close_port     ; CF=0 = not writable = not connected
    
    ;--- PORT OPEN! ---
    ; แสดงข้อความ
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel msg_open]
    mov     rdx, msg_open_len
    syscall
    
    ; แปลง port number เป็น string และแสดง
    mov     rax, r15
    lea     rdi, [rel port_str]
    call    itoa
    
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel port_str]
    mov     rdx, rcx        ; length returned by itoa
    syscall
    
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel msg_newline]
    mov     rdx, 1
    syscall

.close_port:
    ; ปิด socket
    mov     rdi, [rel scan_sockfd]
    mov     rax, SYS_CLOSE
    syscall

.next_port:
    inc     r15
    jmp     .scan_loop

.scan_done:
    mov     rax, SYS_WRITE
    mov     rdi, STDOUT
    lea     rsi, [rel msg_done]
    mov     rdx, msg_done_len
    syscall
    
    mov     rax, SYS_EXIT
    xor     rdi, rdi
    syscall

;--- itoa: แปลง integer เป็น ASCII string ---
; Input: rax = number, rdi = buffer pointer
; Output: rcx = length
itoa:
    push    rbx
    push    rdx
    
    lea     rbx, [rdi + 7]  ; เริ่มจากท้าย buffer
    mov     byte [rbx], 0   ; null terminator
    dec     rbx
    
    xor     rcx, rcx        ; length counter
    
    test    rax, rax
    jnz     .convert
    ; special case: 0
    mov     byte [rbx], '0'
    dec     rbx
    inc     rcx
    jmp     .done
    
.convert:
    test    rax, rax
    jz      .done
    
    xor     rdx, rdx
    mov     r8d, 10
    div     r8              ; rax = quotient, rdx = remainder
    
    add     dl, '0'
    mov     byte [rbx], dl
    dec     rbx
    inc     rcx
    
    jmp     .convert
    
.done:
    lea     rdi, [rbx + 1]  ; point to start of number
    pop     rdx
    pop     rbx
    ret
```

---

## 22. Advanced: epoll-based Multi-connection Server

```nasm
; epoll_server.asm
; High-performance epoll server ใช้ edge-triggered mode
; รองรับ concurrent connections จำนวนมาก

global _start

section .data
    MSG_WELCOME     db "Welcome! You are connected to the epoll server.", 10
    MSG_WELCOME_LEN equ $ - MSG_WELCOME
    
    PORT            equ 9090
    MAX_EVENTS      equ 64
    BUFFER_SIZE     equ 4096
    BACKLOG         equ 128

SYS_READ        equ 0
SYS_WRITE       equ 1
SYS_CLOSE       equ 3
SYS_SOCKET      equ 41
SYS_ACCEPT4     equ 288
SYS_BIND        equ 49
SYS_LISTEN      equ 50
SYS_SENDTO      equ 44
SYS_RECVFROM    equ 45
SYS_SETSOCKOPT  equ 54
SYS_FCNTL       equ 72
SYS_EPOLL_CREATE1 equ 291
SYS_EPOLL_CTL   equ 233
SYS_EPOLL_WAIT  equ 232
SYS_EXIT        equ 60

AF_INET         equ 2
SOCK_STREAM     equ 1
SOCK_NONBLOCK   equ 2048
SOCK_CLOEXEC    equ 524288
SOL_SOCKET      equ 1
SO_REUSEADDR    equ 2
SO_REUSEPORT    equ 15
F_GETFL         equ 3
F_SETFL         equ 4
O_NONBLOCK      equ 2048
EPOLL_CLOEXEC   equ 524288
EPOLL_CTL_ADD   equ 1
EPOLL_CTL_DEL   equ 2
EPOLLIN         equ 0x001
EPOLLOUT        equ 0x004
EPOLLERR        equ 0x008
EPOLLHUP        equ 0x010
EPOLLET         equ 0x80000000
STDOUT          equ 1
EAGAIN          equ 11

section .bss
    server_fd       resq 1
    epfd            resq 1
    server_addr     resb 16
    client_addr     resb 16
    client_addrlen  resd 1
    opt_val         resd 1
    ; events array: MAX_EVENTS * 12 bytes each
    events          resb MAX_EVENTS * 12
    buffer          resb BUFFER_SIZE
    ep_event        resb 12     ; temp epoll_event

section .text

;--- set_nonblocking helper ---
set_nonblocking:
    ; rdi = fd
    push    rdi
    mov     rsi, F_GETFL
    xor     rdx, rdx
    mov     rax, SYS_FCNTL
    syscall
    test    rax, rax
    js      .err
    or      rax, O_NONBLOCK
    mov     rdx, rax
    pop     rdi
    push    rdi
    mov     rsi, F_SETFL
    mov     rax, SYS_FCNTL
    syscall
.err:
    pop     rdi
    ret

;--- add_to_epoll: เพิ่ม fd เข้า epoll ---
; rdi = epfd, rsi = fd, rdx = events mask
add_to_epoll:
    push    rbx
    push    r12
    push    r13
    
    mov     r12, rdi    ; epfd
    mov     r13, rsi    ; fd
    
    ; ตั้งค่า ep_event structure
    lea     rbx, [rel ep_event]
    mov     dword [rbx],     edx         ; events
    mov     qword [rbx + 4], r13         ; data.fd (64-bit)
    
    mov     rdi, r12                     ; epfd
    mov     rsi, EPOLL_CTL_ADD
    mov     rdx, r13                     ; fd
    lea     r10, [rel ep_event]
    mov     rax, SYS_EPOLL_CTL
    syscall
    
    pop     r13
    pop     r12
    pop     rbx
    ret

_start:
    ;--- สร้าง server socket ---
    mov     rdi, AF_INET
    mov     rsi, SOCK_STREAM
    xor     rdx, rdx
    mov     rax, SYS_SOCKET
    syscall
    test    rax, rax
    js      .fatal
    mov     [rel server_fd], rax

    ;--- SO_REUSEADDR + SO_REUSEPORT ---
    mov     dword [rel opt_val], 1
    mov     rdi, [rel server_fd]
    mov     rsi, SOL_SOCKET
    mov     rdx, SO_REUSEADDR
    lea     r10, [rel opt_val]
    mov     r8,  4
    mov     rax, SYS_SETSOCKOPT
    syscall
    
    mov     rdi, [rel server_fd]
    mov     rsi, SOL_SOCKET
    mov     rdx, SO_REUSEPORT
    lea     r10, [rel opt_val]
    mov     r8,  4
    mov     rax, SYS_SETSOCKOPT
    syscall

    ;--- ตั้ง server socket เป็น non-blocking ---
    mov     rdi, [rel server_fd]
    call    set_nonblocking

    ;--- bind ---
    lea     rbx, [rel server_addr]
    mov     word  [rbx],     AF_INET
    mov     ax, PORT
    xchg    al, ah
    mov     word  [rbx + 2], ax
    mov     dword [rbx + 4], 0
    mov     qword [rbx + 8], 0
    
    mov     rdi, [rel server_fd]
    lea     rsi, [rel server_addr]
    mov     rdx, 16
    mov     rax, SYS_BIND
    syscall
    test    rax, rax
    js      .fatal

    ;--- listen ---
    mov     rdi, [rel server_fd]
    mov     rsi, BACKLOG
    mov     rax, SYS_LISTEN
    syscall
    test    rax, rax
    js      .fatal

    ;--- สร้าง epoll instance ---
    mov     rdi, EPOLL_CLOEXEC
    mov     rax, SYS_EPOLL_CREATE1
    syscall
    test    rax, rax
    js      .fatal
    mov     [rel epfd], rax

    ;--- เพิ่ม server_fd เข้า epoll ---
    mov     rdi, [rel epfd]
    mov     rsi, [rel server_fd]
    mov     rdx, EPOLLIN | EPOLLET     ; edge-triggered
    call    add_to_epoll

    ;=== EVENT LOOP ===
.event_loop:
    mov     rdi, [rel epfd]
    lea     rsi, [rel events]
    mov     rdx, MAX_EVENTS
    mov     r10, -1             ; infinite timeout
    mov     rax, SYS_EPOLL_WAIT
    syscall
    
    test    rax, rax
    js      .event_loop         ; interrupted, retry
    jz      .event_loop         ; no events
    
    ; r14 = event count, r15 = event index
    mov     r14, rax
    xor     r15, r15

.process_event:
    cmp     r15, r14
    jge     .event_loop         ; ประมวลผลครบ
    
    ; คำนวณ pointer ไปยัง event[r15]
    imul    rbx, r15, 12        ; offset = index * 12
    lea     rbx, [rel events + rbx]
    
    mov     r12d, dword [rbx]           ; events flags
    mov     r13, qword  [rbx + 4]       ; data (fd)
    
    ; ตรวจสอบว่าเป็น server socket หรือไม่
    cmp     r13, [rel server_fd]
    je      .handle_new_connection
    
    ; ตรวจ error/hangup
    test    r12d, EPOLLERR | EPOLLHUP
    jnz     .handle_close
    
    ; ตรวจ readable
    test    r12d, EPOLLIN
    jnz     .handle_read
    
    jmp     .next_event

.handle_new_connection:
    ; รับ connections ทั้งหมดจนหมด (edge-triggered!)
.accept_all:
    mov     dword [rel client_addrlen], 16
    
    mov     rdi, [rel server_fd]
    lea     rsi, [rel client_addr]
    lea     rdx, [rel client_addrlen]
    mov     r10, SOCK_NONBLOCK | SOCK_CLOEXEC
    mov     rax, SYS_ACCEPT4
    syscall
    
    test    rax, rax
    js      .accept_done        ; EAGAIN = no more connections
    
    ; เพิ่ม client fd เข้า epoll
    mov     r8, rax             ; client fd
    push    r8
    
    mov     rdi, [rel epfd]
    mov     rsi, r8
    mov     rdx, EPOLLIN | EPOLLET | EPOLLRDHUP
    call    add_to_epoll
    
    ; ส่ง welcome message
    pop     r8
    mov     rdi, r8
    lea     rsi, [rel MSG_WELCOME]
    mov     rdx, MSG_WELCOME_LEN
    xor     r10, r10
    mov     rax, SYS_SENDTO
    syscall
    
    jmp     .accept_all         ; ลอง accept อีกครั้ง

.accept_done:
    jmp     .next_event

.handle_read:
    ; อ่านข้อมูลจนหมด (edge-triggered!)
.read_all:
    mov     rdi, r13            ; client fd
    lea     rsi, [rel buffer]
    mov     rdx, BUFFER_SIZE
    xor     r10, r10
    mov     rax, SYS_RECVFROM
    syscall
    
    test    rax, rax
    js      .read_done          ; EAGAIN = no more data
    jz      .handle_close       ; 0 = connection closed
    
    ; Echo กลับ
    mov     rbp, rax            ; bytes read
    mov     rdi, r13
    lea     rsi, [rel buffer]
    mov     rdx, rbp
    xor     r10, r10
    mov     rax, SYS_SENDTO
    syscall
    
    jmp     .read_all

.read_done:
    jmp     .next_event

.handle_close:
    ; ลบออกจาก epoll และปิด
    mov     rdi, [rel epfd]
    mov     rsi, EPOLL_CTL_DEL
    mov     rdx, r13
    xor     r10, r10
    mov     rax, SYS_EPOLL_CTL
    syscall
    
    mov     rdi, r13
    mov     rax, SYS_CLOSE
    syscall

.next_event:
    inc     r15
    jmp     .process_event

.fatal:
    mov     rax, SYS_EXIT
    mov     rdi, 1
    syscall

EPOLLRDHUP  equ 0x2000
```

---

## 23. Helper Functions สำหรับ Network Programming

### 23.1 IP Address Formatting

```nasm
; format_ip: แปลง 32-bit IP (network byte order) เป็น dotted decimal string
; Input: eax = IP address (network byte order), rdi = output buffer
; Output: rcx = string length
format_ip:
    push    rbx
    push    rdx
    push    r8
    push    r9
    
    mov     r8, rdi         ; เก็บ buffer pointer
    
    ; bswap เพื่อ process octet ทีละตัว (most significant first)
    bswap   eax
    
    ; 4 octets
    mov     r9d, 4
.octet_loop:
    ; ดึง octet สูงสุด
    mov     bl,  al
    shr     eax, 8          ; เลื่อนไปดึง octet ถัดไป
    
    ; แปลง byte เป็น 1-3 digit decimal
    movzx   ecx, bl
    ; แปลง ecx เป็น ASCII ใน buffer
    push    rax
    push    r9
    mov     rax, rcx
    call    byte_to_ascii   ; output ใน rdi, length ใน rcx
    pop     r9
    pop     rax
    
    ; ถ้าไม่ใช่ octet สุดท้าย ใส่ '.'
    dec     r9d
    jz      .no_dot
    mov     byte [rdi], '.'
    inc     rdi
.no_dot:
    
    test    r9d, r9d
    jnz     .octet_loop
    
    ; คำนวณ length
    mov     rcx, rdi
    sub     rcx, r8
    
    pop     r9
    pop     r8
    pop     rdx
    pop     rbx
    ret

; byte_to_ascii: แปลง byte เป็น decimal string
; Input: rax = value (0-255), rdi = output buffer
; Output: rdi updated, rcx = bytes written
byte_to_ascii:
    push    rdx
    push    r8
    
    mov     r8, rdi
    xor     rcx, rcx
    
    cmp     rax, 100
    jl      .lt100
    ; >= 100: แสดง hundreds
    xor     rdx, rdx
    mov     r9d, 100
    div     r9d
    add     al, '0'
    mov     byte [rdi], al
    inc     rdi
    inc     rcx
    mov     rax, rdx
    
.lt100:
    cmp     rax, 10
    jl      .lt10
    ; >= 10: แสดง tens
    xor     rdx, rdx
    mov     r9d, 10
    div     r9d
    add     al, '0'
    mov     byte [rdi], al
    inc     rdi
    inc     rcx
    mov     rax, rdx
    
.lt10:
    add     al, '0'
    mov     byte [rdi], al
    inc     rdi
    inc     rcx
    
    pop     r8
    pop     rdx
    ret
```

### 23.2 Makefile สำหรับ build network programs

```makefile
# Makefile สำหรับ Assembly Network Programs
AS = nasm
LD = ld
ASFLAGS = -f elf64
LDFLAGS = 

PROGRAMS = tcp_echo_server tcp_echo_client udp_server http_server port_scanner epoll_server

all: $(PROGRAMS)

%: %.asm
	$(AS) $(ASFLAGS) $< -o $@.o
	$(LD) $(LDFLAGS) $@.o -o $@
	rm $@.o

clean:
	rm -f $(PROGRAMS) *.o

.PHONY: all clean
```

---

## 24. การ Debug Network Programs

### 24.1 ใช้ strace ดู syscalls

```bash
# ดู network syscalls ทั้งหมด
strace -e trace=network ./tcp_echo_server

# ดู syscalls แบบ verbose พร้อม arguments
strace -v -e socket,bind,listen,accept,connect,send,recv ./tcp_echo_client

# ดู error codes
strace -e trace=network -e fault=connect:error=ECONNREFUSED ./port_scanner
```

### 24.2 ใช้ netstat/ss ดู connections

```bash
# ดู listening sockets
ss -tlnp

# ดู established connections
ss -tnp

# ดู UDP sockets
ss -ulnp

# ดู connections บน port 8080
ss -tnp '( dport = :8080 or sport = :8080 )'
```

### 24.3 ใช้ tcpdump capture packets

```bash
# Capture TCP traffic บน port 12345
tcpdump -i lo -A 'tcp port 12345'

# Capture พร้อมดู hex
tcpdump -i lo -XX 'port 8080'

# Save ลง file
tcpdump -i lo -w capture.pcap 'port 9090'
```

### 24.4 Common Errors และวิธีแก้

```
EADDRINUSE (98): Address already in use
  -> ใช้ SO_REUSEADDR หรือรอ socket timeout

ECONNREFUSED (111): Connection refused  
  -> Port ไม่ได้ถูก listen อยู่

ETIMEDOUT (110): Connection timed out
  -> Network ไม่ถึง host

EAGAIN (11): Resource temporarily unavailable
  -> Non-blocking socket ไม่มีข้อมูล/connection

EMFILE (24): Too many open files
  -> เปิด file descriptor เกิน limit
  -> ulimit -n 65535
  
ENOBUFS (105): No buffer space available
  -> Socket buffer เต็ม
```

---

## 25. Performance Tips สำหรับ Assembly Network Code

### 25.1 Vectorized memory operations

```nasm
; ใช้ SSE/AVX สำหรับ zero-fill sockaddr structures
vzeroaddr:
    ; Zero 16-byte sockaddr_in ด้วย SSE
    pxor    xmm0, xmm0
    movdqu  [server_addr], xmm0
    
; Zero 28-byte sockaddr_in6
    pxor    xmm0, xmm0
    movdqu  [server_addr6],      xmm0
    movq    [server_addr6 + 16], xmm0   ; tail 12 bytes
```

### 25.2 Batch accept

```nasm
; รับ connections หลายอันพร้อมกันก่อน process
batch_accept:
    xor     r12, r12        ; count = 0
.loop:
    mov     dword [client_addrlen], 16
    
    mov     rdi, [server_fd]
    lea     rsi, [client_addr_array + r12 * 16]
    lea     rdx, [client_addrlen]
    mov     r10, SOCK_NONBLOCK | SOCK_CLOEXEC
    mov     rax, SYS_ACCEPT4
    syscall
    
    test    rax, rax
    js      .done           ; no more = EAGAIN
    
    mov     [client_fd_array + r12 * 8], rax
    inc     r12
    cmp     r12, MAX_BATCH
    jl      .loop
.done:
    ; r12 = number of accepted connections
    ret
```

### 25.3 TCP_CORK สำหรับ HTTP responses

```nasm
; TCP_CORK - หน่วงการส่งจนกว่า buffer จะเต็ม หรือ cork ถูกยกเลิก
; ดีสำหรับ HTTP responses ที่ส่ง header + body แยก
TCP_CORK    equ 3

enable_cork:
    mov     dword [opt_val], 1
    mov     rdi, [client_fd]
    mov     rsi, IPPROTO_TCP
    mov     rdx, TCP_CORK
    lea     r10, [opt_val]
    mov     r8, 4
    mov     rax, SYS_SETSOCKOPT
    syscall
    ret

disable_cork:
    mov     dword [opt_val], 0
    mov     rdi, [client_fd]
    mov     rsi, IPPROTO_TCP
    mov     rdx, TCP_CORK
    lea     r10, [opt_val]
    mov     r8, 4
    mov     rax, SYS_SETSOCKOPT
    syscall
    ret

; Pattern การใช้งาน:
; call enable_cork
; ; ส่ง HTTP header
; ; ส่ง HTTP body
; call disable_cork   ; -> kernel ส่งทั้งหมดพร้อมกัน
```

---

## 26. สรุป (Summary)

### 26.1 Syscall Reference Table

| Operation | Syscall | Number | Key Args |
|-----------|---------|--------|----------|
| Create socket | socket | 41 | domain, type, protocol |
| Bind address | bind | 49 | sockfd, addr*, addrlen |
| Listen | listen | 50 | sockfd, backlog |
| Accept (basic) | accept | 43 | sockfd, addr*, addrlen* |
| Accept (flags) | accept4 | 288 | sockfd, addr*, addrlen*, flags |
| Connect | connect | 42 | sockfd, addr*, addrlen |
| Send | sendto | 44 | sockfd, buf, len, flags, addr, addrlen |
| Receive | recvfrom | 45 | sockfd, buf, len, flags, addr*, addrlen* |
| Set options | setsockopt | 54 | sockfd, level, optname, optval*, optlen |
| Non-blocking | fcntl | 72 | fd, F_SETFL, flags\|O_NONBLOCK |
| Multiplex | select | 23 | nfds, readfds, writefds, exceptfds, timeout |
| Poll | poll | 7 | fds*, nfds, timeout_ms |
| Create epoll | epoll_create1 | 291 | flags |
| Modify epoll | epoll_ctl | 233 | epfd, op, fd, event* |
| Wait epoll | epoll_wait | 232 | epfd, events*, maxevents, timeout_ms |
| Shutdown | shutdown | 48 | sockfd, how |
| Close | close | 3 | fd |

### 26.2 Structure Sizes

| Structure | Size | Description |
|-----------|------|-------------|
| sockaddr_in | 16 bytes | IPv4 address |
| sockaddr_in6 | 28 bytes | IPv6 address |
| pollfd | 8 bytes | Poll file descriptor |
| epoll_event | 12 bytes | Epoll event (packed) |
| timeval | 16 bytes | select() timeout |
| fd_set | 128 bytes | select() fd bitmap |

### 26.3 Byte Order Functions

```nasm
; Quick reference:
; htons(x) -> xchg al, ah  (16-bit swap)
; ntohs(x) -> xchg al, ah  (same operation)
; htonl(x) -> bswap eax    (32-bit swap)
; ntohl(x) -> bswap eax    (same operation)
; htonq(x) -> bswap rax    (64-bit swap, if needed)
```

### 26.4 เลือกใช้ I/O Multiplexing

- **select**: ใช้เมื่อ fd < 1024, portable, simple
- **poll**: ใช้เมื่อมี fds จำนวนน้อย-ปานกลาง, ไม่จำกัด fd
- **epoll**: ใช้เมื่อต้องการ high-performance, fds จำนวนมาก, Linux-specific

---

## แบบฝึกหัด (Exercises)

1. แก้ไข TCP Echo Server ให้รองรับหลาย clients พร้อมกันโดยใช้ fork()
2. เพิ่ม HTTP server ให้ serve ไฟล์จาก disk โดยใช้ open/read syscalls
3. สร้าง UDP chat server ที่ broadcast ข้อความไปยังทุก clients
4. ปรับ port scanner ให้สแกน IPv6 addresses
5. เขียน HTTP/1.1 client ที่รองรับ persistent connections (keep-alive)
6. สร้าง proxy server ที่ forward traffic ระหว่าง 2 connections
7. ใช้ epoll + thread pool เพื่อ handle high concurrent connections
8. เพิ่ม TLS/SSL โดยใช้ OpenSSL library ผ่าน dynamic linking

---

*จบ Part 058: Network Programming ใน Assembly*

*Part นี้ครอบคลุมการใช้งาน Linux network syscalls โดยตรง ตั้งแต่การสร้าง socket, การ bind/listen/accept สำหรับ server, การ connect สำหรับ client, การส่ง/รับข้อมูลทั้ง TCP และ UDP, การตั้งค่า socket options, และการใช้ I/O multiplexing ด้วย select/poll/epoll พร้อมโปรแกรมตัวอย่างที่ใช้งานได้จริง*
