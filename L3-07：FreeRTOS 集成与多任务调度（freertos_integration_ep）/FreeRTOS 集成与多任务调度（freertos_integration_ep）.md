# L3-07：R-Car U5L4 FreeRTOS 集成与多任务调度说明

> 本文介绍在 R-Car U5L4 上集成 FreeRTOS，并实现多任务并发调度、任务状态队列通信和串口日志输出的方法。工程创建三个相同优先级的周期工作任务，各任务通过 FreeRTOS Queue 上报执行状态，由独立日志任务统一使用 RLIN3 UART 输出。UART 发送采用中断完成回调和二值信号量同步，避免多个任务同时访问串口造成输出交叉或驱动忙错误。

## 背景介绍

在嵌入式系统中，采集、通信、控制和状态监测等功能通常需要同时运行。如果使用单一主循环依次执行所有功能，某个耗时操作可能阻塞其他功能，导致系统响应不及时。

FreeRTOS 将系统功能划分为多个独立任务，并根据任务优先级、阻塞状态和系统 Tick 进行调度。本工程在 R-Car U5L4 上创建三个周期工作任务，模拟不同执行周期的业务模块：

|任务|执行周期|功能|
|---|---|---|
|TaskA|1000 ms|模拟周期任务 A|
|TaskB|1500 ms|模拟周期任务 B|
|TaskC|2000 ms|模拟周期任务 C|
|UART Logger|事件驱动|接收状态消息并通过串口输出|

三个工作任务不直接操作 UART，而是通过 Queue 将任务名称、运行状态和执行次数发送给日志任务。日志任务是唯一访问 UART 的任务，从而保证串口输出完整、有序。
![](picture/Pasted%20image%2020260729155131.png)
## 环境说明

- 开发工具：VSCode
- 配置工具：FSP Smart Configurator
- 软件平台：
    - RCAR FSP 1.1.1
    - FreeRTOS Kernel V11.2.0
- 目标芯片：R-Car MCU U5L4（R7F711001EBFC）
- 验证工具：万用表
- 串口配置：115200 波特率、8 数据位、1 停止位、无校验、无流控
- 使用外设：
    - SysTick：为 FreeRTOS 提供 1 ms 系统 Tick
    - RLIN3：通过 RLIN30 UART 输出各任务执行状态


![](picture/d6d1fb3a7cf62ae7949e6dfa76ceecbc_compress%281%29.jpg)

本次实验采用了由霆宝科技开发的 R-Car U5L4评估板。

## 功能设计

本工程实现以下功能：

1. 在 R-Car U5L4 上启动 FreeRTOS 调度器。
2. 创建三个相同优先级的周期工作任务。
3. 使用 `vTaskDelayUntil()` 控制任务周期。
4. 使用 Queue 在工作任务与日志任务之间传递状态。
5. 使用 RLIN3 UART 输出任务执行状态。
6. 使用二值信号量等待 UART 发送完成。
7. 使用 UART TXI 中断回调通知日志任务。
8. 记录任务创建失败、Queue 丢包和 UART 超时等诊断信息。
9. 启用抢占式调度和同优先级时间片调度。


## 硬件连接

RLIN30 使用以下引脚：

|板子引脚|外设功能|连接目标|备注|
|---|---|---|---|
|P00_2|RLIN30TX|USB 转串口 RX|串口发送引脚|
|P00_3|RLIN30RX|USB 转串口 TX|当前程序未使用接收|
|GND|GND|USB 转串口 GND|必须共地|

## 软件设计

### FreeRTOS 配置

工程在 Smart Configurator 中添加以下组件：

- FreeRTOS
- FreeRTOS Port（`rm_freertos_port`）
- FreeRTOS Heap 4
- UART Driver on `r_rlin3_uart`
- I/O Port

FreeRTOS 主要配置如下：

|配置项|设置值|说明|
|---|---|---|
|Use Preemption|Enabled|启用抢占调度|
|Use Time Slicing|Enabled|启用同优先级时间片轮转|
|Tick Rate|1000 Hz|1 Tick 等于 1 ms|
|Max Priorities|5|有效优先级为 0～4|
|Use Tickless Idle|Disabled|当前多任务验证阶段关闭|
|Support Static Allocation|Enabled|支持静态创建任务|
|Support Dynamic Allocation|Enabled|支持动态创建任务和内核对象|
|Total Heap Size|16384 字节|为动态任务、Queue、Semaphore 提供空间|
|Use Mutexes|Enabled|支持互斥量|
|Use Task Notifications|Enabled|支持任务通知|
|Use Counting Semaphores|Enabled|支持计数信号量|

### 日志线程配置

Smart Configurator 生成的 `new_thread0` 作为 UART 日志任务：

|配置项|设置值|
|---|---|
|Symbol|`new_thread0`|
|Stack Size|4096 字节|
|Priority|3|
|Memory Allocation|Static|
|Allocate Secure Context|Disabled|

日志任务优先级高于三个工作任务。当工作任务向 Queue 写入状态后，日志任务被唤醒并立即处理消息；Queue 为空时，日志任务阻塞，不会持续占用 CPU。

