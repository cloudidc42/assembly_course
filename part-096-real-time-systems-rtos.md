# Part 096: Real-Time Systems and RTOS

## ระบบเรียลไทม์และ RTOS - จากทฤษฎีสู่การปฏิบัติระดับ Expert

---

## 1. บทนำ: ความหมายของ "Real-Time"

คำว่า "Real-Time" มักถูกเข้าใจผิดว่าหมายถึง "เร็วมาก" แต่ความหมายที่แท้จริงคือ **ระบบที่ต้องตอบสนองภายในเวลาที่กำหนด (deadline) อย่างแน่นอน** ความเร็วเป็นเพียงผลพลอยได้ ไม่ใช่เป้าหมายหลัก

### 1.1 Hard Real-Time vs Soft Real-Time

```
┌─────────────────────────────────────────────────────────────────┐
│                    Classification of Real-Time                   │
├──────────────────────┬──────────────────────┬───────────────────┤
│    Hard Real-Time    │   Firm Real-Time      │  Soft Real-Time   │
├──────────────────────┼──────────────────────┼───────────────────┤
│ Miss deadline =      │ Miss deadline =       │ Miss deadline =   │
│ CATASTROPHIC FAILURE │ result useless but   │ degraded quality  │
│                      │ no system failure    │ acceptable        │
├──────────────────────┼──────────────────────┼───────────────────┤
│ Examples:            │ Examples:            │ Examples:         │
│ - ABS brakes         │ - Video streaming    │ - Web server      │
│ - Pacemaker          │ - Online gaming      │ - Audio player    │
│ - Aircraft control   │ - Weather monitoring │ - UI updates      │
│ - Airbag trigger     │                      │                   │
└──────────────────────┴──────────────────────┴───────────────────┘
```

**Hard Real-Time** คือระบบที่หากพลาด deadline แม้เพียงครั้งเดียว ระบบถือว่าล้มเหลว อาจเกิดอันตรายต่อชีวิตหรือทรัพย์สิน

ตัวอย่างที่ชัดเจน: ถุงลมนิรภัย (Airbag) ต้องพองตัวภายใน 20-30ms หลังจากตรวจจับการชน หากช้ากว่านี้ถือว่าไม่มีประโยชน์และอาจทำให้ผู้โดยสารบาดเจ็บหนักกว่าเดิม

**Soft Real-Time** คือระบบที่ยอมรับได้หาก deadline ผิดพลาดบ้าง แต่ยิ่งผิดน้อยยิ่งดี

### 1.2 Guarantees และ Deadlines

ใน Hard Real-Time ต้องการ **determinism** - ความสามารถในการพิสูจน์ทางคณิตศาสตร์ว่าระบบจะทำงานเสร็จทันเวลาเสมอ

```
Task Timing Model (Liu & Layland 1973):
┌────────────────────────────────────────────────────────────────┐
│  τᵢ = (Cᵢ, Dᵢ, Tᵢ)                                            │
│                                                                  │
│  Cᵢ = Computation time (WCET)    ← ต้องรู้แน่นอน              │
│  Dᵢ = Deadline                   ← ต้องตอบสนองก่อนนี้         │
│  Tᵢ = Period (สำหรับ periodic task)                             │
│                                                                  │
│  Utilization: Uᵢ = Cᵢ/Tᵢ                                       │
│  Total utilization: U = Σ(Cᵢ/Tᵢ)                               │
└────────────────────────────────────────────────────────────────┘
```

---

## 2. WCET Analysis (Worst Case Execution Time)

WCET คือเวลาสูงสุดที่ task หนึ่งๆ จะใช้ในการทำงานในสถานการณ์ที่แย่ที่สุด การวิเคราะห์ WCET เป็นหัวใจของ Hard Real-Time systems

### 2.1 วิธีการวิเคราะห์ WCET

**Method 1: Measurement-Based Analysis**
```c
// วัด execution time จริงๆ ด้วย DWT (Data Watchpoint and Trace)
// บน ARM Cortex-M

void wcet_measure_start(void) {
    CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;  // Enable DWT
    DWT->CYCCNT = 0;                                    // Reset counter
    DWT->CTRL |= DWT_CTRL_CYCCNTENA_Msk;               // Start counting
}

uint32_t wcet_measure_stop(void) {
    return DWT->CYCCNT;  // ค่าที่ได้คือ CPU cycles
}

// การใช้งาน:
void measure_task_wcet(void) {
    uint32_t max_cycles = 0;
    
    for (int run = 0; run < 10000; run++) {
        // เตรียม worst-case input
        prepare_worst_case_input();
        
        wcet_measure_start();
        run_task_under_test();
        uint32_t cycles = wcet_measure_stop();
        
        if (cycles > max_cycles) {
            max_cycles = cycles;
        }
    }
    
    // เวลาใน microseconds (สมมติ CPU 168 MHz)
    float wcet_us = (float)max_cycles / 168.0f;
    printf("WCET: %u cycles (%.2f µs)\n", max_cycles, wcet_us);
}
```

**Method 2: Static Analysis (IPET - Implicit Path Enumeration Technique)**
```
Control Flow Graph Analysis:
         ┌──────────┐
         │  Entry   │
         └────┬─────┘
              │
         ┌────▼─────┐
         │  Block A │  cycles: 5
         └────┬─────┘
         ┌────▼─────┐
    ┌────┤  Branch  ├────┐
    │    └──────────┘    │
    │ (true)             │ (false)
    ▼                    ▼
┌───────┐            ┌───────┐
│ Blk B │  c:12      │ Blk C │  c:3
└───┬───┘            └───┬───┘
    └──────────┬──────────┘
               ▼
         ┌─────────┐
         │  Blk D  │  c:8
         └─────────┘

WCET = A + max(B, C) + D = 5 + 12 + 8 = 25 cycles
```

### 2.2 Cache และ Pipeline Effects

สิ่งที่ทำให้ WCET วิเคราะห์ยากมาก:

```
Cache Miss Penalty (ARM Cortex-M7):
- L1 I-Cache miss: ~5 cycles
- L1 D-Cache miss: ~10-20 cycles
- Branch misprediction: ~12 cycles

Pipeline stalls:
- Data hazard: 1-3 cycles
- Load-use hazard: 1 cycle
- Store-load forwarding: variable

ตัวอย่างการคำนวณ WCET แบบ conservative:
WCET_pessimistic = BCET × (1 + cache_miss_ratio × penalty)
```

---

## 3. Interrupt Latency และ Scheduling Latency

### 3.1 Interrupt Latency

```
Timeline ของ Interrupt:

Hardware Event
     │
     ▼ t₀
─────┼──── Current Instruction Finishes ─────
                                             │
                                             ▼ t₁
                                    CPU Enters ISR Vector Table
                                             │ (3-12 cycles บน Cortex-M4)
                                             ▼ t₂
                                    ISR Prologue (push registers)
                                             │
                                             ▼ t₃
                                    First ISR Instruction Executes

Interrupt Latency = t₃ - t₀

Factors affecting latency:
1. CPU pipeline depth
2. Current instruction complexity (เช่น divide อาจใช้ 2-12 cycles)
3. Memory wait states
4. Interrupt nesting/masking
```

**การวัด Interrupt Latency บน ARM Cortex-M4:**
```c
// ใช้ GPIO toggle และ oscilloscope
// หรือใช้ DWT

volatile uint32_t irq_entry_time;
volatile uint32_t irq_trigger_time;

// ใน ISR:
void EXTI0_IRQHandler(void) {
    irq_entry_time = DWT->CYCCNT;  // บันทึกเวลาที่เข้า ISR
    
    // process interrupt...
    
    EXTI->PR = EXTI_PR_PR0;  // Clear pending bit
}

// ใน main หรือ task อื่น:
void trigger_and_measure(void) {
    irq_trigger_time = DWT->CYCCNT;
    
    // Software trigger interrupt
    EXTI->SWIER = EXTI_SWIER_SWIER0;
    
    // รอให้ ISR รัน
    __DSB(); __ISB();
    
    uint32_t latency = irq_entry_time - irq_trigger_time;
    printf("IRQ Latency: %u cycles\n", latency);
}
```

### 3.2 Scheduling Latency

```
Scheduling Latency Timeline:

Task A (high priority) becomes READY
        │
        ▼
RTOS Kernel notices (usually in ISR or SysTick)
        │  ← Kernel overhead
        ▼
PendSV triggered for context switch
        │  ← Context save time
        ▼
Task A context restored
        │  ← Context restore time
        ▼
Task A first instruction executes

Total Scheduling Latency = Kernel overhead + Context switch time

FreeRTOS typical values (Cortex-M4 @ 168MHz):
- Context switch time: ~3-5 µs
- SysTick period: typically 1ms (configTICK_RATE_HZ = 1000)
- Worst case scheduling latency: up to 1ms + context switch time
```

---

## 4. Scheduling Algorithms

### 4.1 Preemptive vs Cooperative Scheduling

```
Cooperative Scheduling:
┌─────────┐    yield()    ┌─────────┐    yield()    ┌─────────┐
│  Task A │──────────────►│  Task B │──────────────►│  Task C │
└─────────┘               └─────────┘               └─────────┘
• Task ต้องยอม CPU เอง
• ไม่มีความ Deterministic
• ง่ายต่อการ debug (no race conditions)
• ใช้ใน: Arduino cooperative scheduler, early embedded systems

Preemptive Scheduling:
┌─────────┐
│  Task A │
└────┬────┘
     │ HIGH PRIORITY TASK READY!
     │ ◄── Kernel preempts Task A
     ▼
┌─────────┐
│  Task B │ (high priority)
└────┬────┘
     │ Task B completes/blocks
     ▼
┌─────────┐
│  Task A │ (resumes)
└─────────┘
• Kernel สามารถ interrupt task ตลอดเวลา
• Deterministic response time
• ต้องระวัง race conditions, shared data
• ใช้ใน: FreeRTOS, Zephyr, VxWorks, RTEMS
```

### 4.2 Rate Monotonic Scheduling (RM)

Rate Monotonic เป็น optimal fixed-priority scheduling algorithm สำหรับ periodic tasks

**กฎ:** Task ที่มี period สั้นกว่าได้รับ priority สูงกว่า

