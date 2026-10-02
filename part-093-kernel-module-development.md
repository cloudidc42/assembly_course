# Part 093: Linux Kernel Module Development

## บทนำ (Introduction)

Linux Kernel Module Development คือการเขียนโค้ดที่ทำงานใน **kernel space** ซึ่งมีสิทธิ์การเข้าถึงฮาร์ดแวร์โดยตรง, จัดการ interrupt, และเชื่อมต่อกับ device driver framework ของ Linux kernel

การพัฒนา kernel module แตกต่างจาก userspace programming อย่างสิ้นเชิง:
- ไม่มี standard C library (libc)
- ไม่สามารถใช้ floating point ได้ (โดยปกติ)
- Stack size จำกัดที่ 8KB
- Bug ใดๆ อาจทำให้ระบบ crash ได้ทันที (kernel panic)
- ต้องจัดการ memory และ synchronization ด้วยตัวเอง

---

## 1. Kernel Module Structure พื้นฐาน

### 1.1 โครงสร้าง Module ที่สมบูรณ์

```c
// hello_module.c - Simple kernel module
#include <linux/module.h>      // MODULE_LICENSE, module_init, module_exit
#include <linux/kernel.h>      // printk, KERN_INFO
#include <linux/init.h>        // __init, __exit macros

// Module metadata - บังคับต้องมี MODULE_LICENSE
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name <your@email.com>");
MODULE_DESCRIPTION("A simple Hello World kernel module");
MODULE_VERSION("1.0");

// __init - บอก kernel ว่าฟังก์ชันนี้ใช้แค่ตอน init แล้วสามารถ free memory ได้
static int __init hello_init(void)
{
    // printk ใช้แทน printf ใน kernel space
    // KERN_INFO คือ log level (0=emergency, 7=debug)
    printk(KERN_INFO "Hello, Kernel World!\n");
    printk(KERN_INFO "Module loaded at address: %p\n", hello_init);
    
    // return 0 = success, negative = error
    return 0;
}

// __exit - บอก kernel ว่าฟังก์ชันนี้ใช้แค่ตอน cleanup
static void __exit hello_exit(void)
{
    printk(KERN_INFO "Goodbye, Kernel World!\n");
}

// ลงทะเบียน init และ exit functions
module_init(hello_init);
module_exit(hello_exit);
```

### 1.2 MODULE_LICENSE ที่สำคัญ

```c
// License options ที่ kernel รองรับ:
MODULE_LICENSE("GPL");           // GNU General Public License v2
MODULE_LICENSE("GPL v2");        // เหมือนกัน
MODULE_LICENSE("GPL and additional rights");
MODULE_LICENSE("Dual BSD/GPL");  // เลือกได้ระหว่าง BSD หรือ GPL
MODULE_LICENSE("Dual MIT/GPL");
MODULE_LICENSE("Proprietary");   // ปิด taint flag - บาง kernel symbol ไม่สามารถใช้ได้

// หมายเหตุ: ถ้าใช้ "Proprietary" kernel จะแสดง warning:
// "module license 'Proprietary' taints kernel"
// และ symbol บางตัว เช่น kmalloc ที่ export เฉพาะ GPL จะใช้ไม่ได้
```

### 1.3 printk Log Levels

```c
#include <linux/kernel.h>

// Log levels (ตั้งแต่ critical ไปจนถึง debug)
printk(KERN_EMERG   "System is unusable\n");      // 0 - ฉุกเฉินสุด
printk(KERN_ALERT   "Action must be taken\n");    // 1
printk(KERN_CRIT    "Critical conditions\n");     // 2
printk(KERN_ERR     "Error conditions\n");        // 3
printk(KERN_WARNING "Warning conditions\n");      // 4
printk(KERN_NOTICE  "Normal but significant\n");  // 5
printk(KERN_INFO    "Informational\n");           // 6
printk(KERN_DEBUG   "Debug-level messages\n");    // 7

// Shortcut macros (kernel 3.x+)
pr_emerg("System is unusable\n");
pr_alert("Action must be taken\n");
pr_crit("Critical conditions\n");
pr_err("Error conditions: %d\n", error_code);
pr_warn("Warning: %s\n", message);
pr_notice("Notice: value=%d\n", val);
pr_info("Info: loaded module %s\n", module_name);
pr_debug("Debug: addr=%p size=%zu\n", ptr, size);

// pr_fmt macro - เพิ่ม prefix ให้ทุก message
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

// Dynamic debug (เปิด/ปิดได้ runtime)
pr_devel("Development message\n");  // เฉพาะ DEBUG build
```

---

## 2. Module Parameters

### 2.1 module_param พื้นฐาน

```c
// module_params.c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/moduleparam.h>

// ประกาศตัวแปร
static int count = 1;
static char *name = "default";
static int ports[4];
static int num_ports = 0;

// module_param(variable, type, permissions)
// permissions: 0 = ไม่ expose ใน /sys/module
//              S_IRUGO = อ่านได้ทุกคน
//              S_IRUGO|S_IWUSR = อ่านทุกคน, เขียนเฉพาะ root
module_param(count, int, S_IRUGO);
MODULE_PARM_DESC(count, "Number of times to print message");

module_param(name, charp, S_IRUGO | S_IWUSR);
MODULE_PARM_DESC(name, "Name to include in messages");

// Array parameter
module_param_array(ports, int, &num_ports, S_IRUGO);
MODULE_PARM_DESC(ports, "List of port numbers (max 4)");

static int __init params_init(void)
{
    int i;
    
    pr_info("count=%d, name=%s\n", count, name);
    
    for (i = 0; i < num_ports; i++) {
        pr_info("port[%d] = %d\n", i, ports[i]);
    }
    
    return 0;
}

static void __exit params_exit(void)
{
    pr_info("Module unloaded\n");
}

module_init(params_init);
module_exit(params_exit);

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Module parameters example");
```

### 2.2 การใช้งาน module_param

```bash
# โหลด module พร้อม parameters
sudo insmod module_params.ko count=5 name="hello" ports=80,443,8080

# ดู parameters ใน sysfs
ls /sys/module/module_params/parameters/
cat /sys/module/module_params/parameters/count

# เปลี่ยน parameter runtime (ถ้ามี write permission)
echo 10 > /sys/module/module_params/parameters/count
```

### 2.3 Types ที่ module_param รองรับ

```c
// Supported types:
module_param(var, bool,   perm);  // bool: true/false, y/n, 1/0
module_param(var, int,    perm);  // int
module_param(var, uint,   perm);  // unsigned int
module_param(var, long,   perm);  // long
module_param(var, ulong,  perm);  // unsigned long
module_param(var, short,  perm);  // short
module_param(var, ushort, perm);  // unsigned short
module_param(var, charp,  perm);  // char* (string)
module_param(var, byte,   perm);  // unsigned char
```

---

## 3. Kernel Memory Management

### 3.1 kmalloc และ kfree

```c
#include <linux/slab.h>     // kmalloc, kfree, kzalloc
#include <linux/gfp.h>      // GFP flags

// kmalloc - allocate physically contiguous memory
// size: จำนวน bytes
// flags: บอก kernel ว่าจะ allocate memory ยังไง
void *ptr = kmalloc(size, flags);

// GFP flags ที่สำคัญ:
// GFP_KERNEL   - allocation ปกติ, สามารถ sleep ได้ (ใช้ใน process context)
// GFP_ATOMIC   - ไม่สามารถ sleep ได้ (ใช้ใน interrupt context)
// GFP_DMA      - allocate ใน DMA-accessible memory (24-bit address)
// GFP_DMA32    - allocate ใน 32-bit DMA memory
// GFP_NOWAIT   - เหมือน GFP_ATOMIC แต่ไม่ทำ emergency pool
// GFP_NOIO     - ไม่ start I/O เพื่อ free memory
// GFP_NOFS     - ไม่ใช้ filesystem ขณะ allocate
// __GFP_ZERO   - zero-fill memory ที่ allocate

// ตัวอย่างการใช้งาน:
struct my_data {
    int id;
    char name[64];
    void *buffer;
};

static struct my_data *data;

static int allocate_example(void)
{
    // Allocate struct
    data = kmalloc(sizeof(struct my_data), GFP_KERNEL);
    if (!data) {
        pr_err("Failed to allocate my_data\n");
        return -ENOMEM;
    }
    
    // Zero-fill (เหมือน calloc)
    data = kzalloc(sizeof(struct my_data), GFP_KERNEL);
    
    // Allocate buffer
    data->buffer = kmalloc(1024, GFP_KERNEL);
    if (!data->buffer) {
        pr_err("Failed to allocate buffer\n");
        kfree(data);    // Free ที่ allocate ไปแล้ว
        return -ENOMEM;
    }
    
    data->id = 42;
    strncpy(data->name, "test", sizeof(data->name) - 1);
    
    return 0;
}

static void free_example(void)
{
    if (data) {
        kfree(data->buffer);   // Free buffer ก่อน
        kfree(data);           // Free struct
        data = NULL;           // ป้องกัน use-after-free
    }
}

// krealloc - resize allocation
ptr = krealloc(ptr, new_size, GFP_KERNEL);

// kcalloc - allocate array (n * size), zeroed
ptr = kcalloc(n, size, GFP_KERNEL);

// kmalloc_array - allocate array (overflow-safe)
ptr = kmalloc_array(n, size, GFP_KERNEL);
```

### 3.2 vmalloc และ vfree

```c
#include <linux/vmalloc.h>

// vmalloc - allocate virtually contiguous memory
// ใช้เมื่อต้องการ memory ขนาดใหญ่ที่ไม่ต้องการ physically contiguous
// เหมาะสำหรับ large buffers (> 128KB) แต่ช้ากว่า kmalloc
void *ptr = vmalloc(size);
if (!ptr) {
    pr_err("vmalloc failed for size %zu\n", size);
    return -ENOMEM;
}

// Free vmalloc memory
vfree(ptr);

// vzalloc - zeroed version
ptr = vzalloc(size);

// ข้อแตกต่างระหว่าง kmalloc vs vmalloc:
// kmalloc:
//   - physically AND virtually contiguous
//   - เหมาะสำหรับ DMA operations
//   - จำกัดขนาด (usually ~4MB)
//   - เร็วกว่า
// vmalloc:
//   - virtually contiguous เท่านั้น
//   - สำหรับ large allocations
//   - ไม่เหมาะสำหรับ DMA
//   - ช้ากว่า (ต้องทำ page table mapping)

// ตัวอย่างการใช้กับ large buffer
static void *large_buffer;
static const size_t BUFFER_SIZE = 16 * 1024 * 1024;  // 16MB

static int init_large_buffer(void)
{
    large_buffer = vmalloc(BUFFER_SIZE);
    if (!large_buffer)
        return -ENOMEM;
    
    memset(large_buffer, 0, BUFFER_SIZE);
    pr_info("Allocated %zu MB buffer at %p\n", 
            BUFFER_SIZE / (1024*1024), large_buffer);
    return 0;
}
```

### 3.3 Slab Allocator (kmem_cache)