### 工作任务配置

三个工作任务通过 `xTaskCreate()` 动态创建：

```
#define WORKER_STACK_DEPTH_WORDS    (512U)
#define WORKER_PRIORITY             (2U)
```

在 Cortex-M33 上，`StackType_t` 通常为 4 字节，因此每个工作任务的栈空间约为：

```
512 × 4 = 2048 字节
```

三个工作任务共使用约 6144 字节动态 Heap，剩余 Heap 用于任务控制块、Queue、Semaphore 和内存管理开销。

### UART 配置

RLIN3 UART 配置如下：

|配置项|设置值|
|---|---|
|实例名称|`g_uart0`|
|通道|RLIN30 / Channel 0|
|波特率|115200|
|数据位|8 bit|
|停止位|1 bit|
|校验|None|
|工作方向|TX/RX|
|Callback|`user_uart_callback`|
|RXI Priority|Disabled|
|TXI Priority|Priority 12|
|ERI Priority|Priority 12|

TXI 中断必须启用。RLIN3 驱动使用 TXI 中断发送后续字节，并在全部数据发送完成后产生 `UART_EVENT_TX_COMPLETE` 事件。

### Tick 来源说明

当前工程使用 FreeRTOS Cortex-M33 端口的默认实现，系统 Tick 实际由 Cortex-M SysTick 提供：

```
void vPortSetupTimerInterrupt(void);
void SysTick_Handler(void);
```

当前配置为：

```
#define configTICK_RATE_HZ    1000
```

因此系统每 1 ms 产生一次 Tick。

## 代码编写

### 任务状态结构

任务状态消息包含任务名称、运行状态和执行次数：

```c
typedef enum e_task_run_state
{
    TASK_RUN_STATE_STARTED = 0,
    TASK_RUN_STATE_RUNNING,
    TASK_RUN_STATE_CREATE_FAILED
} task_run_state_t;

typedef struct st_task_status_message
{
    char const      * p_task_name;
    task_run_state_t  state;
    uint32_t          run_count;
} task_status_message_t;
```

任务状态包括：

|状态|含义|
|---|---|
|`STARTED`|任务已经成功启动|
|`RUNNING`|任务完成一次周期运行|
|`CREATE_FAILED`|`xTaskCreate()` 创建任务失败|

### 任务周期配置

三个任务共用同一个入口函数，通过参数指定名称和周期：

```c
static const worker_configuration_t g_task_a_configuration =
{
    .p_task_name  = "TaskA",
    .period_ticks = pdMS_TO_TICKS(1000U)
};

static const worker_configuration_t g_task_b_configuration =
{
    .p_task_name  = "TaskB",
    .period_ticks = pdMS_TO_TICKS(1500U)
};

static const worker_configuration_t g_task_c_configuration =
{
    .p_task_name  = "TaskC",
    .period_ticks = pdMS_TO_TICKS(2000U)
};
```

使用统一入口可以减少重复代码。后续只需要修改配置结构即可增加不同周期的任务。

### Queue 状态上报

工作任务调用 `status_publish()` 上报状态：

```c
static void status_publish (char const * p_task_name,
                            task_run_state_t state,
                            uint32_t run_count)
{
    task_status_message_t message =
    {
        .p_task_name = p_task_name,
        .state       = state,
        .run_count   = run_count
    };

    if (pdTRUE != xQueueSend(g_status_queue,
                             &message,
                             pdMS_TO_TICKS(100U)))
    {
        g_status_drop_count++;
    }
}
```

如果 Queue 在指定时间内没有空位，消息发送失败，程序增加 `g_status_drop_count`，便于调试器观察。

### 周期工作任务

三个任务使用相同入口：

```c
static void worker_task_entry (void * pvParameters)
{
    worker_configuration_t const * p_configuration =
        (worker_configuration_t const *) pvParameters;

    TickType_t last_wake_time = xTaskGetTickCount();
    uint32_t run_count = 0U;

    status_publish(p_configuration->p_task_name,
                   TASK_RUN_STATE_STARTED,
                   run_count);

    while (1)
    {
        vTaskDelayUntil(&last_wake_time,
                        p_configuration->period_ticks);

        /* 在这里添加任务的实际业务代码。 */
        run_count++;

        status_publish(p_configuration->p_task_name,
                       TASK_RUN_STATE_RUNNING,
                       run_count);
    }
}
```

这里使用 `vTaskDelayUntil()`，而不是普通 `vTaskDelay()`。

`vTaskDelayUntil()` 根据上一次计划唤醒时间计算下一次唤醒时刻，可以减小任务执行时间造成的周期累积误差，更适合周期性任务。

### UART 发送完成回调

![](../R7F71001/picture/Pasted%20image%2020260714175754.png)
![](../R7F71001/picture/Pasted%20image%2020260714180058.png)
遇到的问题：heap不足
把10240换成16384

RLIN3 UART 发送完成后进入回调函数：