```
ตัวอย่าง:
Task 1: C₁=1ms, T₁=4ms  → Priority 1 (highest)
Task 2: C₂=2ms, T₂=6ms  → Priority 2
Task 3: C₃=2ms, T₃=12ms → Priority 3 (lowest)

Utilization check:
U = 1/4 + 2/6 + 2/12 = 0.25 + 0.333 + 0.167 = 0.75

RM Utilization Bound: U ≤ n(2^(1/n) - 1)
For n=3: U ≤ 3(2^(1/3) - 1) = 3(1.26 - 1) = 0.78

0.75 ≤ 0.78 → SCHEDULABLE! ✓

Timeline:
t:  0    1    2    3    4    5    6    7    8    9   10   11   12
T1: ████░░░░████░░░░████░░░░████
T2: ░░░░████████░░░░░░░░████████
T3: ░░░░░░░░░░░░████████░░░░░░░░
```

**การคำนวณ Response Time Analysis:**
```
Response time R₁ = C₁ = 1ms (highest priority, no interference)

R₂ = C₂ + ⌈R₂/T₁⌉ × C₁
R₂⁰ = C₂ = 2
R₂¹ = 2 + ⌈2/4⌉ × 1 = 2 + 1 = 3
R₂² = 2 + ⌈3/4⌉ × 1 = 2 + 1 = 3 ✓ (converged)
R₂ = 3ms ≤ D₂ = 6ms ✓

R₃ = C₃ + ⌈R₃/T₁⌉ × C₁ + ⌈R₃/T₂⌉ × C₂
R₃⁰ = 2
R₃¹ = 2 + ⌈2/4⌉×1 + ⌈2/6⌉×2 = 2+1+2 = 5
R₃² = 2 + ⌈5/4⌉×1 + ⌈5/6⌉×2 = 2+2+2 = 6
R₃³ = 2 + ⌈6/4⌉×1 + ⌈6/6⌉×2 = 2+2+2 = 6 ✓
R₃ = 6ms ≤ D₃ = 12ms ✓
```

### 4.3 EDF (Earliest Deadline First)

EDF เป็น dynamic priority algorithm ที่ optimal สำหรับ preemptive uniprocessor

**กฎ:** Task ที่ deadline ใกล้ที่สุดได้รับ priority สูงสุด ณ เวลานั้น

```
EDF Schedulability condition (for implicit deadlines):
U = Σ(Cᵢ/Tᵢ) ≤ 1

ตัวอย่างเดิม: U = 0.75 ≤ 1 → SCHEDULABLE with EDF

EDF vs RM:
┌────────────────┬─────────────────┬──────────────────┐
│                │       RM        │       EDF         │
├────────────────┼─────────────────┼──────────────────┤
│ Priority type  │ Static (fixed)  │ Dynamic          │
│ Max utilization│ ≈69% (for large n)│ 100%            │
│ Implementation │ Simple          │ More complex     │
│ Overhead       │ Low             │ Higher           │
│ Predictability │ High            │ Medium           │
│ Miss pattern   │ Graceful        │ Domino effect    │
│ Used in        │ Most RTOS       │ Linux SCHED_DEADLINE│
└────────────────┴─────────────────┴──────────────────┘
```

---

## 5. Priority Inversion - กรณีศึกษา Mars Pathfinder

### 5.1 เหตุการณ์ที่เกิดขึ้น

ในปี 1997 ยาน Mars Pathfinder ลงจอดบนดาวอังคาร และเริ่มส่งข้อมูลกลับมา แต่เกิดปัญหา: ระบบ reboot ตัวเองซ้ำๆ ทำให้สูญเสียข้อมูลวิทยาศาสตร์อันมีค่า

ระบบที่ใช้: VxWorks RTOS บน IBM RAD6000 processor

**สาเหตุ: Priority Inversion**

```
Task Priority (High to Low):
T_high  : Bus Management Task    (priority 1, highest)
T_medium: Communication Task     (priority 2)  
T_low   : Meteorological Task    (priority 3, lowest)

Shared Resource: Information Bus (protected by mutex M)

Sequence of events that caused the bug:
┌─────────────────────────────────────────────────────────────┐
│ t=0: T_low acquires mutex M (to write sensor data)          │
│ t=1: T_high preempts T_low, tries to acquire M → BLOCKED    │
│ t=2: T_medium becomes READY, preempts T_low (lower priority)│
│      T_low is preempted while HOLDING mutex M!              │
│ t=3: T_medium runs...runs...runs...                         │
│      T_high is BLOCKED waiting for M                        │
│      T_low cannot run (preempted by T_medium)               │
│ t=4: WATCHDOG TIMER EXPIRES!                                 │
│      T_high has been blocked too long → System RESET        │
└─────────────────────────────────────────────────────────────┘

Effective priority of T_high = priority of T_low = LOW!
This is Priority Inversion.
```

### 5.2 Priority Inheritance Protocol (PIP)

วิธีแก้ปัญหา: เมื่อ task priority สูงถูก block โดย mutex ที่ task priority ต่ำถือไว้ ให้ยก priority ของ task priority ต่ำขึ้นมาชั่วคราว

```c
// FreeRTOS Priority Inheritance Implementation concept:
// (FreeRTOS รองรับ Priority Inheritance ใน Mutex แต่ไม่รองรับใน Semaphore)

// สร้าง Mutex (ไม่ใช่ semaphore!) เพื่อใช้ Priority Inheritance:
SemaphoreHandle_t xMutex;
xMutex = xSemaphoreCreateMutex();  // ← ใช้ Mutex สำหรับ PI

// Task low priority (จำลอง T_low):
void vTaskLow(void *pvParameters) {
    while(1) {
        // รอให้ถึงเวลา
        vTaskDelay(pdMS_TO_TICKS(100));
        
        // Acquire mutex - FreeRTOS จะ track ว่า task นี้ถือ mutex
        if(xSemaphoreTake(xMutex, portMAX_DELAY) == pdTRUE) {
            // ทำงานกับ shared resource
            // ถ้า T_high ต้องการ mutex ตอนนี้:
            // FreeRTOS จะ boost priority ของ T_low → priority ของ T_high
            do_sensor_reading();  // อาจใช้เวลาหน่อย
            
            xSemaphoreGive(xMutex);
            // Priority กลับเป็นปกติอัตโนมัติหลังจาก Give
        }
    }
}

// Task high priority (จำลอง T_high):
void vTaskHigh(void *pvParameters) {
    while(1) {
        vTaskDelay(pdMS_TO_TICKS(20));  // period สั้นกว่า
        
        // ต้องการ mutex เพื่ออ่านข้อมูล
        if(xSemaphoreTake(xMutex, pdMS_TO_TICKS(10)) == pdTRUE) {
            // ถ้า T_low กำลัง hold mutex อยู่:
            // T_low จะถูก boost ไป priority เดียวกับ T_high
            // T_medium จะไม่สามารถแย่ง CPU จาก T_low ได้
            
            process_bus_data();
            xSemaphoreGive(xMutex);
        }
    }
}
```

### 5.3 Priority Ceiling Protocol (PCP)

PCP เป็นอีกวิธีหนึ่งที่มีประสิทธิภาพกว่า PI ในบางกรณี

**กฎ PCP:**
1. กำหนด Priority Ceiling ของแต่ละ resource = priority สูงสุดของ task ที่อาจใช้ resource นั้น
2. Task จะ acquire resource ได้ก็ต่อเมื่อ priority ของมันสูงกว่า ceiling ของ resources ทั้งหมดที่ถูก hold โดย tasks อื่นๆ

```
ตัวอย่าง PCP:
Resources: R1 (ceiling=1), R2 (ceiling=2)
Tasks: T1(prio=1), T2(prio=2), T3(prio=3)

การทำงาน:
T3 hold R2 (ceiling=2)
T2 พยายาม acquire R2 → ทำได้ถ้า prio(T2)=2 > current_ceiling=2? NO!
T2 ถูก block แม้ว่า R2 จะว่างอยู่!

ข้อดี: ป้องกัน deadlock และ chained blocking
ข้อเสีย: ซับซ้อนกว่า, อาจ block task โดยไม่จำเป็น
```

---

## 6. FreeRTOS Internals

### 6.1 Task Control Block (TCB) Structure

```c
// FreeRTOS TCB structure (simplified from tasks.c)
// ไฟล์จริง: FreeRTOS/Source/tasks.c

typedef struct tskTaskControlBlock {
    // *** ต้องอยู่ตำแหน่งแรก! ***
    volatile StackType_t *pxTopOfStack;    // Stack pointer ของ task
    
    #if ( portUSING_MPU_WRAPPERS == 1 )
        xMPU_SETTINGS xMPUSettings;        // MPU region settings
    #endif
    
    // Task state list membership
    ListItem_t xStateListItem;             // อยู่ใน ready/blocked/suspended list
    ListItem_t xEventListItem;             // อยู่ใน event list (queue, semaphore)
    
    // Priority
    UBaseType_t uxPriority;               // Current priority
    UBaseType_t uxBasePriority;           // Original priority (ก่อน PI boost)
    
    // Stack information
    StackType_t *pxStack;                 // Stack base pointer
    
    // Task name
    char pcTaskName[configMAX_TASK_NAME_LEN];
    
    // Stack depth (for overflow detection)
    #if ( ( portSTACK_GROWTH > 0 ) || ( configRECORD_STACK_HIGH_ADDRESS == 1 ) )
        StackType_t *pxEndOfStack;
    #endif
    
    // Critical nesting count (per task)
    #if ( portCRITICAL_NESTING_IN_TCB == 1 )
        UBaseType_t uxCriticalNesting;
    #endif
    
    // Run time stats
    #if ( configGENERATE_RUN_TIME_STATS == 1 )
        configRUN_TIME_COUNTER_TYPE ulRunTimeCounter;
    #endif
    
    // Task notification
    #if ( configUSE_TASK_NOTIFICATIONS == 1 )
        volatile uint32_t ulNotifiedValue[configTASK_NOTIFICATION_ARRAY_ENTRIES];
        volatile uint8_t ucNotifyState[configTASK_NOTIFICATION_ARRAY_ENTRIES];
    #endif
    
    // Mutex count (for priority inheritance)
    #if ( configUSE_MUTEXES == 1 )
        UBaseType_t uxMutexesHeld;
    #endif
    
} tskTCB;

typedef tskTCB TCB_t;
```

### 6.2 xTaskCreate - การสร้าง Task