```c
#include <linux/slab.h>

// kmem_cache ใช้สำหรับ allocate objects ที่มีขนาดเท่ากันบ่อยๆ
// มีประสิทธิภาพสูงกว่า kmalloc สำหรับ frequent allocations

struct my_object {
    int id;
    char name[32];
    struct list_head list;
};

static struct kmem_cache *my_cache;

// สร้าง cache
// kmem_cache_create(name, size, align, flags, ctor)
// ctor: constructor ที่เรียกตอน object ถูก allocate (อาจเป็น NULL)
static int init_cache(void)
{
    my_cache = kmem_cache_create(
        "my_object_cache",           // ชื่อ (ดูได้จาก /proc/slabinfo)
        sizeof(struct my_object),    // object size
        0,                           // alignment (0 = ใช้ default)
        SLAB_HWCACHE_ALIGN,         // flags
        NULL                         // constructor
    );
    
    if (!my_cache) {
        pr_err("Failed to create slab cache\n");
        return -ENOMEM;
    }
    
    pr_info("Created slab cache: %s\n", 
            kmem_cache_name(my_cache));
    return 0;
}

// Allocate object จาก cache
static struct my_object *alloc_object(int id, const char *name)
{
    struct my_object *obj;
    
    obj = kmem_cache_alloc(my_cache, GFP_KERNEL);
    if (!obj)
        return NULL;
    
    obj->id = id;
    strncpy(obj->name, name, sizeof(obj->name) - 1);
    INIT_LIST_HEAD(&obj->list);
    
    return obj;
}

// Free object กลับไปที่ cache
static void free_object(struct my_object *obj)
{
    kmem_cache_free(my_cache, obj);
}

// ทำลาย cache (ต้อง free objects ทั้งหมดก่อน)
static void destroy_cache(void)
{
    kmem_cache_destroy(my_cache);
    my_cache = NULL;
}

// Slab flags:
// SLAB_HWCACHE_ALIGN  - align to CPU cache line
// SLAB_POISON         - fill with magic bytes (debug)
// SLAB_RED_ZONE       - เพิ่ม red zones (debug)
// SLAB_ACCOUNT        - account memory to kmemcg
// SLAB_PANIC          - panic if allocation fails
```

---

## 4. Kernel vs User Space ข้อจำกัด

### 4.1 ไม่มี libc

```c
// SALAH - ห้ามใช้ใน kernel
#include <stdio.h>   // ไม่มี
#include <stdlib.h>  // ไม่มี
printf("Hello\n");   // ไม่มี - ใช้ printk แทน
malloc(100);         // ไม่มี - ใช้ kmalloc แทน
free(ptr);           // ไม่มี - ใช้ kfree แทน
memset/memcpy        // มีใน kernel แต่ include ต่างกัน

//ถูก - kernel equivalents
#include <linux/kernel.h>
#include <linux/string.h>   // strlen, strcpy, memset, memcpy
#include <linux/slab.h>     // kmalloc, kfree

printk(KERN_INFO "Hello\n");
kmalloc(100, GFP_KERNEL);
kfree(ptr);

// String functions ใน kernel
size_t kernel_strlen(const char *s)  { return strlen(s); }
char  *kernel_strcpy (char *d, const char *s) { return strcpy(d, s); }
int    kernel_strcmp (const char *a, const char *b) { return strcmp(a, b); }
void  *kernel_memset (void *s, int c, size_t n) { return memset(s, c, n); }
void  *kernel_memcpy (void *d, const void *s, size_t n) { return memcpy(d, s, n); }

// Kernel ยังมี safe string functions
strlcpy(dst, src, size);   // safe string copy
strlcat(dst, src, size);   // safe string concat
snprintf(buf, size, fmt, ...);  // มีใน kernel ด้วย

// User/Kernel space data transfer
#include <linux/uaccess.h>
copy_to_user(user_ptr, kernel_ptr, size);
copy_from_user(kernel_ptr, user_ptr, size);
put_user(value, user_ptr);
get_user(value, user_ptr);
```

### 4.2 No Floating Point

```c
// ห้ามใช้ floating point ใน kernel โดยตรง
// เพราะ FPU state ไม่ถูก save ใน interrupt context

// SALAH
float result = 3.14f * radius * radius;
double pi = 3.14159265358979;

// ถ้าจำเป็นต้องใช้ ต้อง save/restore FPU state
#include <asm/fpu/api.h>

static void kernel_fpu_example(void)
{
    kernel_fpu_begin();  // save FPU state
    
    // ตอนนี้ใช้ FPU ได้ แต่ต้องไม่ sleep หรือ switch context
    float x = 1.5f;
    float y = x * x;
    (void)y;
    
    kernel_fpu_end();    // restore FPU state
}

// ทางเลือกที่ดีกว่า: ใช้ integer arithmetic
// Fixed-point arithmetic: ใช้ scale factor
#define SCALE 1000000  // 6 decimal places

static long fixed_multiply(long a, long b)
{
    return (a * b) / SCALE;
}

// ตัวอย่าง: คำนวณ pi * r^2 ด้วย fixed-point
static long circle_area(long radius)
{
    long pi = 3141592;  // pi * 1000000
    long r2 = radius * radius;
    return (pi * r2) / SCALE;
}
```

### 4.3 Stack Size จำกัด

```c
// Kernel stack มีขนาดเล็กมาก (ปกติ 8KB หรือ 16KB)
// ห้าม allocate arrays ขนาดใหญ่บน stack

// SALAH - stack overflow!
void bad_function(void)
{
    char huge_buffer[1024 * 1024];  // 1MB บน stack = crash!
    int big_array[10000];           // 40KB บน stack = อันตราย
}

// ถูก - allocate จาก heap แทน
void good_function(void)
{
    char *buffer = kmalloc(1024 * 1024, GFP_KERNEL);
    if (!buffer) {
        pr_err("Allocation failed\n");
        return;
    }
    
    // ใช้ buffer
    
    kfree(buffer);
}

// ตรวจสอบ stack usage ด้วย CONFIG_DEBUG_STACKOVERFLOW
// และ/หรือ CONFIG_KASAN

// สำหรับ function ที่เรียกจาก interrupt context
// ต้องระวังมากขึ้น เพราะ IRQ stack มักจะเล็กกว่า
irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    // ห้ามใช้ large local variables ที่นี่
    // ใช้ kmalloc ก็ไม่ได้ใน IRQ (ต้องใช้ GFP_ATOMIC)
    struct my_device *dev = dev_id;
    
    // ทำงานที่รวดเร็ว แล้วส่งไปให้ workqueue ทำงานหนัก
    schedule_work(&dev->work);
    
    return IRQ_HANDLED;
}
```

---

## 5. Kernel Assembly via GCC Inline ASM

### 5.1 asm volatile พื้นฐาน

```c
// GCC inline assembly ใน kernel ใช้สำหรับ:
// 1. Operations ที่ต้องการ specific CPU instructions
// 2. Performance-critical code
// 3. Hardware-specific operations

// Basic syntax:
// asm volatile (
//     "assembly instructions"
//     : output operands
//     : input operands
//     : clobbers
// );

// ตัวอย่าง: อ่าน CPU cycle counter (TSC)
static inline u64 read_tsc(void)
{
    u32 lo, hi;
    
    asm volatile(
        "rdtsc"              // instruction
        : "=a"(lo),          // output: EAX -> lo
          "=d"(hi)           // output: EDX -> hi
        :                    // no inputs
        :                    // no clobbers (EAX, EDX ระบุใน outputs แล้ว)
    );
    
    return ((u64)hi << 32) | lo;
}

// ตัวอย่าง: cpuid instruction
static inline void cpuid(u32 leaf, u32 *eax, u32 *ebx, u32 *ecx, u32 *edx)
{
    asm volatile(
        "cpuid"
        : "=a"(*eax),    // output operands
          "=b"(*ebx),
          "=c"(*ecx),
          "=d"(*edx)
        : "a"(leaf)      // input: leaf -> EAX
        :                // no extra clobbers
    );
}

// ตัวอย่าง: memory barrier
static inline void memory_barrier(void)
{
    asm volatile(
        "mfence"         // Memory Fence - เป็น full memory barrier
        :                // no outputs
        :                // no inputs
        : "memory"       // clobber: บอก compiler ว่า memory อาจเปลี่ยน
    );
}

// Compiler barrier (ไม่ emit CPU instruction)
static inline void compiler_barrier(void)
{
    asm volatile("" ::: "memory");
}
```

### 5.2 Constraints ที่สำคัญ

```c
// Output constraints:
// "=r"  - output ไปที่ register ใดก็ได้
// "=a"  - output ไปที่ EAX/RAX
// "=b"  - output ไปที่ EBX/RBX
// "=c"  - output ไปที่ ECX/RCX
// "=d"  - output ไปที่ EDX/RDX
// "=m"  - output ไปที่ memory
// "=S"  - output ไปที่ ESI/RSI
// "=D"  - output ไปที่ EDI/RDI

// Input constraints:
// "r"   - input จาก register ใดก็ได้
// "a"   - input ที่ EAX/RAX
// "i"   - immediate constant
// "m"   - memory operand
// "0"   - เหมือน operand 0 (read-modify-write)

// Clobbers:
// "memory" - บอก compiler ว่า memory อาจเปลี่ยน
// "cc"     - condition codes (flags register) อาจเปลี่ยน
// "%eax"   - register ที่ถูก clobber

// ตัวอย่าง: atomic increment
static inline int atomic_add_return(int i, atomic_t *v)
{
    int result;
    
    asm volatile(
        LOCK_PREFIX         // lock prefix สำหรับ SMP
        "xaddl %0, %1"      // xadd: exchange and add
        : "=r"(result),     // output: result = old value
          "+m"(v->counter)  // output/input: memory operand
        : "0"(i)            // input: i ไปที่ operand 0 (result)
        : "memory", "cc"    // clobbers
    );
    
    return result + i;  // return new value
}

// ตัวอย่าง: clflush (cache line flush)
static inline void clflush(volatile void *addr)
{
    asm volatile(
        "clflush %0"
        :
        : "m"(*(volatile char *)addr)
        : "memory"
    );
}

// ตัวอย่าง: pause instruction (สำหรับ spin loops)
static inline void cpu_relax(void)
{
    asm volatile(
        "pause"
        ::: "memory"
    );
}

// ARM64 equivalents
#ifdef CONFIG_ARM64
static inline void arm64_isb(void)
{
    asm volatile("isb" ::: "memory");
}

static inline void arm64_dsb(void)
{
    asm volatile("dsb sy" ::: "memory");
}
#endif
```

### 5.3 Kernel-provided Wrappers

```c
// Kernel มี wrappers สำหรับ common assembly operations

// Bit operations
#include <asm/bitops.h>

void set_bit(long nr, volatile unsigned long *addr);
void clear_bit(long nr, volatile unsigned long *addr);
void change_bit(long nr, volatile unsigned long *addr);
int  test_bit(long nr, const volatile unsigned long *addr);
int  test_and_set_bit(long nr, volatile unsigned long *addr);
int  test_and_clear_bit(long nr, volatile unsigned long *addr);

// Memory barriers
#include <asm/barrier.h>

mb();   // full memory barrier
rmb();  // read memory barrier
wmb();  // write memory barrier
smp_mb();  // SMP memory barrier
smp_rmb();
smp_wmb();

// I/O operations
#include <asm/io.h>

inb(port);    // read byte from I/O port
outb(val, port);  // write byte to I/O port
inw(port);    // read word
outw(val, port);
inl(port);    // read long
outl(val, port);
```

---

## 6. Memory-Mapped I/O (MMIO)

### 6.1 ioremap และ iounmap

```c
#include <linux/io.h>

// ioremap - map physical address ไปที่ virtual address
// ใช้สำหรับเข้าถึง hardware registers ที่ map ใน physical memory
void __iomem *ioremap(phys_addr_t offset, size_t size);

// iounmap - unmap ที่ map ไว้
void iounmap(void __iomem *addr);

// ตัวอย่าง: access hardware register
#define MY_DEVICE_BASE  0xFE000000  // Physical base address
#define MY_DEVICE_SIZE  0x10000     // 64KB region

static void __iomem *dev_base;

static int init_mmio(void)
{
    // Request memory region (optional แต่ recommended)
    if (!request_mem_region(MY_DEVICE_BASE, MY_DEVICE_SIZE, "my_device")) {
        pr_err("Cannot request memory region\n");
        return -EBUSY;
    }
    
    // Map physical to virtual
    dev_base = ioremap(MY_DEVICE_BASE, MY_DEVICE_SIZE);
    if (!dev_base) {
        pr_err("Cannot ioremap device memory\n");
        release_mem_region(MY_DEVICE_BASE, MY_DEVICE_SIZE);
        return -ENOMEM;
    }
    
    pr_info("Device mapped at virtual address %p\n", dev_base);
    return 0;
}

static void cleanup_mmio(void)
{
    if (dev_base) {
        iounmap(dev_base);
        dev_base = NULL;
    }
    release_mem_region(MY_DEVICE_BASE, MY_DEVICE_SIZE);
}
```

