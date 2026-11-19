# L3-01 OSTM 操作系统定时器说明

> 本文主要介绍 R-Car U5L4 中 OSTM 定时器的配置与验证方法。实验使用 OSTM0 实现 1ms 周期中断，并基于该中断实现毫秒级延时函数 `DelayMs()`；同时使用 OSTM1 的 Free-running comparison mode 实现微秒级延时函数 `DelayUs()`。最终通过 GPIO 输出波形，使用逻辑分析仪验证 ms/us 延时功能是否正常。

## 背景介绍

OSTM 是 R-Car MCU U5L 系列中的操作系统定时器，可用于周期性中断、系统节拍、延时函数以及时间测量。在嵌入式系统中，系统通常需要一个稳定的时基，例如每 1ms 触发一次中断，用于任务调度、软件定时器或 RTOS tick。

本实验围绕 OSTM 完成两个核心功能：

- 使用 `Interval timer mode` 产生 1ms 周期中断；
- 使用 `Free-running comparison mode` 实现 us 级短延时。

其中 OSTM0 作为系统毫秒时基，OSTM1 作为高速计数器，两者同时运行，互不影响。
![](picture/Pasted%20image%2020260729154525.png)

## 环境说明

- 开发工具：VSCode
- 配置工具：FSP Smart Configurator
- 目标芯片：R-Car MCU U5L4
- 使用外设：OSTM0、OSTM1、GPIO
- 测量工具：逻辑分析仪或示波器


![](picture/Pasted%20image%2020260729154538.png)  
  
本次实验采用了由霆宝科技开发的 R-Car U5L4评估板。
## 功能设计

本实验分为三个阶段：

|阶段|功能|使用外设|
|---|---|---|
|第一阶段|OSTM 周期中断验证|OSTM0|
|第二阶段|1ms 系统 tick + `DelayMs()`|OSTM0|
|第三阶段|Free-running 计数 + `DelayUs()`|OSTM1|

最终设计如下：

|OSTM 通道|模式|作用|
|---|---|---|
|OSTM0 / `g_timer01`|Interval timer mode|1ms 周期中断，作为系统 tick|
|OSTM1 / `g_timer02`|Free-running comparison mode|us 级延时计数|

## 硬件连接

|板子引脚|功能|备注|
|---|---|---|
|AP0_0|GPIO 输出|FSP 中命名为 `LED4`|
|GND|地线|逻辑分析仪/示波器需共地|

测量时将逻辑分析仪或示波器探头接到 `AP0_0`，GND 接开发板 GND。

## 软件设计

### 配置方法

OSTM0 配置如下：

|配置项|设置|
|---|---|
|Name|`g_timer01`|
|Channel|`OSTM0`|
|Mode|`Periodic`|
|Period|`1`|
|Period Unit|`Milliseconds`|
|Callback|`ostm_callback`|
|OSTM Interrupt Priority|`Priority 12`|
|OSTM Mode|`Interval timer mode`|
|Counter Reload|`Enable`|

OSTM1 配置如下：

|配置项|设置|
|---|---|
|Name|`g_timer02`|
|Channel|`OSTM1`|
|Mode|`Periodic`|
|Period|`1000`|
|Period Unit|`Microseconds`|
|Callback|`NULL`|
|OSTM Interrupt Priority|`Disabled`|
|OSTM Mode|`Free-running comparison mode`|
|Counter Reload|`Disable`|

GPIO 配置如下：

|配置项|设置|
|---|---|
|Symbolic Name|`LED4`|
|Pin|`AP0_0`|
|Mode|`Output mode`|
|Output Type|`Push-Pull`|
|Initial Level|Low|

### 代码编写

OSTM0 的回调函数用于产生 1ms 系统 tick：

```c
void ostm_callback(timer_callback_args_t * p_args)
{
    g_ostm_irq_count++;

    if (NULL != p_args)
    {
        g_ostm_last_event = (uint32_t) p_args->event;
    }
}
```

毫秒级延时函数：

```c
static void DelayMs(uint32_t delay_ms)
{
    uint32_t start_tick = g_ostm_irq_count;

    while ((uint32_t) (g_ostm_irq_count - start_tick) < delay_ms)
    {
    }
}
```

微秒级延时函数：

```c
static void DelayUs(uint32_t delay_us)
{
    const uint32_t counts_per_us = 80U;
    uint32_t target_counts = delay_us * counts_per_us;
    uint32_t start_count = Ostm1CounterGet();

    while ((uint32_t) (Ostm1CounterGet() - start_count) < target_counts)
    {
    }
}
```

100us 脉冲输出测试：

```c
static void Pulse100UsTest(void)
{
    R_IOPORT_PinWrite(&g_ioport_ctrl, LED4, BSP_IO_LEVEL_HIGH);

    DelayUs(100U);

    R_IOPORT_PinWrite(&g_ioport_ctrl, LED4, BSP_IO_LEVEL_LOW);

    g_pulse_count++;
}
```

主循环：

```c
while (1)
{
    Pulse100UsTest();
    DelayMs(500U);
}
```

## 功能验证

### 1. OSTM 周期中断验证

将 OSTM0 配置为 1000ms 周期，回调中翻转 GPIO。实际现象为：

- LED 亮约 1s；
- LED 灭约 1s；
- `g_ostm_irq_count` 持续增加。

说明 OSTM0 周期中断工作正常。

### 2. DelayMs 验证

将 OSTM0 改为 1ms 周期中断，回调中累加 `g_ostm_irq_count`。主循环调用：

```
DelayMs(500U);
```

实际现象为：

- LED 约 500ms 翻转一次；
- 完整周期约 1s；
- `g_ostm_irq_count` 持续快速增加。

说明 1ms 系统 tick 和 `DelayMs()` 功能正常。

### 3. DelayUs 验证

使用 OSTM1 的 Free-running comparison mode 实现 `DelayUs()`。测试代码输出 100us 高电平脉冲：

```
DelayUs(100U);
```

逻辑分析仪测得高电平宽度约为：

```
127us
```

随后将参数改为：

```
DelayUs(50U);
```

测得脉冲宽度接近原来的一半，说明 `DelayUs()` 的比例关系正确。

## 注意事项

1. `Interval timer mode` 更适合周期中断和系统 tick。
2. `Free-running comparison mode` 更适合 us 延时和时间测量。
3. `OSTM0` 和 `OSTM1` 不能使用同一个 Channel，否则 FSP 会报 `Duplicate channel`。