```c
// Signature:
BaseType_t xTaskCreate(
    TaskFunction_t pxTaskCode,      // Function pointer ไปยัง task function
    const char * const pcName,      // ชื่อ task (สำหรับ debug)
    const configSTACK_DEPTH_TYPE usStackDepth, // Stack size เป็น words
    void * const pvParameters,      // Parameter ส่งให้ task function
    UBaseType_t uxPriority,         // Priority (0 = lowest)
    TaskHandle_t * const pxCreatedTask  // Handle สำหรับอ้างอิง task
);

// ตัวอย่างการใช้งานจริง:
TaskHandle_t xSensorTask;
TaskHandle_t xControlTask;
TaskHandle_t xCommTask;

// Sensor reading task - real-time periodic
void vSensorTask(void *pvParameters) {
    TickType_t xLastWakeTime;
    const TickType_t xFrequency = pdMS_TO_TICKS(10);  // 100Hz
    
    // Initialize the xLastWakeTime variable with the current time
    xLastWakeTime = xTaskGetTickCount();
    
    for(;;) {
        // Wait for the next cycle
        vTaskDelayUntil(&xLastWakeTime, xFrequency);
        
        // อ่านค่า sensor
        float temperature = read_temperature_sensor();
        float pressure = read_pressure_sensor();
        float humidity = read_humidity_sensor();
        
        // ส่งไปยัง queue สำหรับ processing
        SensorData_t data = {temperature, pressure, humidity, xTaskGetTickCount()};
        xQueueSend(xSensorQueue, &data, 0);  // Non-blocking!
    }
}

// Control loop task - higher priority
void vControlTask(void *pvParameters) {
    SensorData_t sensorData;
    ControlOutput_t output;
    
    for(;;) {
        // รอข้อมูลจาก sensor queue
        if(xQueueReceive(xSensorQueue, &sensorData, pdMS_TO_TICKS(50)) == pdTRUE) {
            // PID control calculation
            output = calculate_pid_control(sensorData);
            
            // Apply control output
            apply_control_output(output);
        }
    }
}

int main(void) {
    hardware_init();
    
    // สร้าง queue ก่อน tasks
    xSensorQueue = xQueueCreate(10, sizeof(SensorData_t));
    
    // สร้าง tasks
    xTaskCreate(
        vSensorTask,           // Task function
        "Sensor",              // Task name
        256,                   // Stack size (256 words = 1024 bytes on 32-bit)
        NULL,                  // Parameters
        3,                     // Priority (3 = medium)
        &xSensorTask           // Handle
    );
    
    xTaskCreate(
        vControlTask,
        "Control",
        512,                   // Control task ต้องการ stack มากกว่า
        NULL,
        4,                     // Higher priority than sensor
        &xControlTask
    );
    
    // Start the scheduler
    vTaskStartScheduler();
    
    // ถ้าถึงตรงนี้ = error! (usually heap exhausted)
    for(;;);
}
```

### 6.3 vTaskDelay vs vTaskDelayUntil

```c
// vTaskDelay: delay relative to NOW
// ปัญหา: execution time สะสมทำให้ period drift!
void vBadPeriodicTask(void *pvParam) {
    for(;;) {
        do_work();           // ใช้เวลา variable
        vTaskDelay(pdMS_TO_TICKS(100));  // delay 100ms จากจุดนี้
        // Actual period = execution_time + 100ms
        // Period DRIFTS over time!
    }
}

// vTaskDelayUntil: absolute timing - ไม่ drift!
void vGoodPeriodicTask(void *pvParam) {
    TickType_t xLastWakeTime = xTaskGetTickCount();
    const TickType_t xPeriod = pdMS_TO_TICKS(100);
    
    for(;;) {
        // MUST call this before doing work (or after)
        vTaskDelayUntil(&xLastWakeTime, xPeriod);
        
        do_work();  // execution time ไม่ทำให้ period drift
        // Next wake time = previous wake time + period (fixed!)
    }
}

/*
Timeline comparison:
vTaskDelay (DRIFTING):
│ work │ delay │ work │ delay │ work │ delay │
0      3      103     106     206     210     310   ← period 100, 103, 104... DRIFT!

vTaskDelayUntil (ACCURATE):
│ work │ delay │ work │ delay │ work │
0      3      100     103     200     203     300   ← period exactly 100ms
*/
```

### 6.4 Semaphores และ Queues

```c
// ===== Binary Semaphore (Signaling) =====
// ใช้สำหรับ synchronization ระหว่าง task กับ ISR

SemaphoreHandle_t xBinarySem;

void vISR_Handler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
    // ส่งสัญญาณจาก ISR
    xSemaphoreGiveFromISR(xBinarySem, &xHigherPriorityTaskWoken);
    
    // Context switch ถ้าจำเป็น
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

void vProcessingTask(void *pvParam) {
    for(;;) {
        // รอสัญญาณจาก ISR
        xSemaphoreTake(xBinarySem, portMAX_DELAY);
        
        // ประมวลผลข้อมูล
        process_interrupt_data();
    }
}

// ===== Counting Semaphore =====
// ใช้สำหรับ resource management (เช่น buffer pool)

SemaphoreHandle_t xCountingSem;

void init_resource_pool(void) {
    // สร้าง semaphore ที่มี 5 resources
    xCountingSem = xSemaphoreCreateCounting(5, 5);  // max=5, initial=5
}

void vConsumerTask(void *pvParam) {
    for(;;) {
        // รอจนกว่าจะมี resource ว่าง
        xSemaphoreTake(xCountingSem, portMAX_DELAY);
        
        // ใช้ resource
        use_shared_resource();
        
        // คืน resource
        xSemaphoreGive(xCountingSem);
    }
}

// ===== Queue =====
// ใช้ส่งข้อมูลระหว่าง tasks

typedef struct {
    uint16_t adc_value;
    uint32_t timestamp;
    uint8_t channel;
} ADC_Data_t;

QueueHandle_t xADCQueue;

void vADCTask(void *pvParam) {
    ADC_Data_t data;
    TickType_t xLastWakeTime = xTaskGetTickCount();
    
    for(;;) {
        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(1)); // 1kHz
        
        data.adc_value = HAL_ADC_GetValue(&hadc1);
        data.timestamp = DWT->CYCCNT;
        data.channel = 0;
        
        // ส่งข้อมูล (non-blocking - ถ้า queue เต็ม ทิ้งข้อมูล)
        xQueueSend(xADCQueue, &data, 0);
    }
}

void vFilterTask(void *pvParam) {
    ADC_Data_t received;
    float filtered_value = 0;
    
    for(;;) {
        // รอข้อมูล (blocking)
        if(xQueueReceive(xADCQueue, &received, portMAX_DELAY) == pdTRUE) {
            // Moving average filter
            filtered_value = 0.95f * filtered_value + 0.05f * received.adc_value;
            
            // ส่งต่อไปยัง control task
            xQueueSend(xControlQueue, &filtered_value, 0);
        }
    }
}
```

---

## 7. FreeRTOS Context Switch: PendSV Handler ใน Assembly

นี่คือหัวใจของ RTOS - การ save/restore CPU state ระหว่าง tasks

### 7.1 ทำความเข้าใจ ARM Cortex-M Exception Stack Frame

```
เมื่อเกิด Exception บน ARM Cortex-M, CPU จะ push registers อัตโนมัติ:

Stack (grows downward) ก่อน exception:
┌──────────────────┐  ← SP (before exception)
│                  │
│   ...task data...│
│                  │
└──────────────────┘

Stack หลัง Hardware Auto-stacking:
┌──────────────────┐  ← SP (after exception)  ← PSP หรือ MSP
│   xPSR           │  +28 bytes from new SP
│   PC (Return)    │  +24 bytes
│   LR             │  +20 bytes
│   R12            │  +16 bytes
│   R3             │  +12 bytes
│   R2             │  +8 bytes
│   R1             │  +4 bytes
│   R0             │  ← SP after auto-stacking
└──────────────────┘

Software-saved registers (ต้อง save เอง):
R4-R11 ต้อง save ด้วยตัวเอง! CPU ไม่ save ให้
```

### 7.2 PendSV Handler - Context Switch Code

```asm
; FreeRTOS PendSV Handler สำหรับ ARM Cortex-M4
; File: portasm.s หรือ port.c (ขึ้นอยู่กับ port)
; ไฟล์จริงใน FreeRTOS/Source/portable/GCC/ARM_CM4F/portasm.S

    .syntax unified
    .thumb
    
    .extern pxCurrentTCB          ; Pointer to current TCB
    .extern vTaskSwitchContext     ; FreeRTOS function เลือก next task
    
    .text
    .thumb_func
    .global xPortPendSVHandler

xPortPendSVHandler:
    ; --------------- SAVE CURRENT TASK CONTEXT ---------------
    
    ; 1. Disable interrupts ชั่วคราว (critical section)
    ;    ป้องกัน interrupt เข้ามาแทรกตอนกำลัง switch
    MRS     r0, psp              ; Read Process Stack Pointer
    ISYNC                        ; Instruction Sync Barrier
    
    ; 2. Load current TCB pointer
    ;    pxCurrentTCB ชี้ไปที่ TCB ของ task ที่กำลังรัน
    LDR     r3, =pxCurrentTCB    ; r3 = address of pxCurrentTCB variable
    LDR     r2, [r3]             ; r2 = pxCurrentTCB (pointer to current TCB)
    
    ; 3. Save FPU context (ถ้า FPU ถูกใช้งาน)
    ;    Lazy stacking: FPU registers save เมื่อจำเป็นเท่านั้น
    TST     lr, #0x10            ; Test bit 4 of LR (EXC_RETURN)
    IT      EQ
    VSTMDBEQ r0!, {s16-s31}      ; Push FPU registers s16-s31 to PSP stack
    
    ; 4. Save R4-R11 (software-saved registers)
    ;    R0-R3, R12, LR, PC, xPSR ถูก save โดย hardware แล้ว
    STMDB   r0!, {r4-r11, r14}   ; Push r4-r11 and LR to PSP stack
    ;    r0 ตอนนี้ = new top of stack (after saving all registers)
    
    ; 5. Save new stack pointer ไปใน TCB
    ;    TCB->pxTopOfStack = r0
    STR     r0, [r2]             ; Store updated SP in current TCB
    
    ; --------------- SWITCH TO NEXT TASK ---------------
    
    ; 6. เลือก next task ที่จะรัน
    STMDB   sp!, {r3}            ; Save r3 (pxCurrentTCB address)
    MOV     r0, #configMAX_SYSCALL_INTERRUPT_PRIORITY
    MSR     basepri, r0          ; Mask interrupts ≤ configMAX_SYSCALL_INTERRUPT_PRIORITY
    DSB
    ISB
    BL      vTaskSwitchContext   ; FreeRTOS เลือก next task, update pxCurrentTCB
    MOV     r0, #0
    MSR     basepri, r0          ; Re-enable interrupts
    LDMIA   sp!, {r3}            ; Restore r3
    
    ; --------------- RESTORE NEXT TASK CONTEXT ---------------
    
    ; 7. Load new current TCB
    LDR     r1, [r3]             ; r1 = pxCurrentTCB (now points to next task's TCB)
    LDR     r0, [r1]             ; r0 = next task's saved SP (pxTopOfStack)
    
    ; 8. Restore R4-R11 and LR from next task's stack
    LDMIA   r0!, {r4-r11, r14}   ; Pop saved registers
    
    ; 9. Restore FPU context (ถ้าจำเป็น)
    TST     lr, #0x10            ; Check if FPU was used
    IT      EQ
    VLDMIAEQ r0!, {s16-s31}      ; Pop FPU registers
    
    ; 10. Update PSP to point to next task's exception frame
    MSR     psp, r0              ; PSP = next task's stack pointer
    ISB
    
    ; 11. Return from exception
    ;     Hardware จะ pop R0-R3, R12, LR, PC, xPSR อัตโนมัติ
    ;     CPU จะเริ่มรัน next task ณ จุดที่ค้างไว้
    BX      lr                   ; Return from exception

    .end
```