### 6.2 readb/writeb/readl/writel

```c
// MMIO access functions - ต้องใช้แทน direct pointer dereference
// เพื่อให้ compiler ไม่ optimize out และ handle endianness ถูกต้อง

// Read functions:
u8  readb(const volatile void __iomem *addr);   // read 8-bit
u16 readw(const volatile void __iomem *addr);   // read 16-bit
u32 readl(const volatile void __iomem *addr);   // read 32-bit
u64 readq(const volatile void __iomem *addr);   // read 64-bit (64-bit arch)

// Write functions:
void writeb(u8  val, volatile void __iomem *addr);  // write 8-bit
void writew(u16 val, volatile void __iomem *addr);  // write 16-bit
void writel(u32 val, volatile void __iomem *addr);  // write 32-bit
void writeq(u64 val, volatile void __iomem *addr);  // write 64-bit

// ตัวอย่าง: hardware register layout
#define REG_CONTROL   0x00   // Control register
#define REG_STATUS    0x04   // Status register
#define REG_DATA      0x08   // Data register
#define REG_INT_MASK  0x0C   // Interrupt mask

// Control bits
#define CTRL_ENABLE   BIT(0)
#define CTRL_RESET    BIT(1)
#define CTRL_INT_EN   BIT(2)

// Status bits
#define STS_READY     BIT(0)
#define STS_ERROR     BIT(1)
#define STS_DATA_AVAIL BIT(2)

static void init_device(void __iomem *base)
{
    u32 control, status;
    
    // Reset device
    writel(CTRL_RESET, base + REG_CONTROL);
    
    // Wait for reset to complete
    int timeout = 1000;
    do {
        status = readl(base + REG_STATUS);
        if (!(status & CTRL_RESET))
            break;
        udelay(1);  // 1 microsecond delay
    } while (--timeout > 0);
    
    if (timeout == 0) {
        pr_err("Device reset timeout\n");
        return;
    }
    
    // Enable device with interrupts
    control = CTRL_ENABLE | CTRL_INT_EN;
    writel(control, base + REG_CONTROL);
    
    pr_info("Device initialized, status=0x%08x\n", 
            readl(base + REG_STATUS));
}

static u32 read_device_data(void __iomem *base)
{
    u32 status = readl(base + REG_STATUS);
    
    if (!(status & STS_DATA_AVAIL)) {
        pr_warn("No data available\n");
        return 0;
    }
    
    return readl(base + REG_DATA);
}

// ioread/iowrite - portable versions
ioread8(addr);
ioread16(addr);
ioread32(addr);
iowrite8(val, addr);
iowrite16(val, addr);
iowrite32(val, addr);

// Big-endian MMIO
readw_be(addr);
readl_be(addr);
writew_be(val, addr);
writel_be(val, addr);
```

---

## 7. IRQ Handling (Interrupt Request)

### 7.1 request_irq และ free_irq

```c
#include <linux/interrupt.h>

// request_irq - ลงทะเบียน interrupt handler
// int request_irq(unsigned int irq,
//                 irq_handler_t handler,
//                 unsigned long flags,
//                 const char *name,
//                 void *dev);

// IRQF flags:
// IRQF_SHARED   - interrupt สามารถ share กับ device อื่นได้
// IRQF_TRIGGER_RISING   - trigger on rising edge
// IRQF_TRIGGER_FALLING  - trigger on falling edge
// IRQF_TRIGGER_HIGH     - level triggered, high
// IRQF_TRIGGER_LOW      - level triggered, low
// IRQF_ONESHOT  - disable IRQ หลัง handler return, re-enable หลัง thread runs
// IRQF_NO_THREAD - ห้าม thread interrupt

// Interrupt handler type:
// typedef irqreturn_t (*irq_handler_t)(int, void *);

// Return values:
// IRQ_NONE     - interrupt ไม่ได้มาจาก device ของเรา
// IRQ_HANDLED  - interrupt จัดการเสร็จแล้ว
// IRQ_WAKE_THREAD - wake up interrupt handler thread

struct my_device {
    void __iomem *base;
    int irq;
    spinlock_t lock;
    atomic_t irq_count;
    struct work_struct work;
};

// Interrupt handler - ต้องทำงานเร็ว!
static irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    struct my_device *dev = (struct my_device *)dev_id;
    u32 status;
    
    // อ่าน interrupt status
    status = readl(dev->base + REG_STATUS);
    
    // ตรวจสอบว่า interrupt มาจาก device ของเรา
    if (!(status & STS_INTERRUPT)) {
        return IRQ_NONE;  // ไม่ใช่ interrupt ของเรา
    }
    
    // Clear interrupt flag
    writel(STS_INTERRUPT, dev->base + REG_STATUS);
    
    // นับ interrupt count
    atomic_inc(&dev->irq_count);
    
    // Schedule work for heavy processing
    // ห้ามทำงานหนักใน interrupt handler!
    schedule_work(&dev->work);
    
    return IRQ_HANDLED;
}

// Work handler - ทำงานใน process context
static void my_work_handler(struct work_struct *work)
{
    struct my_device *dev = container_of(work, struct my_device, work);
    
    // ทำงานหนักที่นี่ได้
    pr_info("Processing IRQ, count=%d\n", 
            atomic_read(&dev->irq_count));
}

static int setup_irq(struct my_device *dev, int irq_num)
{
    int ret;
    
    dev->irq = irq_num;
    INIT_WORK(&dev->work, my_work_handler);
    
    // Request interrupt
    ret = request_irq(
        dev->irq,           // IRQ number
        my_irq_handler,     // handler function
        IRQF_SHARED,        // flags
        "my_device",        // name ใน /proc/interrupts
        dev                 // dev_id (ส่งกลับมาใน handler)
    );
    
    if (ret) {
        pr_err("Failed to request IRQ %d: %d\n", irq_num, ret);
        return ret;
    }
    
    pr_info("IRQ %d registered successfully\n", irq_num);
    return 0;
}

static void release_irq(struct my_device *dev)
{
    // free_irq ต้องใช้ dev_id เดียวกับที่ request
    free_irq(dev->irq, dev);
    
    // Cancel any pending work
    cancel_work_sync(&dev->work);
}
```

### 7.2 Threaded IRQ

```c
// request_threaded_irq - IRQ handler ที่มี threaded portion
// เหมาะสำหรับ slow interrupt processing

static irqreturn_t my_fast_handler(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;
    u32 status;
    
    status = readl(dev->base + REG_STATUS);
    
    if (!(status & STS_INTERRUPT))
        return IRQ_NONE;
    
    // Acknowledge interrupt quickly
    writel(STS_INTERRUPT, dev->base + REG_STATUS);
    
    // Tell kernel to run threaded handler
    return IRQ_WAKE_THREAD;
}

static irqreturn_t my_threaded_handler(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;
    
    // ทำงานหนักได้ที่นี่ (runs in kernel thread context)
    // สามารถ sleep ได้
    
    u32 data = readl(dev->base + REG_DATA);
    pr_info("Data from device: 0x%08x\n", data);
    
    return IRQ_HANDLED;
}

static int setup_threaded_irq(struct my_device *dev, int irq_num)
{
    return request_threaded_irq(
        irq_num,
        my_fast_handler,     // fast handler (top half)
        my_threaded_handler, // threaded handler (bottom half)
        IRQF_SHARED,
        "my_device_threaded",
        dev
    );
}
```

---

## 8. Spinlocks

### 8.1 spin_lock และ spin_unlock

```c
#include <linux/spinlock.h>

// Spinlock - busy-wait lock สำหรับ short critical sections
// เหมาะสำหรับ:
// 1. Code ที่ทำงานใน interrupt context
// 2. Critical sections ที่สั้นมาก
// ห้าม sleep ขณะถือ spinlock!

spinlock_t my_lock;

// Initialize spinlock
spin_lock_init(&my_lock);

// หรือ initialize ตอน declare (static)
static DEFINE_SPINLOCK(my_static_lock);

// Basic lock/unlock (disable preemption)
spin_lock(&my_lock);
// critical section - ห้าม sleep!
spin_unlock(&my_lock);

// ตัวอย่าง: protect shared data structure
struct my_list {
    struct list_head head;
    spinlock_t lock;
    int count;
};

static struct my_list global_list;

static void init_list(struct my_list *list)
{
    INIT_LIST_HEAD(&list->head);
    spin_lock_init(&list->lock);
    list->count = 0;
}

static void add_item(struct my_list *list, struct my_item *item)
{
    spin_lock(&list->lock);
    list_add_tail(&item->node, &list->head);
    list->count++;
    spin_unlock(&list->lock);
}

static void remove_item(struct my_list *list, struct my_item *item)
{
    spin_lock(&list->lock);
    list_del(&item->node);
    list->count--;
    spin_unlock(&list->lock);
}
```

### 8.2 spin_lock_irqsave

```c
// spin_lock_irqsave - disable interrupts และ lock
// ใช้เมื่อ: code ที่ใช้ lock อาจถูกเรียกจากทั้ง process context และ interrupt context

unsigned long flags;

// Save interrupt state และ disable interrupts, then lock
spin_lock_irqsave(&my_lock, flags);
// critical section - ปลอดภัยจาก interrupt
spin_unlock_irqrestore(&my_lock, flags);

// ตัวอย่าง: driver ที่ access shared data จาก IRQ และ process context
struct my_driver {
    spinlock_t data_lock;
    u8 rx_buffer[1024];
    int rx_head, rx_tail;
};

// เรียกจาก interrupt handler
static void irq_receive_byte(struct my_driver *drv, u8 byte)
{
    unsigned long flags;
    int next_head;
    
    spin_lock_irqsave(&drv->data_lock, flags);
    
    next_head = (drv->rx_head + 1) % sizeof(drv->rx_buffer);
    if (next_head != drv->rx_tail) {
        drv->rx_buffer[drv->rx_head] = byte;
        drv->rx_head = next_head;
    }
    
    spin_unlock_irqrestore(&drv->data_lock, flags);
}

// เรียกจาก read() system call (process context)
static int process_read_byte(struct my_driver *drv, u8 *byte)
{
    unsigned long flags;
    int ret = 0;
    
    spin_lock_irqsave(&drv->data_lock, flags);
    
    if (drv->rx_tail != drv->rx_head) {
        *byte = drv->rx_buffer[drv->rx_tail];
        drv->rx_tail = (drv->rx_tail + 1) % sizeof(drv->rx_buffer);
        ret = 1;
    }
    
    spin_unlock_irqrestore(&drv->data_lock, flags);
    return ret;
}

// Other spinlock variants:
spin_lock_irq(&lock);       // disable local IRQs, then lock
spin_unlock_irq(&lock);     // unlock, then enable local IRQs
spin_lock_bh(&lock);        // disable softirqs, then lock
spin_unlock_bh(&lock);      // unlock, then enable softirqs
spin_trylock(&lock);        // try to lock, return 0 if failed
```

---

## 9. Mutex

### 9.1 mutex_init/mutex_lock/mutex_unlock

