# MinRTOS-CortexM3

> Tự xây dựng một RTOS kernel tối giản trên STM32F103 (Cortex-M3) — từ việc đọc hiểu FreeRTOS internals đến tự implement từng thành phần.

---

## Mục tiêu

Project này **không phải** là một ứng dụng dùng FreeRTOS API.

Mục tiêu là đi sâu vào **bên trong kernel** — hiểu cách scheduler hoạt động, context switch diễn ra thế nào, queue và semaphore được implement ra sao — rồi tự viết lại từ đầu trên STM32F103.

```
Giai đoạn 1: Hiểu Cortex-M3 exception model
Giai đoạn 2: Đọc & phân tích FreeRTOS kernel source
Giai đoạn 3: Tự implement MinRTOS từ đầu
```

---

## Hardware

| Thành phần | Chi tiết |
|---|---|
| MCU | STM32F103C8T6 (Blue Pill) |
| Kiến trúc | ARM Cortex-M3, 72MHz |
| Flash | 64KB |
| RAM | 20KB |
| Toolchain | STM32CubeIDE / arm-none-eabi-gcc |

---

## Kiến trúc MinRTOS

```
MinRTOS-CM3/
├── kernel/
│   ├── rtos.h              # Public API
│   ├── rtos_task.c         # Task management, TCB, scheduler
│   ├── rtos_queue.c        # Queue — circular buffer + blocking
│   ├── rtos_semaphore.c    # Semaphore & mutex
│   ├── rtos_list.c         # Doubly linked list
│   └── rtos_port.c         # Cortex-M3 port — context switch, SysTick, PendSV
├── notes/
│   ├── 01_cortex_m3_exception_model.md
│   ├── 02_freertos_tasks_analysis.md
│   ├── 03_freertos_queue_analysis.md
│   └── 04_context_switch_deep_dive.md
├── examples/
│   ├── 01_basic_tasks/     # 3 task chạy preemptive
│   ├── 02_task_delay/      # vTaskDelay + blocked list
│   ├── 03_queue/           # producer-consumer qua queue
│   └── 04_mutex/           # bảo vệ tài nguyên chung
└── README.md
```

---

## API

### Task

```c
// Tạo task
RTOS_Status RTOS_TaskCreate(
    TaskFunction_t  func,       // hàm task
    const char*     name,       // tên (debug)
    uint32_t        stackSize,  // size tính bằng words
    void*           param,      // tham số truyền vào task
    uint8_t         priority,   // 0 = thấp nhất
    TaskHandle_t*   handle      // optional
);

// Delay task (ms)
void RTOS_TaskDelay(uint32_t ms);

// Bắt đầu scheduler — không return
void RTOS_Start(void);
```

### Queue

```c
// Tạo queue
QueueHandle_t RTOS_QueueCreate(uint32_t length, uint32_t itemSize);

// Gửi (block nếu full)
RTOS_Status RTOS_QueueSend(QueueHandle_t queue, const void* data, uint32_t timeout_ms);

// Nhận (block nếu empty)
RTOS_Status RTOS_QueueReceive(QueueHandle_t queue, void* buffer, uint32_t timeout_ms);
```

### Semaphore / Mutex

```c
SemaphoreHandle_t RTOS_SemaphoreCreate(uint32_t initialCount, uint32_t maxCount);
SemaphoreHandle_t RTOS_MutexCreate(void);

RTOS_Status RTOS_SemaphoreTake(SemaphoreHandle_t sem, uint32_t timeout_ms);
RTOS_Status RTOS_SemaphoreGive(SemaphoreHandle_t sem);
```

---

## Cách context switch hoạt động trên Cortex-M3

Đây là phần cốt lõi của kernel. Cortex-M3 có hardware hỗ trợ RTOS nên context switch được thiết kế rất gọn:

```
SysTick IRQ (mỗi 1ms)
    └─► set PendSV pending

PendSV IRQ (priority thấp nhất — chạy sau khi mọi IRQ khác xong)
    ├─► Lưu r4-r11 của task hiện tại vào PSP stack
    ├─► Gọi vTaskSwitchContext() → cập nhật pxCurrentTCB
    └─► Khôi phục r4-r11 của task mới từ PSP stack
```

**Tại sao dùng PendSV thay vì switch thẳng trong SysTick?**

Nếu switch trong SysTick, có thể preempt một IRQ khác đang chạy → stack bị corrupt. PendSV luôn chạy sau cùng, đảm bảo an toàn.

**Stack frame khi exception xảy ra (hardware tự lưu):**

```
Địa chỉ cao
┌─────────┐
│   xPSR  │  ← hardware push
│   PC    │  (return address)
│   LR    │
│   r12   │
│   r3    │
│   r2    │
│   r1    │
│   r0    │
├─────────┤  ← PSP trỏ vào đây
│   r11   │  ← software push (context switch)
│   r10   │
│   r9    │
│   r8    │
│   r7    │
│   r6    │
│   r5    │
│   r4    │
└─────────┘
Địa chỉ thấp
```