### 7.3 Context Switch Diagram

```
BEFORE CONTEXT SWITCH (Task A running):
PSP → ┌─────────┐
      │   R4    │  ← Software-saved (not yet in stack, in registers)
      │   R5    │
      │   ...   │
      │   R11   │
      │   xPSR  │  ← Hardware-saved (already in stack from exception entry)
      │   PC    │  (Task A's return address)
      │   LR    │
      │   R12   │
      │   R3    │
      │   R2    │
      │   R1    │
      │   R0    │
      └─────────┘

AFTER SAVING Task A:
TCB_A→ pxTopOfStack ────────────────────────┐
                                             │
PSP → ┌─────────┐ ← New SP (stored in TCB) ←┘
      │   LR    │  (EXC_RETURN value)
      │   R11   │
      │   R10   │
      │   R9    │
      │   R8    │
      │   R7    │
      │   R6    │
      │   R5    │
      │   R4    │
      │   xPSR  │
      │   PC    │  Task A resumes here next time
      │   LR    │
      │   R12   │
      │   R3    │
      │   R2    │
      │   R1    │
      │   R0    │
      └─────────┘

AFTER RESTORING Task B (from its TCB):
PSP → ┌─────────┐ ← TCB_B→pxTopOfStack
      │   xPSR  │
      │   PC    │  Task B continues from here!
      │   LR    │
      │   R12   │
      │   R3    │
      │   R2    │
      │   R1    │
      │   R0    │
      └─────────┘
CPU restores R4-R11 from stack above,
then hardware pops the rest when BX lr executes
```

---

## 8. ISR-Safe API: FromISR Suffix

### 8.1 ทำไมต้องมี ISR-safe API?

```
ปัญหา: ISR รัน interrupt service routine โดยตรง
ถ้าเรียก xQueueSend() จาก ISR:
1. ISR อาจถูก preempt โดย higher priority ISR (nested interrupts)
2. ISR อาจ call kernel ที่กำลัง suspend task อยู่ → deadlock!
3. ISR ไม่มี "task context" = ไม่มี stack ของ task

วิธีแก้: ใช้ ...FromISR() variants ที่:
1. ไม่ block (never suspends calling context)
2. ใช้ xHigherPriorityTaskWoken flag แทนการ context switch ทันที
3. Context switch ทำที่ปลาย ISR ด้วย portYIELD_FROM_ISR()
```

### 8.2 ตัวอย่าง ISR-Safe API

```c
// ===== Queue API =====
// Normal (task context only):
xQueueSend(xQueue, &data, timeout);      // อาจ block
xQueueReceive(xQueue, &data, timeout);   // อาจ block

// ISR-safe (ใช้ใน ISR เท่านั้น):
xQueueSendFromISR(xQueue, &data, &xHigherPriorityTaskWoken);    // ไม่ block
xQueueReceiveFromISR(xQueue, &data, &xHigherPriorityTaskWoken); // ไม่ block

// ===== Semaphore API =====
xSemaphoreGive(xSemaphore);                                    // task only
xSemaphoreGiveFromISR(xSemaphore, &xHigherPriorityTaskWoken);  // ISR safe

// ===== ตัวอย่างการใช้งานจริง =====

// UART RX ISR - ส่งข้อมูลที่รับได้ไปยัง queue
void USART2_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
    if(USART2->SR & USART_SR_RXNE) {
        uint8_t byte = (uint8_t)(USART2->DR & 0xFF);
        
        // ส่ง byte ไปยัง processing task
        // xHigherPriorityTaskWoken จะ set เป็น pdTRUE ถ้ามี task ที่รอ queue นี้
        // และ task นั้นมี priority สูงกว่า task ที่กำลัง interrupt อยู่
        xQueueSendFromISR(xUARTRxQueue, &byte, &xHigherPriorityTaskWoken);
    }
    
    if(USART2->SR & USART_SR_ORE) {
        // Overrun error - clear it
        volatile uint32_t dummy = USART2->DR;
        (void)dummy;
    }
    
    // ถ้า xHigherPriorityTaskWoken = pdTRUE = มี task priority สูงรอข้อมูลอยู่
    // portYIELD_FROM_ISR จะ set PendSV เพื่อทำ context switch
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

// ADC DMA Complete ISR
void DMA2_Stream0_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
    if(DMA2->LISR & DMA_LISR_TCIF0) {
        DMA2->LIFCR = DMA_LIFCR_CTCIF0;  // Clear interrupt flag
        
        // Notify processing task ว่า ADC data พร้อมแล้ว
        xSemaphoreGiveFromISR(xADCCompleteSemaphore, &xHigherPriorityTaskWoken);
    }
    
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

// ===== Task ที่รับข้อมูลจาก ISR =====
void vUARTProcessTask(void *pvParam) {
    uint8_t byte;
    uint8_t buffer[256];
    uint16_t buf_idx = 0;
    
    for(;;) {
        // รอ byte จาก UART ISR
        if(xQueueReceive(xUARTRxQueue, &byte, portMAX_DELAY) == pdTRUE) {
            buffer[buf_idx++] = byte;
            
            if(byte == '\n' || buf_idx >= 255) {
                buffer[buf_idx] = '\0';
                process_command((char*)buffer);
                buf_idx = 0;
            }
        }
    }
}
```

### 8.3 configMAX_SYSCALL_INTERRUPT_PRIORITY

```c
// FreeRTOSConfig.h
/*
 * configMAX_SYSCALL_INTERRUPT_PRIORITY คือ interrupt priority สูงสุด
 * ที่อนุญาตให้เรียก FreeRTOS API ได้
 *
 * ISR ที่มี priority สูงกว่านี้ (ตัวเลขต่ำกว่า) จะไม่ถูก mask โดย kernel
 * และ ต้องไม่เรียก FreeRTOS API!
 *
 * ARM Cortex-M: priority ต่ำ = ตัวเลขสูง (5, 6, 7...)
 *               priority สูง = ตัวเลขต่ำ (0, 1, 2...)
 */

#define configMAX_SYSCALL_INTERRUPT_PRIORITY    191   // Priority level 5 (bits 7:4 = 0xBx)
// หรือใน ARM format:
#define configMAX_SYSCALL_INTERRUPT_PRIORITY    ( 5 << (8 - configPRIO_BITS) )

/*
 * ตัวอย่างการตั้งค่า Interrupt Priority:
 * 
 * CAN IRQ: priority 4 → TOO HIGH! ห้ามเรียก FreeRTOS API
 * UART IRQ: priority 6 → OK, สามารถเรียก xQueueSendFromISR() ได้
 * I2C IRQ:  priority 7 → OK
 * SysTick:  priority 15 → OK (ต่ำสุด = safe)
 */

// การตั้งค่าจาก HAL:
HAL_NVIC_SetPriority(CAN1_RX0_IRQn, 4, 0);  // ห้ามเรียก FreeRTOS API!
HAL_NVIC_SetPriority(USART2_IRQn, 6, 0);     // เรียก ...FromISR() ได้
HAL_NVIC_SetPriority(I2C1_EV_IRQn, 7, 0);   // เรียก ...FromISR() ได้
```

---

## 9. FreeRTOS Tick: SysTick → xTaskIncrementTick → vTaskSwitchContext

### 9.1 SysTick Handler Chain

```c
// FreeRTOS SysTick flow:
// 
// [SysTick Exception] 
//        ↓
// xPortSysTickHandler() (port.c)
//        ↓
// xTaskIncrementTick() (tasks.c)
//        ↓ returns pdTRUE if context switch needed
// portYIELD_FROM_ISR() → set PendSV pending
//        ↓ (after ISR returns)
// [PendSV Exception]
//        ↓
// xPortPendSVHandler() (portasm.s)
//        ↓
// vTaskSwitchContext() (tasks.c)
//        ↓
// context switch happens

// Source code (simplified):

// 1. SysTick Handler (port.c):
void xPortSysTickHandler(void) {
    vPortRaiseBASEPRI();  // Disable lower priority interrupts
    
    {
        // Call FreeRTOS tick increment
        // Returns pdTRUE if a context switch should occur
        if(xTaskIncrementTick() != pdFALSE) {
            // Set PendSV to trigger context switch after ISR
            portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT;
        }
    }
    
    vPortClearBASEPRIFromISR();  // Re-enable interrupts
}

// 2. xTaskIncrementTick (tasks.c) - simplified:
BaseType_t xTaskIncrementTick(void) {
    TCB_t *pxTCB;
    TickType_t xItemValue;
    BaseType_t xSwitchRequired = pdFALSE;
    
    // Increment tick count
    const TickType_t xConstTickCount = xTickCount + (TickType_t)1;
    xTickCount = xConstTickCount;
    
    // ตรวจสอบว่า xTickCount overflow หรือยัง
    if(xConstTickCount == (TickType_t)0U) {
        taskSWITCH_DELAYED_LISTS();
    }
    
    // ตรวจสอบ delayed tasks ที่ถึงเวลาแล้ว
    if(xConstTickCount >= xNextTaskUnblockTime) {
        for(;;) {
            // ดึง task จาก delayed list
            listGET_OWNER_OF_HEAD_ENTRY(pxDelayedTaskList, pxTCB);
            xItemValue = listGET_LIST_ITEM_VALUE(&(pxTCB->xStateListItem));
            
            if(xConstTickCount < xItemValue) {
                // ยังไม่ถึงเวลา - update next unblock time
                xNextTaskUnblockTime = xItemValue;
                break;
            }
            
            // Task ถึงเวลาแล้ว - ย้ายไป ready list
            uxListRemove(&(pxTCB->xStateListItem));
            prvAddTaskToReadyList(pxTCB);
            
            // ตรวจสอบว่าต้องทำ context switch ไหม
            #if ( configUSE_PREEMPTION == 1 )
            {
                if(pxTCB->uxPriority >= pxCurrentTCB->uxPriority) {
                    xSwitchRequired = pdTRUE;
                }
            }
            #endif
        }
    }
    
    // Time slicing (ถ้าเปิดใช้)
    #if ( ( configUSE_PREEMPTION == 1 ) && ( configUSE_TIME_SLICING == 1 ) )
    {
        if(listCURRENT_LIST_LENGTH(&(pxReadyTasksLists[pxCurrentTCB->uxPriority])) > 1U) {
            xSwitchRequired = pdTRUE;  // ให้ task อื่นที่ priority เท่ากันได้รัน
        }
    }
    #endif
    
    return xSwitchRequired;
}
```