```c
#include <linux/mutex.h>

// Mutex - sleeping lock สำหรับ process context เท่านั้น
// ข้อแตกต่างจาก spinlock:
// - สามารถ sleep ขณะรอ lock ได้
// - ไม่สามารถใช้ใน interrupt context
// - เหมาะสำหรับ long critical sections

struct mutex my_mutex;

// Initialize mutex
mutex_init(&my_mutex);

// หรือ static initialization
static DEFINE_MUTEX(my_static_mutex);

// Lock (may sleep)
mutex_lock(&my_mutex);
// critical section - สามารถ sleep ได้
mutex_unlock(&my_mutex);

// Lock interruptible (สามารถถูก interrupt โดย signal)
if (mutex_lock_interruptible(&my_mutex)) {
    return -ERESTARTSYS;
}
mutex_unlock(&my_mutex);

// Trylock (non-blocking)
if (mutex_trylock(&my_mutex)) {
    // got the lock
    mutex_unlock(&my_mutex);
}

// ตัวอย่าง: protect file operations
struct my_device {
    struct mutex lock;
    bool busy;
    char data[4096];
    size_t data_len;
};

static ssize_t my_device_write(struct my_device *dev,
                                const char *buf, size_t count)
{
    ssize_t ret;
    
    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;
    
    if (dev->busy) {
        ret = -EBUSY;
        goto out;
    }
    
    dev->busy = true;
    
    // Process the data
    if (count > sizeof(dev->data)) {
        ret = -EINVAL;
        dev->busy = false;
        goto out;
    }
    
    memcpy(dev->data, buf, count);
    dev->data_len = count;
    dev->busy = false;
    ret = count;
    
out:
    mutex_unlock(&dev->lock);
    return ret;
}
```

### 9.2 เปรียบเทียบ Synchronization Primitives

```c
// เมื่อไหร่ควรใช้อะไร:
//
// spinlock:
//   - interrupt context
//   - SMP protection ที่สั้นมาก (< 100 cycles)
//   - ห้าม sleep, ห้าม call blocking functions
//
// mutex:
//   - process context เท่านั้น
//   - critical sections ที่ยาวหรืออาจ sleep
//   - file operations, memory allocation
//
// rwlock/rwsem:
//   - read-heavy workloads
//   - หลาย readers พร้อมกัน, แต่ writer ต้อง exclusive
//
// seqlock:
//   - read-heavy, rare writes
//   - readers ไม่บล็อก writers

#include <linux/rwlock.h>
#include <linux/rwsem.h>
#include <linux/seqlock.h>

// Read-write spinlock
rwlock_t rw_lock;
rwlock_init(&rw_lock);

read_lock(&rw_lock);
// multiple readers can hold simultaneously
read_unlock(&rw_lock);

write_lock(&rw_lock);
// exclusive access
write_unlock(&rw_lock);

// Read-write semaphore (สำหรับ process context)
struct rw_semaphore rwsem;
init_rwsem(&rwsem);

down_read(&rwsem);
up_read(&rwsem);

down_write(&rwsem);
up_write(&rwsem);

// Seqlock
seqlock_t seqlock;
seqlock_init(&seqlock);

// Writer
write_seqlock(&seqlock);
// update data
write_sequnlock(&seqlock);

// Reader (retry ถ้า writer ขัด)
unsigned seq;
do {
    seq = read_seqbegin(&seqlock);
    // read data
} while (read_seqretry(&seqlock, seq));
```

---

## 10. Character Device Driver

### 10.1 register_chrdev และ file_operations

```c
// char_device.c - Complete character device driver

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/fs.h>           // file_operations, register_chrdev
#include <linux/cdev.h>         // cdev struct
#include <linux/device.h>       // class_create, device_create
#include <linux/uaccess.h>      // copy_to_user, copy_from_user
#include <linux/mutex.h>
#include <linux/slab.h>

#define DEVICE_NAME     "mychar"
#define CLASS_NAME      "myclass"
#define BUFFER_SIZE     1024

// Device data structure
struct mychar_dev {
    struct cdev cdev;
    char buffer[BUFFER_SIZE];
    size_t buf_len;
    struct mutex lock;
    int open_count;
};

static int major_number;
static struct class *mychar_class;
static struct device *mychar_device;
static struct mychar_dev *mychar;

// open() - เรียกเมื่อ user เปิด /dev/mychar
static int mychar_open(struct inode *inode, struct file *filp)
{
    struct mychar_dev *dev;
    
    // หา device struct จาก cdev
    dev = container_of(inode->i_cdev, struct mychar_dev, cdev);
    filp->private_data = dev;  // เก็บ pointer ไว้ใน file struct
    
    mutex_lock(&dev->lock);
    dev->open_count++;
    mutex_unlock(&dev->lock);
    
    pr_info("%s: opened (count=%d)\n", DEVICE_NAME, dev->open_count);
    return 0;
}

// release() - เรียกเมื่อ user ปิด /dev/mychar
static int mychar_release(struct inode *inode, struct file *filp)
{
    struct mychar_dev *dev = filp->private_data;
    
    mutex_lock(&dev->lock);
    dev->open_count--;
    mutex_unlock(&dev->lock);
    
    pr_info("%s: closed (count=%d)\n", DEVICE_NAME, dev->open_count);
    return 0;
}

// read() - user อ่านข้อมูลจาก device
static ssize_t mychar_read(struct file *filp, char __user *buf,
                            size_t count, loff_t *f_pos)
{
    struct mychar_dev *dev = filp->private_data;
    ssize_t ret;
    
    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;
    
    // ตรวจสอบว่ามีข้อมูลให้อ่านไหม
    if (*f_pos >= dev->buf_len) {
        ret = 0;  // EOF
        goto out;
    }
    
    // ปรับ count ให้ไม่เกิน buffer
    if (count > dev->buf_len - *f_pos)
        count = dev->buf_len - *f_pos;
    
    // Copy kernel buffer -> user space
    if (copy_to_user(buf, dev->buffer + *f_pos, count)) {
        ret = -EFAULT;
        goto out;
    }
    
    *f_pos += count;
    ret = count;
    
    pr_debug("%s: read %zu bytes\n", DEVICE_NAME, count);
    
out:
    mutex_unlock(&dev->lock);
    return ret;
}

// write() - user เขียนข้อมูลเข้า device
static ssize_t mychar_write(struct file *filp, const char __user *buf,
                             size_t count, loff_t *f_pos)
{
    struct mychar_dev *dev = filp->private_data;
    ssize_t ret;
    
    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;
    
    if (count > BUFFER_SIZE) {
        ret = -EINVAL;
        goto out;
    }
    
    // Copy user space -> kernel buffer
    if (copy_from_user(dev->buffer, buf, count)) {
        ret = -EFAULT;
        goto out;
    }
    
    dev->buf_len = count;
    *f_pos = 0;  // Reset position after write
    ret = count;
    
    pr_debug("%s: wrote %zu bytes\n", DEVICE_NAME, count);
    
out:
    mutex_unlock(&dev->lock);
    return ret;
}

// ioctl() commands
#define MYCHAR_IOC_MAGIC    'M'
#define MYCHAR_CLEAR        _IO(MYCHAR_IOC_MAGIC, 0)
#define MYCHAR_GET_SIZE     _IOR(MYCHAR_IOC_MAGIC, 1, int)
#define MYCHAR_SET_SIZE     _IOW(MYCHAR_IOC_MAGIC, 2, int)

// ioctl() - control device
static long mychar_ioctl(struct file *filp, unsigned int cmd,
                          unsigned long arg)
{
    struct mychar_dev *dev = filp->private_data;
    int ret = 0;
    int size;
    
    // Validate command
    if (_IOC_TYPE(cmd) != MYCHAR_IOC_MAGIC)
        return -ENOTTY;
    
    switch (cmd) {
    case MYCHAR_CLEAR:
        mutex_lock(&dev->lock);
        memset(dev->buffer, 0, BUFFER_SIZE);
        dev->buf_len = 0;
        mutex_unlock(&dev->lock);
        pr_info("%s: buffer cleared\n", DEVICE_NAME);
        break;
        
    case MYCHAR_GET_SIZE:
        size = dev->buf_len;
        if (copy_to_user((int __user *)arg, &size, sizeof(int)))
            return -EFAULT;
        break;
        
    case MYCHAR_SET_SIZE:
        if (copy_from_user(&size, (int __user *)arg, sizeof(int)))
            return -EFAULT;
        if (size < 0 || size > BUFFER_SIZE)
            return -EINVAL;
        mutex_lock(&dev->lock);
        dev->buf_len = size;
        mutex_unlock(&dev->lock);
        break;
        
    default:
        return -ENOTTY;
    }
    
    return ret;
}

// lseek() - seek ใน buffer
static loff_t mychar_llseek(struct file *filp, loff_t off, int whence)
{
    struct mychar_dev *dev = filp->private_data;
    loff_t newpos;
    
    switch (whence) {
    case SEEK_SET:
        newpos = off;
        break;
    case SEEK_CUR:
        newpos = filp->f_pos + off;
        break;
    case SEEK_END:
        newpos = dev->buf_len + off;
        break;
    default:
        return -EINVAL;
    }
    
    if (newpos < 0 || newpos > BUFFER_SIZE)
        return -EINVAL;
    
    filp->f_pos = newpos;
    return newpos;
}

// file_operations struct - จดทะเบียน operations ทั้งหมด
static const struct file_operations mychar_fops = {
    .owner          = THIS_MODULE,
    .open           = mychar_open,
    .release        = mychar_release,
    .read           = mychar_read,
    .write          = mychar_write,
    .unlocked_ioctl = mychar_ioctl,
    .llseek         = mychar_llseek,
};

static int __init mychar_init(void)
{
    int ret;
    dev_t dev_num;
    
    // Allocate device number dynamically
    ret = alloc_chrdev_region(&dev_num, 0, 1, DEVICE_NAME);
    if (ret < 0) {
        pr_err("%s: alloc_chrdev_region failed: %d\n", DEVICE_NAME, ret);
        return ret;
    }
    major_number = MAJOR(dev_num);
    
    // Allocate device structure
    mychar = kzalloc(sizeof(struct mychar_dev), GFP_KERNEL);
    if (!mychar) {
        ret = -ENOMEM;
        goto fail_alloc;
    }
    
    // Initialize mutex
    mutex_init(&mychar->lock);
    
    // Initialize and add cdev
    cdev_init(&mychar->cdev, &mychar_fops);
    mychar->cdev.owner = THIS_MODULE;
    ret = cdev_add(&mychar->cdev, dev_num, 1);
    if (ret) {
        pr_err("%s: cdev_add failed: %d\n", DEVICE_NAME, ret);
        goto fail_cdev;
    }
    
    // Create device class (creates /sys/class/myclass)
    mychar_class = class_create(THIS_MODULE, CLASS_NAME);
    if (IS_ERR(mychar_class)) {
        ret = PTR_ERR(mychar_class);
        pr_err("%s: class_create failed: %d\n", DEVICE_NAME, ret);
        goto fail_class;
    }
    
    // Create device file (creates /dev/mychar)
    mychar_device = device_create(mychar_class, NULL, dev_num, 
                                   NULL, DEVICE_NAME);
    if (IS_ERR(mychar_device)) {
        ret = PTR_ERR(mychar_device);
        pr_err("%s: device_create failed: %d\n", DEVICE_NAME, ret);
        goto fail_device;
    }
    
    pr_info("%s: registered with major number %d\n", DEVICE_NAME, major_number);
    pr_info("%s: device created at /dev/%s\n", DEVICE_NAME, DEVICE_NAME);
    
    return 0;

fail_device:
    class_destroy(mychar_class);
fail_class:
    cdev_del(&mychar->cdev);
fail_cdev:
    kfree(mychar);
fail_alloc:
    unregister_chrdev_region(dev_num, 1);
    return ret;
}

static void __exit mychar_exit(void)
{
    dev_t dev_num = MKDEV(major_number, 0);
    
    device_destroy(mychar_class, dev_num);
    class_destroy(mychar_class);
    cdev_del(&mychar->cdev);
    kfree(mychar);
    unregister_chrdev_region(dev_num, 1);
    
    pr_info("%s: unregistered\n", DEVICE_NAME);
}

module_init(mychar_init);
module_exit(mychar_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Developer");
MODULE_DESCRIPTION("Simple character device driver");
```

---

## 11. /proc Interface

### 11.1 proc_create และ seq_file