```c
void user_uart_callback (uart_callback_args_t * p_args)
{
    BaseType_t higher_priority_task_woken = pdFALSE;

    if (NULL == p_args)
    {
        return;
    }

    g_uart_last_event = (uint32_t) p_args->event;

    if ((UART_EVENT_TX_COMPLETE == p_args->event) &&
        (NULL != g_uart_tx_done_semaphore))
    {
        xSemaphoreGiveFromISR(g_uart_tx_done_semaphore,
                              &higher_priority_task_woken);

        portYIELD_FROM_ISR(higher_priority_task_woken);
    }
}
```

### UART 同步发送

UART 驱动的 `write()` 接口是异步发送。日志任务调用发送接口后，通过二值信号量等待发送完成：

```c
static BaseType_t uart_write_blocking (uint8_t const * p_data,
                                       uint32_t data_length)
{
    fsp_err_t err;

    xSemaphoreTake(g_uart_tx_done_semaphore, 0U);

    err = g_uart0.p_api->write(g_uart0.p_ctrl,
                               p_data,
                               data_length);

    g_uart_last_error = err;

    if (FSP_SUCCESS != err)
    {
        return pdFAIL;
    }

    if (pdTRUE != xSemaphoreTake(g_uart_tx_done_semaphore,
                                 pdMS_TO_TICKS(1000U)))
    {
        g_uart_last_error = FSP_ERR_TIMEOUT;
        g_uart_timeout_count++;

        return pdFAIL;
    }

    return pdPASS;
}
```

这种设计保证发送缓冲区在 UART 发送完成前保持有效，同时防止上一条消息尚未发送完成时启动下一次发送。


## 时间片调度说明

工程配置中已经启用：

```
#define configUSE_PREEMPTION      1
#define configUSE_TIME_SLICING    1
```

TaskA、TaskB 和 TaskC 的优先级均为 2，因此当多个任务同时处于 Ready 状态时，FreeRTOS 可以在系统 Tick 到来时对它们进行时间片轮转。

不过，当前三个任务执行少量代码后会调用 `vTaskDelayUntil()` 进入阻塞状态，因此当前串口输出主要验证的是：

- 多任务创建；
- 周期任务唤醒；
- 抢占调度；
- Queue 通信；
- UART 中断同步。

如果要严格验证时间片轮转，应创建两个相同优先级、持续运行且不调用延时函数的测试任务。两个任务分别递增独立计数器，再由监控任务低频打印计数结果。两个计数器都持续增长且数值接近，即可证明同优先级任务正在轮流使用 CPU。

## 功能验证

### 串口设置

串口工具设置为：

```
端口：COM4
波特率：115200
数据位：8
停止位：1
校验：无
流控：无
编码：UTF-8 或 ASCII
```

### 实际输出

程序启动后串口输出如下：

```
FreeRTOS concurrent task status demo started
[tick=3] TaskA STARTED       count=0
[tick=6] TaskB STARTED       count=0
[tick=9] TaskC STARTED       count=0
[tick=1003] TaskA RUNNING    count=1
[tick=1503] TaskB RUNNING    count=1
[tick=2003] TaskC RUNNING    count=1
[tick=2006] TaskA RUNNING    count=2
[tick=3003] TaskB RUNNING    count=2
[tick=3006] TaskA RUNNING    count=3
[tick=4003] TaskC RUNNING    count=2
[tick=4006] TaskA RUNNING    count=4
[tick=4503] TaskB RUNNING    count=3
```

从输出可以确认：

1. TaskA、TaskB、TaskC 均成功创建。
2. 三个任务首先输出 `STARTED` 状态。
3. TaskA 约每 1000 Tick 执行一次。
4. TaskB 约每 1500 Tick 执行一次。
5. TaskC 约每 2000 Tick 执行一次。
6. 各任务计数独立增长。
7. 串口输出没有交叉、乱码或丢失。
8. UART TXI 中断和发送完成信号量工作正常。

### Tick 偏差分析

输出中可能出现：

```
[tick=2003] TaskC RUNNING count=1
[tick=2006] TaskA RUNNING count=2
```

这不表示 TaskA 的周期发生错误。

TaskA 和 TaskC 可能在同一个 Tick 附近同时将消息写入 Queue，日志任务只能逐条输出。115200 波特率发送一行文本需要数毫秒，因此第二条日志显示的 Tick 会稍晚。

当前日志中的 Tick 是日志任务处理 Queue 消息时调用 `xTaskGetTickCount()` 得到的时间，并不是工作任务产生消息时的精确时间。


## 注意事项

1. UART 的 TXI 中断不能设为 Disabled，否则驱动无法正常发送后续字节，也不会产生 `UART_EVENT_TX_COMPLETE`。
    
2. UART TXI 优先级为 12，可以在回调中使用 FreeRTOS 的 `FromISR` 接口。中断优先级必须满足 FreeRTOS 最大系统调用中断优先级限制。
           
3. `new_thread0` 由 Smart Configurator 静态创建，它的 4096 字节栈不占用 FreeRTOS 的 16384 字节动态 Heap。
    
4. 如果后续任务增加大型局部数组、浮点格式化或深层函数调用，需要重新评估任务栈大小。
    
5. 如果启用 Tickless Idle，应重新验证串口中断唤醒、任务周期和低功耗状态下的时钟配置。