---

## 10. Memory Management: heap_1/2/4/5

FreeRTOS มี 5 implementations ของ memory allocator ให้เลือกใช้

### 10.1 Comparison Table

```
┌──────────┬──────────────────────────────────────────────────────┐
│ Scheme   │ Description                                          │
├──────────┼──────────────────────────────────────────────────────┤
│ heap_1   │ • Allocate-only, NEVER FREE                          │
│          │ • Simplest possible allocator                        │
│          │ • Single array partitioned by pointer increment      │
│          │ • Deterministic O(1)                                 │
│          │ • ใช้สำหรับ static systems ที่ allocate ครั้งเดียว  │
├──────────┼──────────────────────────────────────────────────────┤
│ heap_2   │ • Can free but no coalescence (ไม่รวม free blocks)   │
│          │ • Best-fit algorithm                                 │
│          │ • Fragment memory over time                          │
│          │ • ใช้เมื่อ block size เท่ากันเสมอ                    │
├──────────┼──────────────────────────────────────────────────────┤
│ heap_3   │ • Wrapper รอบ stdlib malloc()/free()                 │
│          │ • Thread-safe ด้วย suspend scheduler                 │
│          │ • ขนาด heap ตาม linker script                        │
│          │ • ไม่ Deterministic                                   │
├──────────┼──────────────────────────────────────────────────────┤
│ heap_4   │ • Best fit + coalescence (รวม adjacent free blocks)  │
│          │ • ลด fragmentation ได้ดี                             │
│          │ • RECOMMENDED สำหรับส่วนใหญ่                         │
│          │ • Single memory region                               │
├──────────┼──────────────────────────────────────────────────────┤
│ heap_5   │ • Same as heap_4 แต่รองรับ multiple memory regions   │
│          │ • ใช้เมื่อ RAM ไม่ต่อเนื่อง (เช่น SRAM + CCMRAM)    │
│          │ • ต้อง call vPortDefineHeapRegions() ก่อนใช้         │
└──────────┴──────────────────────────────────────────────────────┘
```

### 10.2 heap_4 Implementation (Core Concepts)

```c
// heap_4.c simplified implementation

// Header ของแต่ละ block ใน heap
typedef struct A_BLOCK_LINK {
    struct A_BLOCK_LINK *pxNextFreeBlock;  // ชี้ไป free block ถัดไป
    size_t xBlockSize;                     // ขนาด block (รวม header)
} BlockLink_t;

// Bit ที่บ่งบอกว่า block ถูก allocate แล้ว
#define heapBLOCK_ALLOCATED_BITMASK    ( ( size_t ) 1 << ( ( sizeof(size_t) * 8 ) - 1 ) )

static uint8_t ucHeap[configTOTAL_HEAP_SIZE];  // Heap memory array
static BlockLink_t xStart, *pxEnd = NULL;

// pvPortMalloc - thread-safe allocation:
void *pvPortMalloc(size_t xWantedSize) {
    BlockLink_t *pxBlock, *pxPreviousBlock, *pxNewBlockLink;
    void *pvReturn = NULL;
    
    vTaskSuspendAll();  // Disable scheduler for thread safety
    {
        // Initialize heap if first call
        if(pxEnd == NULL) {
            prvHeapInit();
        }
        
        // Adjust size to include block header and alignment
        if(xWantedSize & portBYTE_ALIGNMENT_MASK) {
            xWantedSize += (portBYTE_ALIGNMENT - (xWantedSize & portBYTE_ALIGNMENT_MASK));
        }
        xWantedSize += heapSTRUCT_SIZE;
        
        // Find a free block (best-fit search)
        pxPreviousBlock = &xStart;
        pxBlock = xStart.pxNextFreeBlock;
        
        while((pxBlock->xBlockSize < xWantedSize) && (pxBlock->pxNextFreeBlock != NULL)) {
            pxPreviousBlock = pxBlock;
            pxBlock = pxBlock->pxNextFreeBlock;
        }
        
        if(pxBlock != pxEnd) {
            pvReturn = (void*)(((uint8_t*)pxPreviousBlock->pxNextFreeBlock) + heapSTRUCT_SIZE);
            
            // Remove from free list
            pxPreviousBlock->pxNextFreeBlock = pxBlock->pxNextFreeBlock;
            
            // ถ้า block ใหญ่กว่าที่ต้องการมาก → แบ่ง block
            if((pxBlock->xBlockSize - xWantedSize) > heapMINIMUM_BLOCK_SIZE) {
                // สร้าง new block จากส่วนที่เหลือ
                pxNewBlockLink = (void*)(((uint8_t*)pxBlock) + xWantedSize);
                pxNewBlockLink->xBlockSize = pxBlock->xBlockSize - xWantedSize;
                pxBlock->xBlockSize = xWantedSize;
                
                // Insert new block กลับเข้า free list
                prvInsertBlockIntoFreeList(pxNewBlockLink);
            }
            
            // Mark block as allocated
            pxBlock->xBlockSize |= heapBLOCK_ALLOCATED_BITMASK;
            pxBlock->pxNextFreeBlock = NULL;
            
            xFreeBytesRemaining -= pxBlock->xBlockSize;
        }
    }
    xTaskResumeAll();
    
    return pvReturn;
}

// vPortFree - พร้อม coalescence:
void vPortFree(void *pv) {
    uint8_t *puc = (uint8_t*)pv;
    BlockLink_t *pxLink;
    
    if(pv != NULL) {
        // ย้อนกลับ pointer ไปหา block header
        puc -= heapSTRUCT_SIZE;
        pxLink = (void*)puc;
        
        // ตรวจสอบว่า block นี้ถูก allocate จริงๆ
        configASSERT((pxLink->xBlockSize & heapBLOCK_ALLOCATED_BITMASK) != 0);
        
        // Clear allocated bit
        pxLink->xBlockSize &= ~heapBLOCK_ALLOCATED_BITMASK;
        
        vTaskSuspendAll();
        {
            xFreeBytesRemaining += pxLink->xBlockSize;
            
            // ใส่กลับเข้า free list (sorted by address)
            // การ insert แบบ sorted ทำให้ coalescence ง่ายขึ้น
            prvInsertBlockIntoFreeList(pxLink);
            // prvInsertBlockIntoFreeList จะรวม adjacent free blocks อัตโนมัติ
        }
        xTaskResumeAll();
    }
}
```

### 10.3 heap_5: Multiple Memory Regions

```c
// ใช้สำหรับ MCU ที่มี RAM หลายชิ้น (เช่น STM32 SRAM + CCMRAM)
// STM32F4: SRAM (192KB at 0x20000000) + CCMRAM (64KB at 0x10000000)

#include "FreeRTOS.h"
#include "portable.h"

// ต้องเรียกก่อน vTaskStartScheduler() และก่อน pvPortMalloc() ทุกครั้ง
void setup_heap_5(void) {
    // Define memory regions
    HeapRegion_t xHeapRegions[] = {
        // CCMRAM (fast, 64KB) - ใช้ก่อนเพราะเร็วกว่า
        { (uint8_t*)0x10000000, 64 * 1024 },
        
        // SRAM (general purpose, 112KB - เหลือหลังจาก .bss, .data)
        // ใช้ linker symbol หาที่ว่าง
        { (uint8_t*)&_heap_start, &_heap_end - &_heap_start },
        
        // Terminate array
        { NULL, 0 }
    };
    
    vPortDefineHeapRegions(xHeapRegions);
}

// ใน linker script (STM32.ld):
/*
SECTIONS {
    .bss : { *(.bss) } > SRAM
    
    _heap_start = .;  ← ที่ว่างใน SRAM เริ่มตรงนี้
}
_heap_end = ORIGIN(SRAM) + LENGTH(SRAM);
*/
```

---

## 11. Stack Overflow Detection

### 11.1 วิธีการตรวจจับ

```c
// FreeRTOSConfig.h:
#define configCHECK_FOR_STACK_OVERFLOW    2   // Method 2 (most thorough)

/*
 * Method 1 (configCHECK_FOR_STACK_OVERFLOW = 1):
 * ตรวจสอบแค่ว่า SP ออกนอก stack boundary ตอน context switch
 * เร็วมาก แต่ detect ได้เฉพาะ overflow ที่ยังเกิดอยู่ตอน switch
 *
 * Method 2 (configCHECK_FOR_STACK_OVERFLOW = 2):
 * เพิ่มการตรวจ "canary bytes" 16 bytes ที่ปลาย stack
 * ถ้า bytes เหล่านี้เปลี่ยน = stack overflow เคยเกิดขึ้น
 */

// Callback function ที่ต้องเขียนเอง:
void vApplicationStackOverflowHook(
    TaskHandle_t xTask,
    char *pcTaskName
) {
    // เกิด stack overflow!
    // ตรงนี้ต้อง handle อย่างเหมาะสม
    
    // ตัวเลือก 1: หยุดระบบ (สำหรับ debugging)
    taskDISABLE_INTERRUPTS();
    
    // แสดงชื่อ task ที่ overflow (ถ้ามี UART/display)
    // บันทึกลง flash memory สำหรับ post-mortem analysis
    
    for(;;) {
        // หยุดอยู่ตรงนี้ให้ debugger เห็น
    }
}
```

### 11.2 Stack Sizing Guidelines