```c
#include <linux/proc_fs.h>    // proc_create, proc_remove
#include <linux/seq_file.h>   // seq_file, seq_printf

// /proc interface ใช้สำหรับ:
// 1. Expose kernel data ให้ userspace อ่านได้
// 2. ให้ userspace configure module parameters
// 3. Debug information

// ตัวอย่าง: สร้าง /proc/mymodule
#define PROC_NAME "mymodule"

// Data ที่ต้องการ expose ผ่าน /proc
static int show_count = 0;

// seq_show - เรียกสำหรับแต่ละ line ใน output
static int mymodule_proc_show(struct seq_file *m, void *v)
{
    seq_printf(m, "Module: %s\n", KBUILD_MODNAME);
    seq_printf(m, "Show count: %d\n", show_count++);
    seq_printf(m, "Jiffies: %lu\n", jiffies);
    seq_printf(m, "HZ: %d\n", HZ);
    seq_printf(m, "Page size: %lu KB\n", PAGE_SIZE / 1024);
    
    return 0;
}

// ใช้ single_open สำหรับ simple /proc files
static int mymodule_proc_open(struct inode *inode, struct file *file)
{
    return single_open(file, mymodule_proc_show, NULL);
}

static const struct proc_ops mymodule_proc_ops = {
    .proc_open    = mymodule_proc_open,
    .proc_read    = seq_read,       // standard seq_file read
    .proc_lseek   = seq_lseek,      // standard seek
    .proc_release = single_release, // release for single_open
};

static struct proc_dir_entry *proc_entry;

static int __init proc_example_init(void)
{
    proc_entry = proc_create(PROC_NAME, 0444, NULL, &mymodule_proc_ops);
    if (!proc_entry) {
        pr_err("Failed to create /proc/%s\n", PROC_NAME);
        return -ENOMEM;
    }
    
    pr_info("Created /proc/%s\n", PROC_NAME);
    return 0;
}

static void __exit proc_example_exit(void)
{
    proc_remove(proc_entry);
    pr_info("Removed /proc/%s\n", PROC_NAME);
}
```

### 11.2 seq_file สำหรับ Multi-line Output

```c
// สำหรับ output หลาย records ใช้ seq_file iterator interface

struct my_record {
    int id;
    char name[32];
    unsigned long value;
    struct list_head list;
};

static LIST_HEAD(records_list);
static DEFINE_MUTEX(records_lock);

// seq_start - เริ่ม iteration, return NULL เพื่อหยุด
static void *records_seq_start(struct seq_file *m, loff_t *pos)
{
    mutex_lock(&records_lock);
    return seq_list_start(&records_list, *pos);
}

// seq_next - ไปยัง record ถัดไป
static void *records_seq_next(struct seq_file *m, void *v, loff_t *pos)
{
    return seq_list_next(v, &records_list, pos);
}

// seq_stop - หยุด iteration
static void records_seq_stop(struct seq_file *m, void *v)
{
    mutex_unlock(&records_lock);
}

// seq_show - print current record
static int records_seq_show(struct seq_file *m, void *v)
{
    struct my_record *rec = list_entry(v, struct my_record, list);
    seq_printf(m, "%4d %-32s %lu\n", rec->id, rec->name, rec->value);
    return 0;
}

static const struct seq_operations records_seq_ops = {
    .start = records_seq_start,
    .next  = records_seq_next,
    .stop  = records_seq_stop,
    .show  = records_seq_show,
};

static int records_proc_open(struct inode *inode, struct file *file)
{
    // print header
    return seq_open(file, &records_seq_ops);
}

static const struct proc_ops records_proc_ops = {
    .proc_open    = records_proc_open,
    .proc_read    = seq_read,
    .proc_lseek   = seq_lseek,
    .proc_release = seq_release,
};
```

---

## 12. sysfs Interface

### 12.1 kobject และ sysfs_create_file

```c
#include <linux/kobject.h>
#include <linux/sysfs.h>

// sysfs - virtual filesystem ที่ /sys
// ใช้สำหรับ expose device/driver attributes ให้ userspace

// Simple sysfs attribute
struct kobject *my_kobj;
static int my_value = 42;

// show function - เรียกเมื่อ user อ่าน attribute
static ssize_t my_attr_show(struct kobject *kobj,
                             struct kobj_attribute *attr,
                             char *buf)
{
    return sprintf(buf, "%d\n", my_value);
}

// store function - เรียกเมื่อ user เขียน attribute
static ssize_t my_attr_store(struct kobject *kobj,
                              struct kobj_attribute *attr,
                              const char *buf, size_t count)
{
    int ret;
    
    ret = kstrtoint(buf, 10, &my_value);
    if (ret < 0)
        return ret;
    
    pr_info("my_value set to %d\n", my_value);
    return count;
}

// สร้าง attribute
static struct kobj_attribute my_attribute = __ATTR(
    my_value,            // attribute name -> /sys/.../my_value
    0664,                // permissions
    my_attr_show,        // read function
    my_attr_store        // write function
);

// Read-only attribute
static struct kobj_attribute ro_attribute = __ATTR_RO(my_value);

// Write-only attribute
static struct kobj_attribute wo_attribute = __ATTR_WO(my_value);

static int __init sysfs_example_init(void)
{
    int ret;
    
    // สร้าง kobject ใต้ /sys/kernel/
    my_kobj = kobject_create_and_add("my_module", kernel_kobj);
    if (!my_kobj)
        return -ENOMEM;
    
    // สร้าง sysfs file
    ret = sysfs_create_file(my_kobj, &my_attribute.attr);
    if (ret) {
        pr_err("Failed to create sysfs file\n");
        kobject_put(my_kobj);
        return ret;
    }
    
    pr_info("sysfs entry created at /sys/kernel/my_module/my_value\n");
    return 0;
}

static void __exit sysfs_example_exit(void)
{
    sysfs_remove_file(my_kobj, &my_attribute.attr);
    kobject_put(my_kobj);
    pr_info("sysfs entry removed\n");
}
```

### 12.2 Attribute Groups

```c
// สะดวกกว่าการสร้างทีละ file
static struct attribute *my_attrs[] = {
    &my_attribute.attr,
    &ro_attribute.attr,
    NULL,  // must be NULL-terminated
};

static struct attribute_group my_attr_group = {
    .name  = "settings",  // optional subdirectory name
    .attrs = my_attrs,
};

// สร้างทั้ง group
ret = sysfs_create_group(my_kobj, &my_attr_group);

// ลบทั้ง group
sysfs_remove_group(my_kobj, &my_attr_group);
```

---

## 13. DMA (Direct Memory Access)

### 13.1 dma_alloc_coherent

```c
#include <linux/dma-mapping.h>

// dma_alloc_coherent - allocate DMA-coherent memory
// Memory ที่ allocate จะ:
// 1. Accessible ทั้ง CPU และ DMA device
// 2. Cache coherent (ไม่ต้องทำ cache flush)
// 3. Physically contiguous

struct dma_example {
    struct device *dev;
    void *cpu_addr;         // Virtual address สำหรับ CPU
    dma_addr_t dma_addr;    // Physical/bus address สำหรับ DMA device
    size_t size;
};

static int allocate_dma_buffer(struct dma_example *ex, 
                                struct device *dev,
                                size_t size)
{
    ex->dev = dev;
    ex->size = size;
    
    // Allocate coherent DMA buffer
    ex->cpu_addr = dma_alloc_coherent(
        dev,
        size,
        &ex->dma_addr,   // returns bus address for device
        GFP_KERNEL
    );
    
    if (!ex->cpu_addr) {
        dev_err(dev, "Failed to allocate DMA buffer\n");
        return -ENOMEM;
    }
    
    dev_info(dev, "DMA buffer: cpu=%p, dma=0x%llx, size=%zu\n",
             ex->cpu_addr, (u64)ex->dma_addr, size);
    
    return 0;
}

static void free_dma_buffer(struct dma_example *ex)
{
    if (ex->cpu_addr) {
        dma_free_coherent(ex->dev, ex->size, 
                          ex->cpu_addr, ex->dma_addr);
        ex->cpu_addr = NULL;
    }
}
```

### 13.2 dma_map_single (Streaming DMA)

```c
// dma_map_single - map existing buffer สำหรับ single DMA transfer
// ใช้สำหรับ buffer ที่ allocate ด้วย kmalloc แล้ว map ชั่วคราว

static int dma_transfer_example(struct device *dev,
                                  void *buffer, size_t size,
                                  int direction)
{
    dma_addr_t dma_addr;
    
    // direction:
    // DMA_TO_DEVICE     - CPU -> device (write)
    // DMA_FROM_DEVICE   - device -> CPU (read)
    // DMA_BIDIRECTIONAL - both directions
    
    // Map buffer
    dma_addr = dma_map_single(dev, buffer, size, direction);
    
    if (dma_mapping_error(dev, dma_addr)) {
        dev_err(dev, "DMA mapping failed\n");
        return -EIO;
    }
    
    // ตอนนี้ dma_addr ส่งให้ hardware เพื่อทำ DMA transfer
    // ... program DMA controller ...
    
    // หลังจาก transfer เสร็จ (synchronize if DMA_FROM_DEVICE)
    dma_sync_single_for_cpu(dev, dma_addr, size, direction);
    
    // Unmap
    dma_unmap_single(dev, dma_addr, size, direction);
    
    return 0;
}

// dma_map_sg - scatter-gather DMA
// ใช้สำหรับ non-contiguous buffers
#include <linux/scatterlist.h>

struct scatterlist sg[4];
int i, nents;

sg_init_table(sg, 4);
for (i = 0; i < 4; i++) {
    sg_set_buf(&sg[i], buffers[i], sizes[i]);
}

nents = dma_map_sg(dev, sg, 4, DMA_TO_DEVICE);
if (!nents) {
    dev_err(dev, "dma_map_sg failed\n");
    return -EIO;
}

// ใช้ sg_dma_address/sg_dma_len สำหรับแต่ละ segment
for (i = 0; i < nents; i++) {
    dev_dbg(dev, "sg[%d]: addr=0x%llx, len=%u\n",
            i, sg_dma_address(&sg[i]), sg_dma_len(&sg[i]));
}

dma_unmap_sg(dev, sg, 4, DMA_TO_DEVICE);
```

---

## 14. Module Build System (Kbuild)

### 14.1 Kbuild Makefile

```makefile
# Makefile สำหรับ out-of-tree kernel module

# ชื่อ module (ไม่มี .ko extension)
obj-m += hello_module.o
obj-m += mychar.o

# ถ้า module ประกอบจากหลาย source files:
mychar-objs := char_device.o ioctl.o proc.o

# KDIR - path ไปยัง kernel source/build directory
# สามารถ override ได้จาก command line
KDIR ?= /lib/modules/$(shell uname -r)/build

# PWD ของ module source
PWD := $(shell pwd)

# Default target
all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

# Clean build artifacts
clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean

# Install module
install:
	$(MAKE) -C $(KDIR) M=$(PWD) modules_install
	depmod -a

# เปิด debug info
ccflags-y := -DDEBUG -g

# Cross-compile (สำหรับ ARM)
# make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-
```

### 14.2 การ Build และ Load Module

```bash
# Build module
make

# หรือระบุ kernel version
make KDIR=/lib/modules/5.15.0-generic/build

# ดู module info
modinfo hello_module.ko

# Load module
sudo insmod hello_module.ko

# Load พร้อม parameters
sudo insmod mymodule.ko count=5 name="test"

# ดู loaded modules
lsmod
lsmod | grep hello

# ดู module info ของ loaded module
cat /sys/module/hello_module/version
cat /sys/module/hello_module/description

# Unload module
sudo rmmod hello_module

# หรือใช้ modprobe (ดู dependencies ด้วย)
sudo modprobe hello_module count=5
sudo modprobe -r hello_module

# ดู kernel messages
dmesg | tail -20
dmesg | grep hello_module

# Follow kernel log
dmesg -w

# ดู module dependencies
modprobe --show-depends mymodule

# ติดตั้ง module ถาวร
sudo cp hello_module.ko /lib/modules/$(uname -r)/extra/
sudo depmod -a
echo "hello_module" | sudo tee -a /etc/modules
```

### 14.3 dmesg Debugging