---

## Task Control Block (TCB)

```c
typedef struct {
    volatile uint32_t*  pxTopOfStack;   // PHẢI là field đầu tiên — port.c dùng offset 0
    ListItem_t          stateListItem;  // node trong ready/blocked list
    ListItem_t          eventListItem;  // node trong event list (queue, semaphore)
    uint8_t             priority;
    uint32_t*           pxStack;        // bottom of stack (để kiểm tra overflow)
    char                name[16];
    uint32_t            ticksToWait;    // dùng cho TaskDelay
} TCB_t;
```

---

## Scheduler — Ready List

```
Priority 3: [Task A] → [Task B]
Priority 2: [Task C]
Priority 1: [Task D] → [Task E]
Priority 0: [Idle Task]
```

Scheduler luôn chọn task có priority cao nhất trong ready list. Cùng priority thì round-robin.

---

## Lộ trình thực hiện

### Giai đoạn 1 — Cortex-M3 fundamentals (2 tuần)

- [ ] Hiểu exception model: SysTick, PendSV, HardFault
- [ ] Phân biệt MSP (Main Stack Pointer) và PSP (Process Stack Pointer)
- [ ] Dùng debugger quan sát stack frame khi exception xảy ra
- [ ] Viết SysTick handler đơn giản, đo thời gian bằng DWT cycle counter

### Giai đoạn 2 — Phân tích FreeRTOS kernel (3 tuần)

- [ ] `list.c` — linked list là nền tảng của mọi thứ
- [ ] `tasks.c` — `xTaskCreate`, `vTaskSwitchContext`, TCB structure
- [ ] `queue.c` — circular buffer, blocking mechanism
- [ ] `port.c` (Cortex-M3) — đọc hiểu ASM context switch
- [ ] Ghi notes vào thư mục `notes/`

### Giai đoạn 3 — Implement MinRTOS (5 tuần)

- [ ] **Tuần 1**: Task creation + stack initialization
- [ ] **Tuần 2**: SysTick + PendSV + context switch — chạy được 2 task preemptive
- [ ] **Tuần 3**: Ready list theo priority + `RTOS_TaskDelay` + blocked list
- [ ] **Tuần 4**: Queue với circular buffer + blocking khi full/empty
- [ ] **Tuần 5**: Mutex + semaphore + stack overflow detection + document

---

## Ghi chú kỹ thuật — những chỗ dễ sai

**1. Stack phải được khởi tạo đúng format Cortex-M3**

Khi `RTOS_TaskCreate` được gọi, stack của task mới phải trông như thể task đó đã bị interrupt một lần. Hardware sẽ pop r0–r3, r12, LR, PC, xPSR khi return từ PendSV. Nếu sai thứ tự → HardFault ngay lần switch đầu tiên.

```c
// Khởi tạo stack frame cho task mới
*(--pxTopOfStack) = 0x01000000;     // xPSR — bit T phải = 1 (Thumb mode)
*(--pxTopOfStack) = (uint32_t)func; // PC — địa chỉ hàm task
*(--pxTopOfStack) = 0xFFFFFFFD;     // LR — EXC_RETURN, return to Thread mode PSP
*(--pxTopOfStack) = 0;              // r12
*(--pxTopOfStack) = 0;              // r3
*(--pxTopOfStack) = 0;              // r2
*(--pxTopOfStack) = 0;              // r1
*(--pxTopOfStack) = (uint32_t)param;// r0 — tham số đầu tiên của task
// Software-saved registers
*(--pxTopOfStack) = 0;              // r11
// ... r10 đến r4
```

**2. PendSV phải có priority thấp nhất**

```c
// Trong SystemInit hoặc RTOS_Start
NVIC_SetPriority(PendSV_IRQn, 0xFF); // thấp nhất
NVIC_SetPriority(SysTick_IRQn, 0x00); // cao nhất
```

**3. Disable interrupt khi access shared data trong kernel**

```c
#define RTOS_ENTER_CRITICAL()  __disable_irq()
#define RTOS_EXIT_CRITICAL()   __enable_irq()
```

---

## Tham khảo

| Tài liệu | Dùng để |
|---|---|
| [FreeRTOS Source Code](https://github.com/FreeRTOS/FreeRTOS-Kernel) | Đọc & phân tích kernel |
| [The Definitive Guide to Cortex-M3/M4 — Joseph Yiu](https://www.amazon.com/Definitive-Guide-Cortex-M3-M4-Processors/dp/0124080820) | Chương 8–9: exception model, stack frame |
| [ARM Cortex-M3 Technical Reference Manual](https://developer.arm.com/documentation/ddi0337/latest) | Tra cứu register, instruction |
| [FreeRTOS Kernel Developer Docs](https://www.freertos.org/Documentation/RTOS_book.html) | Kiến trúc tổng quan |

---

## Tác giả

Học firmware engineer — STM32F103 · FreeRTOS Internals · Cortex-M3

---

*Project này là một phần trong hành trình học Embedded Systems từ bề mặt xuống tận kernel.*