```c
// ขนาด stack ที่เหมาะสม:
// - แต่ละ function call frame: ~20-100 bytes (ขึ้นกับ local variables)
// - Local arrays: เพิ่มขนาดที่ประกาศ
// - FreeRTOS overhead: ~30-50 bytes
// - Interrupt nesting: อีก 32-64 bytes

// ตัวอย่างการคำนวณ:
void vMyTask(void *pvParam) {
    char buffer[256];     // +256 bytes
    uint32_t data[32];    // +128 bytes
    float matrix[4][4];   // +64 bytes
    // function call overhead: +~50 bytes
    // FreeRTOS overhead: ~50 bytes
    // TOTAL: ~550 bytes → ใช้ stack 256 words (1KB) เพื่อให้มี margin
}

// ดูการใช้ stack จริงๆ ด้วย:
void vTaskPrintStackUsage(void) {
    UBaseType_t uxHighWaterMark;
    
    // ส่งคืนจำนวน words ที่ยังไม่เคยใช้ (minimum free)
    uxHighWaterMark = uxTaskGetStackHighWaterMark(NULL);  // NULL = current task
    
    printf("Stack HWM: %u words remaining\n", uxHighWaterMark);
    
    // ถ้า HWM < 20 words = อันตราย!
}

// ดูทุก tasks:
void vTaskMonitor(void *pvParam) {
    TaskStatus_t *pxTaskStatusArray;
    UBaseType_t uxArraySize, x;
    
    for(;;) {
        uxArraySize = uxTaskGetNumberOfTasks();
        pxTaskStatusArray = pvPortMalloc(uxArraySize * sizeof(TaskStatus_t));
        
        if(pxTaskStatusArray != NULL) {
            uxArraySize = uxTaskGetSystemState(pxTaskStatusArray, uxArraySize, NULL);
            
            printf("%-16s %8s %8s %8s\n", "Task", "Priority", "StackHWM", "State");
            for(x = 0; x < uxArraySize; x++) {
                printf("%-16s %8u %8u %8u\n",
                    pxTaskStatusArray[x].pcTaskName,
                    pxTaskStatusArray[x].uxCurrentPriority,
                    pxTaskStatusArray[x].usStackHighWaterMark,
                    pxTaskStatusArray[x].eCurrentState);
            }
            
            vPortFree(pxTaskStatusArray);
        }
        
        vTaskDelay(pdMS_TO_TICKS(5000));  // ตรวจทุก 5 วินาที
    }
}
```

---

## 12. Watchdog Timer

### 12.1 IWDG vs WWDG บน STM32

```c
// ===== IWDG (Independent Watchdog) =====
// - ใช้ LSI clock (40kHz, อิสระจาก main clock)
// - ไม่มีทางปิดหลังจาก enable แล้ว
// - ใช้สำหรับ system-level deadlock detection

void iwdg_init(uint32_t timeout_ms) {
    // Enable IWDG peripheral
    IWDG->KR = 0xCCCC;  // Start IWDG
    IWDG->KR = 0x5555;  // Allow register access
    
    // Prescaler: LSI (40kHz) / prescaler / reload = timeout
    // timeout_ms = (reload + 1) * prescaler / 40000 * 1000
    
    IWDG->PR = IWDG_PR_PR_2;  // Prescaler = /64 → 625Hz
    // reload = timeout_ms * 625 / 1000 - 1
    IWDG->RLR = (timeout_ms * 625 / 1000) - 1;
    
    // Wait for registers to update
    while(IWDG->SR != 0);
    
    IWDG->KR = 0xAAAA;  // Reload and start
}

void iwdg_feed(void) {
    IWDG->KR = 0xAAAA;  // Reset countdown
}

// Task ที่ feed watchdog:
void vWatchdogTask(void *pvParam) {
    for(;;) {
        // ตรวจว่า critical tasks ยังรัน
        if(is_sensor_task_alive() && is_control_task_alive()) {
            iwdg_feed();
        }
        // ถ้า task ใด task หนึ่งหยุด → ไม่ feed → watchdog reset!
        
        vTaskDelay(pdMS_TO_TICKS(500));  // Feed ทุก 500ms (timeout = 1s)
    }
}

// ===== WWDG (Window Watchdog) =====
// - ต้อง feed ภายใน "window" ที่กำหนด
// - ห้าม feed เร็วเกินไปหรือช้าเกินไป
// - ตรวจจับทั้ง task freeze และ task ที่รัน loop เร็วผิดปกติ

void wwdg_init(void) {
    RCC->APB1ENR |= RCC_APB1ENR_WWDGEN;  // Enable WWDG clock
    
    // WWDG clock = PCLK1/4096 = 42MHz/4096 ≈ 10kHz
    // กับ prescaler /8: ~1.25kHz
    
    WWDG->CFR = WWDG_CFR_WDGTB_1 |  // Prescaler /8
                (0x50 << 0);          // Window = 0x50 (80 decimal)
    
    WWDG->CR = WWDG_CR_WDGA |        // Enable WWDG
               0x7F;                  // Counter = 0x7F (127 decimal)
    // Counter counts from 0x7F down to 0x40 → RESET
    // Must feed when counter is between 0x50 and 0x7F
}

void wwdg_feed(void) {
    WWDG->CR = WWDG_CR_WDGA | 0x7F;  // Reload counter
    // ถ้าเรียกตอน counter > 0x50 → reset ทันที (too early!)
}
```

---

## 13. Timing Analysis: DWT สำหรับวัด Task Execution Time

### 13.1 DWT (Data Watchpoint and Trace) Setup

```c
// DWT cycle counter เป็นวิธีที่ถูก overhead น้อยที่สุดในการวัดเวลา

void dwt_init(void) {
    // Enable trace and debug block
    CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
    
    // Reset cycle counter
    DWT->CYCCNT = 0;
    
    // Enable cycle counter
    DWT->CTRL |= DWT_CTRL_CYCCNTENA_Msk;
}

// Macro สำหรับ profiling:
#define DWT_START()     uint32_t _dwt_start = DWT->CYCCNT
#define DWT_STOP(name)  printf(#name ": %u cycles (%.2f µs)\n", \
                            DWT->CYCCNT - _dwt_start, \
                            (float)(DWT->CYCCNT - _dwt_start) / SystemCoreClock * 1e6f)

// ตัวอย่างการใช้งาน:
void profile_task_functions(void) {
    DWT_START();
    run_pid_calculation();
    DWT_STOP(PID);
    
    DWT_START();
    update_motor_pwm();
    DWT_STOP(PWM_Update);
    
    DWT_START();
    read_encoder();
    DWT_STOP(Encoder_Read);
}
```

### 13.2 Real-Time Profiling บน FreeRTOS

```c
// ใช้ FreeRTOS Run-Time Stats

// FreeRTOSConfig.h:
#define configGENERATE_RUN_TIME_STATS           1
#define configUSE_TRACE_FACILITY                1
#define configUSE_STATS_FORMATTING_FUNCTIONS    1

// ต้องให้ FreeRTOS counter ที่เร็วกว่า SysTick (อย่างน้อย 10x faster)
// ใช้ TIM2 หรือ TIM3:
#define portCONFIGURE_TIMER_FOR_RUN_TIME_STATS()    vSetupTimerForRunTimeStats()
#define portGET_RUN_TIME_COUNTER_VALUE()             ulGetRuntimeCounterValue()

volatile uint32_t ulHighFrequencyTimerTicks = 0;

void vSetupTimerForRunTimeStats(void) {
    // Setup TIM2 เป็น free-running counter 10kHz (10x ของ SysTick 1kHz)
    RCC->APB1ENR |= RCC_APB1ENR_TIM2EN;
    TIM2->PSC = (SystemCoreClock / 10000) - 1;  // 10kHz
    TIM2->ARR = 0xFFFFFFFF;
    TIM2->CR1 = TIM_CR1_CEN;
}

uint32_t ulGetRuntimeCounterValue(void) {
    return TIM2->CNT;
}

// ดู stats:
void vPrintRuntimeStats(void *pvParam) {
    char *pcBuffer = pvPortMalloc(1024);
    
    for(;;) {
        if(pcBuffer != NULL) {
            vTaskGetRunTimeStats(pcBuffer);
            printf("\n=== Run Time Stats ===\n%s\n", pcBuffer);
        }
        vTaskDelay(pdMS_TO_TICKS(10000));
    }
}
/* Output ตัวอย่าง:
=== Run Time Stats ===
Task            Abs Time        % Time
SensorTask      45231           25%
ControlTask     38492           21%
CommTask        12847           7%
WatchdogTask    843             <1%
IDLE            85042           47%
*/
```

---

## 14. Real Application: Sensor Read + Filter + Control Loop

### 14.1 ระบบควบคุมอุณหภูมิ PID