```bash
# ดู kernel log ทั้งหมด
dmesg

# แสดง human-readable timestamp
dmesg -T

# แสดง log level
dmesg -x

# Clear kernel ring buffer
sudo dmesg -C

# Follow (เหมือน tail -f)
dmesg -w

# Filter by facility/level
dmesg --level=err,warn
dmesg --facility=kern

# ดู log จาก file (สำหรับ saved logs)
journalctl -k           # kernel messages จาก systemd journal
journalctl -k -b -1     # ของ boot ก่อนหน้า
journalctl -k --since "1 hour ago"
```

---

## 15. Complete Example: /dev/hello Character Device Driver

```c
// hello_device.c - Complete /dev/hello character device driver
// สาธิตการใช้งาน: open, read, write, ioctl, proc, sysfs

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>
#include <linux/mutex.h>
#include <linux/slab.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/kobject.h>
#include <linux/sysfs.h>
#include <linux/atomic.h>
#include <linux/string.h>

#define DEVICE_NAME     "hello"
#define CLASS_NAME      "hello_class"
#define BUFFER_SIZE     4096
#define HELLO_VERSION   "1.0.0"

// IOCTL commands
#define HELLO_IOC_MAGIC     'H'
#define HELLO_RESET         _IO(HELLO_IOC_MAGIC, 0)
#define HELLO_GET_LEN       _IOR(HELLO_IOC_MAGIC, 1, int)
#define HELLO_SET_MSG       _IOW(HELLO_IOC_MAGIC, 2, char[256])
#define HELLO_GET_STATS     _IOR(HELLO_IOC_MAGIC, 3, struct hello_stats)

struct hello_stats {
    unsigned long read_count;
    unsigned long write_count;
    unsigned long open_count;
    unsigned long byte_count;
};

// Main device structure
struct hello_device {
    // cdev
    struct cdev cdev;
    
    // Data buffer
    char *buffer;
    size_t buf_len;
    
    // Synchronization
    struct mutex lock;
    
    // Statistics
    atomic_long_t read_count;
    atomic_long_t write_count;
    atomic_long_t open_count;
    atomic_long_t byte_count;
    
    // sysfs
    struct kobject *kobj;
};

// Global device
static int major_num;
static struct class *hello_class;
static struct device *hello_dev_node;
static struct hello_device *hello_dev;

// proc entry
static struct proc_dir_entry *hello_proc;

// ===== file_operations =====

static int hello_open(struct inode *inode, struct file *filp)
{
    struct hello_device *dev = container_of(inode->i_cdev,
                                             struct hello_device, cdev);
    filp->private_data = dev;
    atomic_long_inc(&dev->open_count);
    
    pr_info(DEVICE_NAME ": opened by PID %d (%s)\n",
            current->pid, current->comm);
    return 0;
}

static int hello_release(struct inode *inode, struct file *filp)
{
    pr_info(DEVICE_NAME ": closed\n");
    return 0;
}

static ssize_t hello_read(struct file *filp, char __user *buf,
                           size_t count, loff_t *f_pos)
{
    struct hello_device *dev = filp->private_data;
    ssize_t ret = 0;
    size_t available;
    
    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;
    
    if (*f_pos >= dev->buf_len) {
        // EOF
        ret = 0;
        goto out;
    }
    
    available = dev->buf_len - *f_pos;
    if (count > available)
        count = available;
    
    if (copy_to_user(buf, dev->buffer + *f_pos, count)) {
        ret = -EFAULT;
        goto out;
    }
    
    *f_pos += count;
    ret = count;
    
    atomic_long_inc(&dev->read_count);
    atomic_long_add(count, &dev->byte_count);
    
out:
    mutex_unlock(&dev->lock);
    return ret;
}

static ssize_t hello_write(struct file *filp, const char __user *buf,
                            size_t count, loff_t *f_pos)
{
    struct hello_device *dev = filp->private_data;
    ssize_t ret;
    
    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;
    
    if (count > BUFFER_SIZE) {
        ret = -EINVAL;
        goto out;
    }
    
    // Clear buffer and write from beginning
    memset(dev->buffer, 0, BUFFER_SIZE);
    
    if (copy_from_user(dev->buffer, buf, count)) {
        ret = -EFAULT;
        goto out;
    }
    
    dev->buf_len = count;
    *f_pos = 0;
    ret = count;
    
    atomic_long_inc(&dev->write_count);
    atomic_long_add(count, &dev->byte_count);
    
    pr_info(DEVICE_NAME ": wrote %zu bytes: %.40s%s\n",
            count, dev->buffer,
            count > 40 ? "..." : "");
    
out:
    mutex_unlock(&dev->lock);
    return ret;
}

static long hello_ioctl(struct file *filp, unsigned int cmd,
                         unsigned long arg)
{
    struct hello_device *dev = filp->private_data;
    struct hello_stats stats;
    char msg[256];
    int len;
    int ret = 0;
    
    if (_IOC_TYPE(cmd) != HELLO_IOC_MAGIC)
        return -ENOTTY;
    
    switch (cmd) {
    case HELLO_RESET:
        mutex_lock(&dev->lock);
        memset(dev->buffer, 0, BUFFER_SIZE);
        dev->buf_len = 0;
        mutex_unlock(&dev->lock);
        pr_info(DEVICE_NAME ": reset\n");
        break;
        
    case HELLO_GET_LEN:
        len = (int)dev->buf_len;
        if (copy_to_user((int __user *)arg, &len, sizeof(int)))
            return -EFAULT;
        break;
        
    case HELLO_SET_MSG:
        if (copy_from_user(msg, (char __user *)arg, sizeof(msg)))
            return -EFAULT;
        msg[sizeof(msg) - 1] = '\0';
        
        mutex_lock(&dev->lock);
        len = snprintf(dev->buffer, BUFFER_SIZE,
                       "Hello, %s!\n", msg);
        dev->buf_len = len;
        mutex_unlock(&dev->lock);
        pr_info(DEVICE_NAME ": message set for '%s'\n", msg);
        break;
        
    case HELLO_GET_STATS:
        stats.read_count  = atomic_long_read(&dev->read_count);
        stats.write_count = atomic_long_read(&dev->write_count);
        stats.open_count  = atomic_long_read(&dev->open_count);
        stats.byte_count  = atomic_long_read(&dev->byte_count);
        if (copy_to_user((struct hello_stats __user *)arg,
                         &stats, sizeof(stats)))
            return -EFAULT;
        break;
        
    default:
        return -ENOTTY;
    }
    
    return ret;
}

static loff_t hello_llseek(struct file *filp, loff_t off, int whence)
{
    struct hello_device *dev = filp->private_data;
    loff_t newpos;
    
    switch (whence) {
    case SEEK_SET: newpos = off; break;
    case SEEK_CUR: newpos = filp->f_pos + off; break;
    case SEEK_END: newpos = dev->buf_len + off; break;
    default: return -EINVAL;
    }
    
    if (newpos < 0 || newpos > BUFFER_SIZE)
        return -EINVAL;
    
    return (filp->f_pos = newpos);
}

static const struct file_operations hello_fops = {
    .owner          = THIS_MODULE,
    .open           = hello_open,
    .release        = hello_release,
    .read           = hello_read,
    .write          = hello_write,
    .unlocked_ioctl = hello_ioctl,
    .llseek         = hello_llseek,
};

// ===== /proc/hello_info =====

static int hello_proc_show(struct seq_file *m, void *v)
{
    struct hello_device *dev = hello_dev;
    
    seq_printf(m, "=== /dev/hello device info ===\n");
    seq_printf(m, "Version    : %s\n", HELLO_VERSION);
    seq_printf(m, "Major      : %d\n", major_num);
    seq_printf(m, "Buffer used: %zu / %d bytes\n", 
               dev->buf_len, BUFFER_SIZE);
    seq_printf(m, "Opens      : %ld\n", 
               atomic_long_read(&dev->open_count));
    seq_printf(m, "Reads      : %ld\n", 
               atomic_long_read(&dev->read_count));
    seq_printf(m, "Writes     : %ld\n", 
               atomic_long_read(&dev->write_count));
    seq_printf(m, "Bytes xfr  : %ld\n", 
               atomic_long_read(&dev->byte_count));
    
    if (dev->buf_len > 0) {
        seq_printf(m, "Content    : %.*s", 
                   (int)min(dev->buf_len, (size_t)64),
                   dev->buffer);
        if (dev->buf_len > 64)
            seq_printf(m, "...");
        seq_printf(m, "\n");
    }
    
    return 0;
}

static int hello_proc_open(struct inode *inode, struct file *file)
{
    return single_open(file, hello_proc_show, NULL);
}

static const struct proc_ops hello_proc_ops = {
    .proc_open    = hello_proc_open,
    .proc_read    = seq_read,
    .proc_lseek   = seq_lseek,
    .proc_release = single_release,
};

// ===== sysfs attributes =====

static ssize_t version_show(struct kobject *kobj,
                              struct kobj_attribute *attr, char *buf)
{
    return sprintf(buf, "%s\n", HELLO_VERSION);
}

static ssize_t stats_show(struct kobject *kobj,
                           struct kobj_attribute *attr, char *buf)
{
    struct hello_device *dev = hello_dev;
    return sprintf(buf, "reads=%ld writes=%ld opens=%ld bytes=%ld\n",
                   atomic_long_read(&dev->read_count),
                   atomic_long_read(&dev->write_count),
                   atomic_long_read(&dev->open_count),
                   atomic_long_read(&dev->byte_count));
}

static ssize_t message_show(struct kobject *kobj,
                              struct kobj_attribute *attr, char *buf)
{
    struct hello_device *dev = hello_dev;
    ssize_t ret;
    
    mutex_lock(&dev->lock);
    ret = scnprintf(buf, PAGE_SIZE, "%.*s", 
                    (int)dev->buf_len, dev->buffer);
    mutex_unlock(&dev->lock);
    
    return ret;
}

static ssize_t message_store(struct kobject *kobj,
                               struct kobj_attribute *attr,
                               const char *buf, size_t count)
{
    struct hello_device *dev = hello_dev;
    size_t len = min(count, (size_t)(BUFFER_SIZE - 1));
    
    mutex_lock(&dev->lock);
    memcpy(dev->buffer, buf, len);
    dev->buffer[len] = '\0';
    dev->buf_len = len;
    mutex_unlock(&dev->lock);
    
    pr_info(DEVICE_NAME ": sysfs message set\n");
    return count;
}

static struct kobj_attribute version_attr = __ATTR_RO(version);
static struct kobj_attribute stats_attr   = __ATTR_RO(stats);
static struct kobj_attribute message_attr = __ATTR_RW(message);

static struct attribute *hello_attrs[] = {
    &version_attr.attr,
    &stats_attr.attr,
    &message_attr.attr,
    NULL,
};

static struct attribute_group hello_attr_group = {
    .attrs = hello_attrs,
};

// ===== Module init/exit =====

static int __init hello_device_init(void)
{
    int ret;
    dev_t devno;
    
    pr_info(DEVICE_NAME ": initializing version %s\n", HELLO_VERSION);
    
    // 1. Allocate device structure
    hello_dev = kzalloc(sizeof(*hello_dev), GFP_KERNEL);
    if (!hello_dev)
        return -ENOMEM;
    
    // 2. Allocate buffer
    hello_dev->buffer = kzalloc(BUFFER_SIZE, GFP_KERNEL);
    if (!hello_dev->buffer) {
        ret = -ENOMEM;
        goto fail_buffer;
    }
    
    // Set initial message
    snprintf(hello_dev->buffer, BUFFER_SIZE, "Hello from kernel!\n");
    hello_dev->buf_len = strlen(hello_dev->buffer);
    
    // 3. Initialize synchronization and counters
    mutex_init(&hello_dev->lock);
    atomic_long_set(&hello_dev->read_count, 0);
    atomic_long_set(&hello_dev->write_count, 0);
    atomic_long_set(&hello_dev->open_count, 0);
    atomic_long_set(&hello_dev->byte_count, 0);
    
    // 4. Allocate device number
    ret = alloc_chrdev_region(&devno, 0, 1, DEVICE_NAME);
    if (ret < 0) {
        pr_err(DEVICE_NAME ": alloc_chrdev_region failed\n");
        goto fail_chrdev;
    }
    major_num = MAJOR(devno);
    
    // 5. Initialize and register cdev
    cdev_init(&hello_dev->cdev, &hello_fops);
    hello_dev->cdev.owner = THIS_MODULE;
    ret = cdev_add(&hello_dev->cdev, devno, 1);
    if (ret) {
        pr_err(DEVICE_NAME ": cdev_add failed\n");
        goto fail_cdev;
    }
    
    // 6. Create device class
    hello_class = class_create(THIS_MODULE, CLASS_NAME);
    if (IS_ERR(hello_class)) {
        ret = PTR_ERR(hello_class);
        goto fail_class;
    }
    
    // 7. Create device node /dev/hello
    hello_dev_node = device_create(hello_class, NULL, devno,
                                    NULL, DEVICE_NAME);
    if (IS_ERR(hello_dev_node)) {
        ret = PTR_ERR(hello_dev_node);
        goto fail_device;
    }
    
    // 8. Create /proc/hello_info
    hello_proc = proc_create("hello_info", 0444, NULL, &hello_proc_ops);
    if (!hello_proc)
        pr_warn(DEVICE_NAME ": failed to create proc entry\n");
    
    // 9. Create sysfs entries /sys/kernel/hello/
    hello_dev->kobj = kobject_create_and_add(DEVICE_NAME, kernel_kobj);
    if (!hello_dev->kobj) {
        pr_warn(DEVICE_NAME ": kobject_create_and_add failed\n");
    } else {
        ret = sysfs_create_group(hello_dev->kobj, &hello_attr_group);
        if (ret)
            pr_warn(DEVICE_NAME ": sysfs_create_group failed\n");
    }
    
    pr_info(DEVICE_NAME ": registered (major=%d)\n", major_num);
    pr_info(DEVICE_NAME ": /dev/%s created\n", DEVICE_NAME);
    pr_info(DEVICE_NAME ": /proc/hello_info created\n");
    pr_info(DEVICE_NAME ": /sys/kernel/hello/ created\n");
    
    return 0;

fail_device:
    class_destroy(hello_class);
fail_class:
    cdev_del(&hello_dev->cdev);
fail_cdev:
    unregister_chrdev_region(devno, 1);
fail_chrdev:
    kfree(hello_dev->buffer);
fail_buffer:
    kfree(hello_dev);
    return ret;
}

static void __exit hello_device_exit(void)
{
    dev_t devno = MKDEV(major_num, 0);
    
    // Remove sysfs
    if (hello_dev->kobj) {
        sysfs_remove_group(hello_dev->kobj, &hello_attr_group);
        kobject_put(hello_dev->kobj);
    }
    
    // Remove /proc entry
    if (hello_proc)
        proc_remove(hello_proc);
    
    // Remove device and class
    device_destroy(hello_class, devno);
    class_destroy(hello_class);
    
    // Remove cdev
    cdev_del(&hello_dev->cdev);
    unregister_chrdev_region(devno, 1);
    
    // Free memory
    kfree(hello_dev->buffer);
    kfree(hello_dev);
    
    pr_info(DEVICE_NAME ": unregistered\n");
}

module_init(hello_device_init);
module_exit(hello_device_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Assembly Course");
MODULE_DESCRIPTION("Complete /dev/hello character device driver");
MODULE_VERSION(HELLO_VERSION);
```

---

## 16. Userspace Test Program

```c
// test_hello.c - Test program สำหรับ /dev/hello driver

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <errno.h>

#define DEVICE_PATH "/dev/hello"

// IOCTL commands (ต้อง match กับ kernel module)
#define HELLO_IOC_MAGIC     'H'
#define HELLO_RESET         _IO(HELLO_IOC_MAGIC, 0)
#define HELLO_GET_LEN       _IOR(HELLO_IOC_MAGIC, 1, int)
#define HELLO_SET_MSG       _IOW(HELLO_IOC_MAGIC, 2, char[256])

struct hello_stats {
    unsigned long read_count;
    unsigned long write_count;
    unsigned long open_count;
    unsigned long byte_count;
};
#define HELLO_GET_STATS _IOR(HELLO_IOC_MAGIC, 3, struct hello_stats)

int main(void)
{
    int fd;
    char buf[1024];
    ssize_t n;
    int len;
    struct hello_stats stats;
    char msg[256];
    
    printf("=== Testing /dev/hello ===\n\n");
    
    // 1. Open device
    fd = open(DEVICE_PATH, O_RDWR);
    if (fd < 0) {
        perror("open");
        return 1;
    }
    printf("1. Opened %s (fd=%d)\n", DEVICE_PATH, fd);
    
    // 2. Read initial message
    n = read(fd, buf, sizeof(buf) - 1);
    if (n < 0) {
        perror("read");
        close(fd);
        return 1;
    }
    buf[n] = '\0';
    printf("2. Read %zd bytes: '%s'\n", n, buf);
    
    // 3. Write new message
    const char *new_msg = "Hello from userspace!";
    n = write(fd, new_msg, strlen(new_msg));
    printf("3. Wrote %zd bytes\n", n);
    
    // 4. Seek to beginning and read back
    lseek(fd, 0, SEEK_SET);
    n = read(fd, buf, sizeof(buf) - 1);
    buf[n] = '\0';
    printf("4. Read back: '%s'\n", buf);
    
    // 5. IOCTL: Get length
    if (ioctl(fd, HELLO_GET_LEN, &len) == 0)
        printf("5. IOCTL GET_LEN: %d bytes\n", len);
    
    // 6. IOCTL: Set message
    strncpy(msg, "World", sizeof(msg) - 1);
    if (ioctl(fd, HELLO_SET_MSG, msg) == 0) {
        lseek(fd, 0, SEEK_SET);
        n = read(fd, buf, sizeof(buf) - 1);
        buf[n] = '\0';
        printf("6. IOCTL SET_MSG 'World': '%s'\n", buf);
    }
    
    // 7. IOCTL: Get stats
    if (ioctl(fd, HELLO_GET_STATS, &stats) == 0) {
        printf("7. Stats: reads=%lu writes=%lu opens=%lu bytes=%lu\n",
               stats.read_count, stats.write_count,
               stats.open_count, stats.byte_count);
    }
    
    // 8. IOCTL: Reset
    if (ioctl(fd, HELLO_RESET) == 0)
        printf("8. Reset successful\n");
    
    // 9. Read after reset (should be empty)
    lseek(fd, 0, SEEK_SET);
    n = read(fd, buf, sizeof(buf) - 1);
    printf("9. After reset, read %zd bytes\n", n);
    
    // 10. Close
    close(fd);
    printf("10. Closed device\n\n");
    
    // Read /proc entry
    printf("=== /proc/hello_info ===\n");
    FILE *proc = fopen("/proc/hello_info", "r");
    if (proc) {
        while (fgets(buf, sizeof(buf), proc))
            printf("%s", buf);
        fclose(proc);
    }
    
    // Read sysfs entries
    printf("\n=== /sys/kernel/hello/ ===\n");
    
    FILE *sf;
    sf = fopen("/sys/kernel/hello/version", "r");
    if (sf) {
        fgets(buf, sizeof(buf), sf);
        printf("version: %s", buf);
        fclose(sf);
    }
    
    sf = fopen("/sys/kernel/hello/stats", "r");
    if (sf) {
        fgets(buf, sizeof(buf), sf);
        printf("stats: %s", buf);
        fclose(sf);
    }
    
    // Write via sysfs
    sf = fopen("/sys/kernel/hello/message", "w");
    if (sf) {
        fprintf(sf, "Hello via sysfs!");
        fclose(sf);
        printf("Wrote via sysfs\n");
    }
    
    // Read back via /dev/hello
    fd = open(DEVICE_PATH, O_RDONLY);
    if (fd >= 0) {
        n = read(fd, buf, sizeof(buf) - 1);
        buf[n] = '\0';
        printf("Read via /dev: '%s'\n", buf);
        close(fd);
    }
    
    printf("\nAll tests completed!\n");
    return 0;
}
```

---

## 17. Advanced Topics: Workqueue และ Tasklet

### 17.1 Workqueue

```c
#include <linux/workqueue.h>

// Workqueue ใช้สำหรับ deferred work ใน process context
// เหมาะสำหรับงานที่ใช้เวลานานจาก interrupt context

struct my_work {
    struct work_struct work;
    int data;
    char message[64];
};

// Work handler - ทำงานใน process context
static void my_work_func(struct work_struct *work)
{
    struct my_work *mw = container_of(work, struct my_work, work);
    
    // สามารถ sleep ได้ที่นี่
    pr_info("Work handler: data=%d, msg=%s\n", mw->data, mw->message);
    
    // ทำงานหนักได้
    msleep(100);  // sleep 100ms
    
    kfree(mw);  // Free work struct
}

// Schedule work จาก interrupt handler
static irqreturn_t irq_handler(int irq, void *dev_id)
{
    struct my_work *work;
    
    work = kmalloc(sizeof(*work), GFP_ATOMIC);  // ใช้ GFP_ATOMIC ใน IRQ
    if (!work)
        return IRQ_HANDLED;
    
    INIT_WORK(&work->work, my_work_func);
    work->data = 42;
    strncpy(work->message, "from IRQ", sizeof(work->message));
    
    // Schedule on system workqueue
    schedule_work(&work->work);
    
    return IRQ_HANDLED;
}

// Delayed work
static DECLARE_DELAYED_WORK(my_delayed_work, my_work_func);

// Schedule delayed work (100ms later)
schedule_delayed_work(&my_delayed_work, msecs_to_jiffies(100));

// Cancel delayed work
cancel_delayed_work_sync(&my_delayed_work);

// Custom workqueue (สำหรับ isolation)
static struct workqueue_struct *my_wq;

my_wq = create_singlethread_workqueue("my_queue");
// หรือ
my_wq = alloc_workqueue("my_queue", WQ_UNBOUND | WQ_HIGHPRI, 0);

queue_work(my_wq, &work->work);

// ลบ workqueue
flush_workqueue(my_wq);
destroy_workqueue(my_wq);
```

### 17.2 Tasklet

```c
#include <linux/interrupt.h>

// Tasklet - deferred work ที่ทำงานใน softirq context
// ห้าม sleep, แต่เร็วกว่า workqueue
// deprecated ใน kernel ใหม่ๆ (ใช้ threaded IRQ แทน)

struct my_tasklet {
    struct tasklet_struct tasklet;
    u32 data;
};

static struct my_tasklet my_tl;

static void tasklet_func(unsigned long data)
{
    struct my_tasklet *tl = (struct my_tasklet *)data;
    pr_info("Tasklet: data=0x%08x\n", tl->data);
}

// Initialize
tasklet_init(&my_tl.tasklet, tasklet_func, (unsigned long)&my_tl);

// หรือ static initialization
static DECLARE_TASKLET(my_static_tasklet, tasklet_func, (unsigned long)&my_tl);

// Schedule tasklet
tasklet_schedule(&my_tl.tasklet);

// Kill tasklet (รอให้ทำงานเสร็จ)
tasklet_kill(&my_tl.tasklet);
```

---

## 18. Kernel Timers