```c
// ระบบควบคุมอุณหภูมิ incubator (ตู้ฟักไข่)
// Sensor: PT100 RTD อ่านผ่าน ADC
// Actuator: Heater (PWM), Cooler (Fan PWM)
// Target: อุณหภูมิ ±0.1°C จาก setpoint

// ===== Configuration =====
#define SENSOR_TASK_PERIOD_MS    10      // 100Hz sensor sampling
#define FILTER_TASK_PERIOD_MS    10      // 100Hz filtering
#define CONTROL_TASK_PERIOD_MS   50      // 20Hz control loop
#define COMM_TASK_PERIOD_MS      1000    // 1Hz data logging

#define SENSOR_QUEUE_SIZE        20
#define FILTERED_QUEUE_SIZE      10

// ===== Task Priorities =====
#define PRIORITY_WATCHDOG    6
#define PRIORITY_CONTROL     5
#define PRIORITY_SENSOR      4
#define PRIORITY_FILTER      3
#define PRIORITY_COMM        2
#define PRIORITY_IDLE        0  // FreeRTOS idle task

// ===== Data Structures =====
typedef struct {
    float raw_adc;
    float temperature_c;
    uint32_t timestamp_ticks;
    bool valid;
} SensorReading_t;

typedef struct {
    float filtered_temp;
    float temp_rate;          // Rate of change °C/s
    uint32_t sample_count;
    uint32_t timestamp_ticks;
} FilteredData_t;

typedef struct {
    float setpoint;
    float current_temp;
    float heater_pwm;         // 0.0 to 1.0
    float cooler_pwm;         // 0.0 to 1.0
    float pid_error;
    float pid_integral;
    float pid_derivative;
} ControlState_t;

// ===== Global Handles =====
QueueHandle_t    xSensorQueue;
QueueHandle_t    xFilteredQueue;
SemaphoreHandle_t xControlMutex;
TaskHandle_t     xSensorTaskHandle;
TaskHandle_t     xFilterTaskHandle;
TaskHandle_t     xControlTaskHandle;

volatile ControlState_t gControlState;

// ===== Sensor Task =====
void vSensorTask(void *pvParam) {
    TickType_t xLastWakeTime = xTaskGetTickCount();
    SensorReading_t reading;
    
    // Calibration constants (measured during calibration)
    const float ADC_SCALE = 3.3f / 4096.0f;         // Vref/resolution
    const float PT100_SLOPE = 0.00385f;              // Ω/Ω/°C
    const float PT100_R0 = 100.0f;                   // Ω at 0°C
    const float BRIDGE_R = 100.0f;                   // Bridge resistor
    const float VREF = 3.3f;
    
    for(;;) {
        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(SENSOR_TASK_PERIOD_MS));
        
        // อ่าน ADC (average ของ 4 samples เพื่อลด noise)
        uint32_t adc_sum = 0;
        for(int i = 0; i < 4; i++) {
            HAL_ADC_Start(&hadc1);
            HAL_ADC_PollForConversion(&hadc1, 1);
            adc_sum += HAL_ADC_GetValue(&hadc1);
        }
        float adc_avg = (float)adc_sum / 4.0f;
        
        // Convert ADC to voltage
        float v_sense = adc_avg * ADC_SCALE;
        
        // Convert voltage to resistance (Wheatstone bridge)
        // V_out = V_ref * R_pt100 / (R_pt100 + R_bridge) - V_ref/2
        // Solving for R_pt100:
        float r_pt100 = BRIDGE_R * (v_sense + VREF / 2.0f) / (VREF - v_sense - VREF / 2.0f);
        
        // Convert resistance to temperature (Callendar-Van Dusen simplified)
        float temp = (r_pt100 / PT100_R0 - 1.0f) / PT100_SLOPE;
        
        reading.raw_adc = adc_avg;
        reading.temperature_c = temp;
        reading.timestamp_ticks = xTaskGetTickCount();
        reading.valid = (temp > -50.0f && temp < 200.0f);  // Sanity check
        
        // ส่งข้อมูลไป Filter task
        if(reading.valid) {
            xQueueSend(xSensorQueue, &reading, 0);  // Non-blocking
        }
    }
}

// ===== Filter Task: Kalman Filter =====
void vFilterTask(void *pvParam) {
    SensorReading_t raw;
    FilteredData_t filtered;
    
    // Kalman filter state
    float kf_state = 25.0f;          // Initial temperature estimate
    float kf_variance = 1.0f;        // Initial variance
    const float kf_q = 0.01f;        // Process noise
    const float kf_r = 0.5f;         // Measurement noise
    
    float prev_filtered = 25.0f;
    TickType_t prev_time = xTaskGetTickCount();
    uint32_t sample_count = 0;
    
    for(;;) {
        if(xQueueReceive(xSensorQueue, &raw, pdMS_TO_TICKS(100)) == pdTRUE) {
            // Kalman Filter Update
            // Predict:
            kf_variance += kf_q;
            
            // Update:
            float kf_gain = kf_variance / (kf_variance + kf_r);
            kf_state = kf_state + kf_gain * (raw.temperature_c - kf_state);
            kf_variance = (1.0f - kf_gain) * kf_variance;
            
            // Calculate rate of change
            TickType_t current_time = raw.timestamp_ticks;
            float dt = (float)(current_time - prev_time) / configTICK_RATE_HZ;
            
            if(dt > 0.0f) {
                filtered.temp_rate = (kf_state - prev_filtered) / dt;
            } else {
                filtered.temp_rate = 0.0f;
            }
            
            filtered.filtered_temp = kf_state;
            filtered.sample_count = ++sample_count;
            filtered.timestamp_ticks = current_time;
            
            prev_filtered = kf_state;
            prev_time = current_time;
            
            // ส่งไป Control task
            xQueueSend(xFilteredQueue, &filtered, 0);
        }
    }
}

// ===== Control Task: PID Controller =====
void vControlTask(void *pvParam) {
    FilteredData_t filtered;
    TickType_t xLastWakeTime = xTaskGetTickCount();
    
    // PID parameters (tuned for incubator)
    const float Kp = 5.0f;
    const float Ki = 0.1f;
    const float Kd = 2.0f;
    
    float integral = 0.0f;
    float prev_error = 0.0f;
    float setpoint = 37.5f;  // Target: 37.5°C for chicken eggs
    
    // Anti-windup limits
    const float INTEGRAL_MAX = 50.0f;
    const float INTEGRAL_MIN = -50.0f;
    
    for(;;) {
        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(CONTROL_TASK_PERIOD_MS));
        
        // รับข้อมูลล่าสุด (ดึงทั้งหมดจาก queue เอาแค่ล่าสุด)
        FilteredData_t latest = {0};
        bool got_data = false;
        
        while(xQueueReceive(xFilteredQueue, &filtered, 0) == pdTRUE) {
            latest = filtered;
            got_data = true;
        }
        
        if(!got_data) continue;
        
        float dt = (float)CONTROL_TASK_PERIOD_MS / 1000.0f;  // seconds
        float error = setpoint - latest.filtered_temp;
        
        // PID calculation
        integral += error * dt;
        integral = fminf(fmaxf(integral, INTEGRAL_MIN), INTEGRAL_MAX);  // Anti-windup
        
        float derivative = (error - prev_error) / dt;
        prev_error = error;
        
        float pid_output = Kp * error + Ki * integral + Kd * derivative;
        
        // Map PID output to heater/cooler
        float heater_pwm = 0.0f;
        float cooler_pwm = 0.0f;
        
        if(pid_output > 0.0f) {
            heater_pwm = fminf(pid_output / 10.0f, 1.0f);  // normalize
        } else {
            cooler_pwm = fminf(-pid_output / 10.0f, 1.0f);
        }
        
        // Apply to hardware
        // TIM3 CH1 = heater PWM, CH2 = cooler fan PWM
        TIM3->CCR1 = (uint32_t)(heater_pwm * TIM3->ARR);
        TIM3->CCR2 = (uint32_t)(cooler_pwm * TIM3->ARR);
        
        // Update global state (with mutex protection)
        if(xSemaphoreTake(xControlMutex, pdMS_TO_TICKS(1)) == pdTRUE) {
            gControlState.setpoint = setpoint;
            gControlState.current_temp = latest.filtered_temp;
            gControlState.heater_pwm = heater_pwm;
            gControlState.cooler_pwm = cooler_pwm;
            gControlState.pid_error = error;
            gControlState.pid_integral = integral;
            gControlState.pid_derivative = derivative;
            xSemaphoreGive(xControlMutex);
        }
    }
}

// ===== Communication Task =====
void vCommTask(void *pvParam) {
    char json_buffer[256];
    ControlState_t state_copy;
    
    for(;;) {
        vTaskDelay(pdMS_TO_TICKS(COMM_TASK_PERIOD_MS));
        
        // Copy state safely
        if(xSemaphoreTake(xControlMutex, pdMS_TO_TICKS(10)) == pdTRUE) {
            memcpy(&state_copy, (const void*)&gControlState, sizeof(ControlState_t));
            xSemaphoreGive(xControlMutex);
        }
        
        // Format as JSON and send via UART
        snprintf(json_buffer, sizeof(json_buffer),
            "{\"t\":%.2f,\"sp\":%.2f,\"h\":%.2f,\"c\":%.2f,"
            "\"err\":%.3f,\"int\":%.3f,\"der\":%.3f}\n",
            state_copy.current_temp,
            state_copy.setpoint,
            state_copy.heater_pwm,
            state_copy.cooler_pwm,
            state_copy.pid_error,
            state_copy.pid_integral,
            state_copy.pid_derivative
        );
        
        HAL_UART_Transmit(&huart2, (uint8_t*)json_buffer, strlen(json_buffer), 100);
    }
}

// ===== Main Function =====
int main(void) {
    HAL_Init();
    SystemClock_Config();
    
    MX_GPIO_Init();
    MX_ADC1_Init();
    MX_TIM3_Init();
    MX_USART2_UART_Init();
    
    // DWT สำหรับ profiling
    dwt_init();
    
    // สร้าง inter-task communication objects
    xSensorQueue   = xQueueCreate(SENSOR_QUEUE_SIZE, sizeof(SensorReading_t));
    xFilteredQueue = xQueueCreate(FILTERED_QUEUE_SIZE, sizeof(FilteredData_t));
    xControlMutex  = xSemaphoreCreateMutex();
    
    configASSERT(xSensorQueue != NULL);
    configASSERT(xFilteredQueue != NULL);
    configASSERT(xControlMutex != NULL);
    
    // เริ่ม Timer สำหรับ PWM
    HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);
    HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_2);
    
    // สร้าง Tasks (priority สูง = number สูง ใน FreeRTOS)
    xTaskCreate(vSensorTask,   "Sensor",   256, NULL, PRIORITY_SENSOR,   &xSensorTaskHandle);
    xTaskCreate(vFilterTask,   "Filter",   512, NULL, PRIORITY_FILTER,   &xFilterTaskHandle);
    xTaskCreate(vControlTask,  "Control",  512, NULL, PRIORITY_CONTROL,  &xControlTaskHandle);
    xTaskCreate(vCommTask,     "Comm",     256, NULL, PRIORITY_COMM,     NULL);
    xTaskCreate(vTaskMonitor,  "Monitor",  512, NULL, 1,                 NULL);
    
    // Watchdog
    iwdg_init(2000);  // 2 second watchdog timeout
    
    // Start Scheduler!
    vTaskStartScheduler();
    
    // ไม่ควรถึงตรงนี้
    Error_Handler();
}
```

---

## 15. RTOS Trace Tools

### 15.1 Percepio Tracealyzer

```c
// Tracealyzer เป็น commercial trace tool สำหรับ FreeRTOS
// Segger SystemView เป็น free alternative ที่ดีมาก

// การ setup Segger SystemView:
// 1. download SystemView package
// 2. copy SEGGER/SYSTEMVIEW/Src ไปในโปรเจค
// 3. ใน FreeRTOSConfig.h:

#define INCLUDE_xTaskGetIdleTaskHandle   1
#define INCLUDE_pxTaskGetStackStart      1

// Include ใน main.c:
#include "SEGGER_SYSVIEW.h"

void main(void) {
    // ...hardware init...
    
    SEGGER_SYSVIEW_Conf();  // เรียกก่อน vTaskStartScheduler()
    
    // ...create tasks...
    
    vTaskStartScheduler();
}

// SystemView จะบันทึก:
// - Task switches (เมื่อไหร่, ใครสลับกับใคร)
// - ISR entry/exit
// - Queue/semaphore operations
// - CPU utilization per task
// - Timeline visualization
```

### 15.2 Custom Trace Hooks