```c
#include <linux/timer.h>
#include <linux/jiffies.h>

// HZ - ticks per second (ปกติ 250 หรือ 1000)
// jiffies - current tick count

// ตัวอย่าง: periodic timer
static struct timer_list my_timer;
static int timer_count = 0;

static void timer_callback(struct timer_list *t)
{
    timer_count++;
    pr_info("Timer callback #%d, jiffies=%lu\n", timer_count, jiffies);
    
    // Re-arm timer (ถ้าต้องการ periodic)
    if (timer_count < 10) {
        mod_timer(&my_timer, jiffies + HZ);  // 1 second
    }
}

static int __init timer_example_init(void)
{
    timer_setup(&my_timer, timer_callback, 0);
    
    // Start timer: fire after 1 second
    mod_timer(&my_timer, jiffies + HZ);
    
    pr_info("Timer started\n");
    return 0;
}

static void __exit timer_example_exit(void)
{
    del_timer_sync(&my_timer);
    pr_info("Timer stopped\n");
}

// High-resolution timers
#include <linux/hrtimer.h>

static struct hrtimer my_hrtimer;

static enum hrtimer_restart hrtimer_callback(struct hrtimer *timer)
{
    pr_info("hrtimer callback\n");
    
    // Restart timer
    hrtimer_forward_now(timer, ns_to_ktime(1000000));  // 1ms
    return HRTIMER_RESTART;
    
    // หรือ return HRTIMER_NORESTART เพื่อหยุด
}

// Initialize and start hrtimer
hrtimer_init(&my_hrtimer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
my_hrtimer.function = hrtimer_callback;
hrtimer_start(&my_hrtimer, ns_to_ktime(1000000), HRTIMER_MODE_REL);

// Cancel
hrtimer_cancel(&my_hrtimer);
```

---

## 19. Wait Queues

```c
#include <linux/wait.h>

// Wait queue ใช้สำหรับ block process จนกว่า condition จะเป็น true
// เป็นกลไกพื้นฐานสำหรับ blocking I/O

DECLARE_WAIT_QUEUE_HEAD(my_queue);

// หรือ dynamic initialization
wait_queue_head_t my_queue;
init_waitqueue_head(&my_queue);

// ตัวอย่าง: blocking read driver
static bool data_ready = false;

// ใน read() function
static ssize_t blocking_read(struct file *filp, char __user *buf,
                              size_t count, loff_t *f_pos)
{
    // Block จนกว่า data_ready จะเป็น true
    if (wait_event_interruptible(my_queue, data_ready == true))
        return -ERESTARTSYS;  // interrupted by signal
    
    // Now we have data, serve it
    data_ready = false;
    // ... copy data to user ...
    
    return count;
}

// ใน interrupt handler หรือ producer
static irqreturn_t data_received_irq(int irq, void *dev)
{
    data_ready = true;
    wake_up_interruptible(&my_queue);  // wake up readers
    return IRQ_HANDLED;
}

// wait_event variants:
wait_event(wq, condition);              // uninterruptible wait
wait_event_interruptible(wq, cond);     // interruptible (return -ERESTARTSYS)
wait_event_timeout(wq, cond, timeout);  // with timeout (jiffies)
wait_event_interruptible_timeout(wq, cond, timeout);  // both

// Wake up variants:
wake_up(&wq);               // wake up one exclusive waiter or all non-exclusive
wake_up_all(&wq);           // wake up all waiters
wake_up_interruptible(&wq); // wake up only TASK_INTERRUPTIBLE
```

---

## 20. เทคนิค Debugging ใน Kernel

### 20.1 KASAN (Kernel Address Sanitizer)

```bash
# เปิด KASAN ใน kernel config:
CONFIG_KASAN=y
CONFIG_KASAN_OUTLINE=y  # หรือ INLINE

# KASAN ตรวจจับ:
# - use-after-free
# - out-of-bounds access
# - double-free
# - use-after-scope

# Output เมื่อตรวจพบ bug:
# ==================================================================
# BUG: KASAN: slab-out-of-bounds in my_function+0x40/0x80
# Write of size 1 at addr ffff888000000000 by task test/123
```

### 20.2 DEBUG_KMEMLEAK

```bash
# Memory leak detection
CONFIG_DEBUG_KMEMLEAK=y

# Trigger manual scan:
echo scan > /sys/kernel/debug/kmemleak

# อ่าน leaks:
cat /sys/kernel/debug/kmemleak
```

### 20.3 LOCKDEP (Lock Dependency Detector)

```bash
CONFIG_LOCKDEP=y
CONFIG_DEBUG_MUTEXES=y
CONFIG_DEBUG_SPINLOCK=y

# LOCKDEP ตรวจจับ:
# - deadlocks
# - lock ordering violations
# - recursive locking
```

### 20.4 Dynamic Debug

```c
// Dynamic debug ให้เปิด/ปิด debug messages runtime
// ใช้ pr_debug() หรือ dev_dbg()

pr_debug("This is dynamic debug: value=%d\n", val);

// เปิด debug สำหรับ module ที่ต้องการ:
echo "module mymodule +p" > /sys/kernel/debug/dynamic_debug/control

// หรือ specific file:
echo "file drivers/my/mydriver.c +p" > /sys/kernel/debug/dynamic_debug/control

// เปิดทั้งหมด:
echo "module mymodule +pflmt" > /sys/kernel/debug/dynamic_debug/control
# p = print, f = function name, l = line number, m = module, t = thread

// ปิด:
echo "module mymodule -p" > /sys/kernel/debug/dynamic_debug/control
```

---

## 21. Platform Device Driver Framework

### 21.1 platform_driver

```c
#include <linux/platform_device.h>
#include <linux/of.h>       // Device Tree
#include <linux/of_gpio.h>  // GPIO from DT

// ตัวอย่าง: Platform device driver
struct myplatform_data {
    void __iomem *base;
    int irq;
    struct clk *clk;
};

// Probe - เรียกเมื่อ device match กับ driver
static int myplatform_probe(struct platform_device *pdev)
{
    struct myplatform_data *data;
    struct resource *res;
    int ret;
    
    dev_info(&pdev->dev, "probing device\n");
    
    // Allocate private data
    data = devm_kzalloc(&pdev->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;
    
    // Get memory-mapped resource from Device Tree/ACPI
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    data->base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(data->base))
        return PTR_ERR(data->base);
    
    // Get IRQ
    data->irq = platform_get_irq(pdev, 0);
    if (data->irq < 0)
        return data->irq;
    
    // Request IRQ (devm_ version: auto-freed on driver detach)
    ret = devm_request_irq(&pdev->dev, data->irq,
                            my_irq_handler, 0, "myplatform", data);
    if (ret)
        return ret;
    
    // Get clock
    data->clk = devm_clk_get(&pdev->dev, "main");
    if (IS_ERR(data->clk))
        return PTR_ERR(data->clk);
    
    ret = clk_prepare_enable(data->clk);
    if (ret)
        return ret;
    
    // Store private data
    platform_set_drvdata(pdev, data);
    
    dev_info(&pdev->dev, "probe successful\n");
    return 0;
}

static int myplatform_remove(struct platform_device *pdev)
{
    struct myplatform_data *data = platform_get_drvdata(pdev);
    
    clk_disable_unprepare(data->clk);
    // devm_* resources freed automatically
    
    dev_info(&pdev->dev, "removed\n");
    return 0;
}

// Device Tree compatible strings
static const struct of_device_id myplatform_dt_ids[] = {
    { .compatible = "myvendor,mydevice-v1" },
    { .compatible = "myvendor,mydevice-v2" },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, myplatform_dt_ids);

static struct platform_driver myplatform_driver = {
    .probe  = myplatform_probe,
    .remove = myplatform_remove,
    .driver = {
        .name  = "myplatform",
        .of_match_table = myplatform_dt_ids,
    },
};

// Simplified registration macro
module_platform_driver(myplatform_driver);
```

---

## 22. สรุปและแนวทางการพัฒนาต่อ

### 22.1 Common Pitfalls

```
1. Sleeping in atomic context:
   - ห้าม sleep/schedule ขณะถือ spinlock หรือใน interrupt context
   - ใช้ lockdep เพื่อตรวจสอบ

2. Use-after-free:
   - ตั้ง pointer = NULL หลัง kfree
   - ใช้ KASAN สำหรับ detection

3. Race conditions:
   - ทุก shared data ต้องมี synchronization
   - ใช้ RCU สำหรับ read-heavy data structures

4. Stack overflow:
   - ห้ามใช้ large local variables
   - จำ stack size จำกัดที่ 8-16KB

5. Memory leaks:
   - ทุก kmalloc/vmalloc ต้องมี kfree/vfree
   - ใช้ devm_* สำหรับ platform drivers
   - ใช้ kmemleak สำหรับ detection

6. Integer overflow:
   - ระวัง size_t arithmetic
   - ใช้ check_mul_overflow, check_add_overflow

7. Missing NULL checks:
   - ตรวจสอบ return value ของทุก allocation function
   - container_of อาจ return pointer ที่ไม่ถูกต้องถ้า offset ผิด
```

### 22.2 Resources และ Documentation

```
- Kernel Documentation: /Documentation (ใน kernel source)
- LWN.net - excellent kernel development articles
- kernelnewbies.org - beginner resources
- The Linux Kernel Module Programming Guide
- Linux Device Drivers (LDD3) - ebook ฟรี
- Bootlin Kernel Slides - slides ที่ดีมากจาก Bootlin
- kernel.org/doc/html/latest/ - official online documentation

Essential Books:
- "Linux Device Drivers" by Rubini & Corbet (LDD3)
- "Linux Kernel Development" by Robert Love
- "Understanding the Linux Kernel" by Bovet & Cesati
- "Linux Kernel Programming" by Kaiwan N Billimoria
```

### 22.3 Next Steps

```
หลังจากเรียน Part 093 แล้ว ควรศึกษาต่อใน:

Part 094: USB Device Driver Development
Part 095: PCI/PCIe Driver Development
Part 096: Network Device Drivers (NAPI)
Part 097: Block Device Drivers
Part 098: SPI/I2C Device Drivers
Part 099: GPIO and Pin Control Subsystem
Part 100: eBPF and Kernel Observability

Advanced topics:
- IOMMU and VFIO (passthrough to VMs)
- RDMA drivers
- GPU drivers (DRM/KMS framework)
- Real-time Linux (PREEMPT_RT)
- eBPF kernel programs
```

---

## Quick Reference Card

```c
// ===== MEMORY =====
kmalloc(size, GFP_KERNEL)        // Allocate (process context)
kmalloc(size, GFP_ATOMIC)        // Allocate (interrupt context)
kzalloc(size, GFP_KERNEL)        // Allocate + zero
kfree(ptr)                       // Free
vmalloc(size)                    // Large allocation
vfree(ptr)                       // Free vmalloc
kmem_cache_create(...)           // Create slab cache
kmem_cache_alloc(cache, flags)   // Alloc from cache
kmem_cache_free(cache, ptr)      // Free to cache

// ===== SYNC =====
spin_lock(&lock)                 // Lock (process/IRQ context)
spin_unlock(&lock)
spin_lock_irqsave(&lock, flags)  // Lock + disable IRQs
spin_unlock_irqrestore(&lock, flags)
mutex_lock(&mutex)               // Lock (process context, can sleep)
mutex_unlock(&mutex)
mutex_lock_interruptible(&mutex) // Lock, interruptible by signal

// ===== MMIO =====
ioremap(phys, size)              // Map physical to virtual
iounmap(virt)                    // Unmap
readb/readw/readl(addr)          // Read from MMIO
writeb/writew/writel(val, addr)  // Write to MMIO

// ===== IRQ =====
request_irq(irq, handler, flags, name, dev) // Register IRQ
free_irq(irq, dev)                           // Unregister IRQ
disable_irq(irq)                             // Disable IRQ
enable_irq(irq)                              // Enable IRQ

// ===== USER/KERNEL SPACE =====
copy_to_user(dst, src, n)        // Kernel -> User
copy_from_user(dst, src, n)      // User -> Kernel

// ===== DEBUG =====
pr_err/pr_warn/pr_info/pr_debug(fmt, ...)  // printk helpers
dump_stack()                               // Print stack trace
BUG_ON(condition)                          // Bug if condition true
WARN_ON(condition)                         // Warning if condition true
```

---

*Part 093 - Linux Kernel Module Development*
*Assembly & Systems Programming Course*
*ระดับ: Expert | Lines: 1200+*