```c
// FreeRTOS มี trace macros ที่ user สามารถ override ได้
// ใน FreeRTOSConfig.h:

// เรียกทุกครั้งที่มี context switch
#define traceTASK_SWITCHED_IN()   vMyTraceTaskSwitchedIn()
#define traceTASK_SWITCHED_OUT()  vMyTraceTaskSwitchedOut()

// เรียกเมื่อ task เข้า/ออก blocked state
#define traceBLOCKING_ON_QUEUE_RECEIVE(pxQueue)  vMyTraceBlocking()
#define traceQUEUE_RECEIVE(pxQueue)               vMyTraceUnblocking()

// Implementation:
typedef struct {
    uint32_t timestamp;
    uint8_t  task_id;
    uint8_t  event_type;
} TraceEvent_t;

#define TRACE_BUFFER_SIZE 1024
TraceEvent_t gTraceBuffer[TRACE_BUFFER_SIZE];
uint32_t gTraceIdx = 0;

void vMyTraceTaskSwitchedIn(void) {
    if(gTraceIdx < TRACE_BUFFER_SIZE) {
        gTraceBuffer[gTraceIdx].timestamp = DWT->CYCCNT;
        gTraceBuffer[gTraceIdx].task_id   = (uint8_t)uxTaskGetTaskNumber(NULL);
        gTraceBuffer[gTraceIdx].event_type = 0;  // SWITCH_IN
        gTraceIdx++;
    }
}
```

---

## 16. Advanced Topics

### 16.1 Tickless Idle - ประหยัดพลังงาน

```c
// FreeRTOSConfig.h:
#define configUSE_TICKLESS_IDLE    1

// เมื่อ idle task รัน, FreeRTOS จะ:
// 1. คำนวณว่าจะ idle นานแค่ไหน (ถึง next wake event)
// 2. Stop SysTick timer
// 3. เข้า low-power mode (WFI/WFE บน ARM)
// 4. Wake up เมื่อ event เกิด หรือ timer expire
// 5. Compensate tick count

// Custom low-power hook:
void vPortSuppressTicksAndSleep(TickType_t xExpectedIdleTime) {
    // Stop SysTick
    SysTick->CTRL &= ~SysTick_CTRL_ENABLE_Msk;
    
    // Setup RTC wakeup timer เพื่อ wake ตาม xExpectedIdleTime
    uint32_t sleep_ms = xExpectedIdleTime * portTICK_PERIOD_MS;
    rtc_set_wakeup(sleep_ms);
    
    // Enter STOP mode (STM32)
    __WFI();  // Wait For Interrupt
    
    // Wake up! คำนวณเวลาที่หลับจริง
    uint32_t actual_sleep = rtc_get_elapsed_ms();
    TickType_t ticks_slept = actual_sleep / portTICK_PERIOD_MS;
    
    // Re-enable SysTick
    SysTick->CTRL |= SysTick_CTRL_ENABLE_Msk;
    
    // Compensate tick count
    vTaskStepTick(ticks_slept);
}
```

### 16.2 Task Notifications (เร็วกว่า Semaphore)

```c
// Task Notification เป็น mechanism ที่เร็วมากเพราะไม่ต้องสร้าง queue/semaphore
// แต่ละ task มี notification value (uint32_t) ของตัวเอง

// ส่ง notification จาก task:
xTaskNotify(xTargetTask, 0x01, eSetBits);  // Set bit 0
xTaskNotify(xTargetTask, 42,   eSetValueWithOverwrite);  // Set value

// ส่งจาก ISR:
xTaskNotifyFromISR(xTargetTask, 0x01, eSetBits, &xHigherPriorityTaskWoken);

// รับ notification:
uint32_t notification_value;
xTaskNotifyWait(
    0x00,               // Bits to clear on entry
    0xFFFFFFFF,         // Bits to clear on exit
    &notification_value,
    portMAX_DELAY       // Timeout
);

// ตัวอย่าง: แทนที่ semaphore ด้วย notification
void vFastISR(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
    // เร็วกว่า xSemaphoreGiveFromISR ประมาณ 45%
    vTaskNotifyGiveFromISR(xWaitingTask, &xHigherPriorityTaskWoken);
    
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

void vWaitingTask(void *pvParam) {
    for(;;) {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);  // Wait for notification
        process_data();
    }
}
```

### 16.3 การใช้ MPU (Memory Protection Unit) กับ FreeRTOS

```c
// MPU ป้องกัน task หนึ่งเขียนทับ memory ของ task อื่น
// ต้องใช้ xTaskCreateRestricted() แทน xTaskCreate()

// กำหนด regions สำหรับ task:
const TaskParameters_t xSensorTaskParams = {
    .pvTaskCode    = vSensorTask,
    .pcName        = "Sensor",
    .usStackDepth  = 256,
    .pvParameters  = NULL,
    .uxPriority    = PRIORITY_SENSOR | portPRIVILEGE_BIT,  // Privileged mode
    .puxStackBuffer = xSensorTaskStack,
    
    .xRegions = {
        // Region 0: Read-only access to sensor data buffer
        {
            .pvBaseAddress   = sensor_data_buffer,
            .ulLengthInBytes = sizeof(sensor_data_buffer),
            .ulParameters    = portMPU_REGION_READ_ONLY | portMPU_REGION_CACHEABLE
        },
        // Region 1: Read-write access to own state
        {
            .pvBaseAddress   = &sensor_task_state,
            .ulLengthInBytes = sizeof(sensor_task_state),
            .ulParameters    = portMPU_REGION_READ_WRITE
        },
        // Region 2: Unused
        { 0, 0, 0 }
    }
};

xTaskCreateRestricted(&xSensorTaskParams, &xSensorTaskHandle);
```

---

## 17. สรุปและ Checklist สำหรับ Real-Time Systems

### 17.1 Design Checklist

```
Real-Time System Design Checklist:

ANALYSIS PHASE:
[ ] กำหนด tasks ทั้งหมด พร้อม period, deadline, WCET
[ ] คำนวณ utilization: U = Σ(Cᵢ/Tᵢ) ≤ threshold
[ ] เลือก scheduling algorithm (RM หรือ EDF)
[ ] วิเคราะห์ response time ด้วย RTA
[ ] ระบุ shared resources ทั้งหมด
[ ] ออกแบบ protocol ป้องกัน priority inversion

IMPLEMENTATION PHASE:
[ ] ใช้ vTaskDelayUntil() ไม่ใช่ vTaskDelay() สำหรับ periodic tasks
[ ] ใช้ Mutex (ไม่ใช่ semaphore) สำหรับ mutual exclusion → ได้ Priority Inheritance
[ ] ใช้ ...FromISR() APIs ใน ISR
[ ] ตั้ง configMAX_SYSCALL_INTERRUPT_PRIORITY ถูกต้อง
[ ] Enable stack overflow detection (configCHECK_FOR_STACK_OVERFLOW = 2)
[ ] กำหนด stack size พอเพียง + เผื่อ margin 30%

TESTING PHASE:
[ ] วัด WCET จริงๆ ด้วย DWT
[ ] ตรวจ stack high water mark ทุก task
[ ] ทดสอบ worst-case timing ด้วย load injector
[ ] Stress test: รัน 24-48 ชั่วโมงโดยไม่ reset
[ ] Monitor heap fragmentation
[ ] ตรวจสอบด้วย RTOS trace tool

DEPLOYMENT PHASE:
[ ] Enable watchdog timer
[ ] ตั้งค่า fault handlers (HardFault, MemManage, etc.)
[ ] บันทึก fault information ลง non-volatile memory
[ ] ทดสอบ recovery จาก fault scenarios
```

### 17.2 Common Pitfalls

```
ข้อผิดพลาดที่พบบ่อยใน Real-Time Systems:

1. vTaskDelay() แทน vTaskDelayUntil()
   → Period drift สะสมเวลาผ่านไป

2. เรียก FreeRTOS API จาก ISR ที่ priority สูงเกินไป
   → System hang หรือ crash ที่หาสาเหตุยาก

3. Stack ขนาดเล็กเกินไป
   → Stack overflow ทำให้ข้อมูลเสียหายโดยไม่ทราบสาเหตุ

4. ใช้ Semaphore แทน Mutex สำหรับ mutual exclusion
   → Priority inversion ไม่ถูกแก้ไข

5. Blocking ใน ISR
   → xSemaphoreTake(portMAX_DELAY) ใน ISR = deadlock!

6. Global variables ไม่มี protection
   → Race condition, data corruption

7. malloc()/free() ใน RTOS task
   → Non-deterministic, fragmentation
   → ใช้ pvPortMalloc()/vPortFree() แทน

8. Polling ใน task แทน event-driven
   → CPU waste, latency เพิ่ม

9. Task priority ไม่ถูกต้อง
   → Low-priority task แย่ CPU จาก high-priority task

10. ไม่มี configASSERT
    → Error ถูกซ่อนอยู่, debug ยาก
    → ตั้งค่า: #define configASSERT(x) if(!(x)) { taskDISABLE_INTERRUPTS(); for(;;); }
```

---

## 18. Comparison: FreeRTOS vs Other RTOS

```
┌──────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│ Feature      │  FreeRTOS    │   Zephyr     │    RTEMS     │   VxWorks    │
├──────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ License      │ MIT          │ Apache 2.0   │ BSD          │ Commercial   │
│ Footprint    │ ~5-10KB      │ ~30KB+       │ ~50KB+       │ Variable     │
│ Scheduling   │ Preemptive   │ Preemptive   │ Preemptive   │ Preemptive   │
│ POSIX        │ Limited      │ Yes          │ Full         │ Yes          │
│ Certification│ IEC 61508    │ TÜV SÜD     │ DO-178C      │ DO-178C      │
│ Ecosystem    │ Large        │ Growing      │ Mature       │ Large        │
│ Hard RT      │ Yes (with    │ Yes          │ Yes          │ Yes          │
│              │ careful use) │              │              │              │
└──────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
```

---

## สรุป

Part 096 ได้ครอบคลุมหัวข้อ Real-Time Systems และ RTOS ในเชิงลึก ตั้งแต่:

1. **Hard vs Soft Real-Time**: ความแตกต่างและเกณฑ์การจำแนก
2. **WCET Analysis**: การวิเคราะห์ worst-case execution time ด้วย DWT
3. **Interrupt & Scheduling Latency**: การวัดและการลด latency
4. **Scheduling Algorithms**: RM, EDF, การพิสูจน์ schedulability
5. **Priority Inversion**: กรณีศึกษา Mars Pathfinder และวิธีแก้ไข
6. **FreeRTOS Internals**: TCB, context switch assembly code
7. **ISR-safe API**: การใช้ ...FromISR() อย่างถูกต้อง
8. **Memory Management**: heap_1 ถึง heap_5
9. **Stack Overflow Detection**: วิธีตรวจจับและป้องกัน
10. **Real Application**: ระบบควบคุมอุณหภูมิ PID แบบครบวงจร

ความสำเร็จของ Real-Time System ขึ้นอยู่กับการวิเคราะห์อย่างรอบคอบ การออกแบบที่ถูกต้อง และการทดสอบที่ครอบคลุม ไม่ใช่แค่การเขียนโค้ดที่ดี

---

*Part 096 - Real-Time Systems and RTOS | Assembly & Embedded Systems Course*
