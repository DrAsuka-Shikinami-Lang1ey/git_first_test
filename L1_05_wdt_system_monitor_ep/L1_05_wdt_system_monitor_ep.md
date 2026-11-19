# L1_05 窗口看门狗与系统监控说明

>本文主要介绍 R-Car U5L4 工程中窗口看门狗 WDTB 的配置与验证方法。实验通过配置 WDTB 的窗口刷新机制、75% 预警中断和复位功能，实现系统运行状态监控，并通过 LED 现象验证正常喂狗、不喂狗复位以及窗口外喂狗错误检测。


## 背景介绍

>看门狗定时器常用于嵌入式系统的异常恢复。当程序跑飞、死循环或未能按时刷新看门狗时，WDT 会触发系统复位，从而提升系统可靠性。窗口看门狗相比普通看门狗更严格，要求程序必须在指定时间窗口内喂狗，过早或过晚都会被认为是异常。
  本实验使用 WDTB 验证系统监控功能，并使用 LED 显示程序运行和复位现象。
  ![](picture/Pasted%20image%2020260729153637.png)
## 环境说明

- 开发板：R-Car U5L4 开发板

- MCU 型号：R7F711001EBFC

- 开发工具：Visual Studio Code

- 配置工具：Renesas FSP Smart Configurator

- 构建工具：CMake + Ninja

- 编译工具链：GCC 13.3.1

- 调试下载器：Renesas E2

- 主要外设：WDTB、IOPORT

![](picture/Pasted%20image%2020260729144649.png)
## 功能设计

本实验主要实现以下功能：

1. 配置 WDTB 窗口看门狗。
2. 启用 WDT 75% 预警中断。
3. 在正常模式下，收到预警中断后及时喂狗，系统稳定运行。
4. 在不喂狗模式下，验证 WDT 超时复位。
5. 在窗口外喂狗模式下，验证刷新错误导致系统复位。
6. 使用 LED 显示系统启动、正常运行和复位现象。

## 软件设计

本实验主要使用两个外设模块：

- IOPORT：控制 LED 输出。
- WDTB：实现窗口看门狗和系统复位监控。

代码采用结构体封装方式，将 LED、WDT 和应用逻辑分别封装为：

- `LedDev`：LED 设备对象。
- `WDTDev`：WDT 设备对象。
- `WDTApp`：应用逻辑对象。

这样可以让 `hal_entry()` 更简洁，也方便后续扩展。

### 配置方法

#### 1. LED 引脚配置

在 FSP Smart Configurator 的 Pins 页面中，将 `AP0_0` 配置为 GPIO 输出：

| 配置项 | 设置 |
|--|--|
| Mode | Output mode |
| Output Type | Push-Pull |
| Initial Output | Low |
| Input Buffer | Disable |
| Generate data | g_bsp_pin_cfg |

配置完成后点击 `Generate Project Content`。

#### 2. WDTB 配置

在 Stacks 页面中添加 WDT Driver，并进行如下配置：

| 配置项                | 设置              |
| ------------------ | --------------- |
| Name               | g_wdt0          |
| Unit               | Unit 0          |
| Reset Control      | Reset           |
| Window Open Period | 75%             |
| Overflow Time      | 2^16 / WDTBTCKI |
| 75% Interrupt      | Enabled         |
| Callback           | wdt_callback    |
|                    |                 |
## 代码编写

### 1. 测试模式选择

```c
#define WDT_TEST_NORMAL_FEED   (0)
#define WDT_TEST_NO_FEED       (1)
#define WDT_TEST_WINDOW_ERROR  (2)

#define WDT_TEST_MODE WDT_TEST_WINDOW_ERROR
```

通过修改 `WDT_TEST_MODE` 可以切换实验模式。

- `WDT_TEST_NORMAL_FEED`：正常喂狗。
- `WDT_TEST_NO_FEED`：故意不喂狗。
- `WDT_TEST_WINDOW_ERROR`：窗口外喂狗。

### 2. LED 设备封装

```c
struct LedDev
{
    bsp_io_port_pin_t pin;
    bsp_io_level_t level;

    void (*Write)(LedDev * const p_dev, bsp_io_level_t level);
    void (*Toggle)(LedDev * const p_dev);
    void (*Blink)(LedDev * const p_dev, uint32_t times, uint32_t delay_ms);
};
```

该结构体用于保存 LED 引脚、当前电平以及 LED 操作函数。

```c
static LedDev g_led =
{
    .pin    = USER_LED_PIN,
    .level  = BSP_IO_LEVEL_LOW,
    .Write  = LedWrite,
    .Toggle = LedToggle,
    .Blink  = LedBlink,
};
```

### 3. WDT 设备封装

```c
struct WDTDev
{
    const wdt_instance_t * p_instance;
    volatile bool warning_flag;
    volatile uint32_t warning_count;

    fsp_err_t (*Init)(WDTDev * const p_dev);
    fsp_err_t (*Refresh)(WDTDev * const p_dev);
};
```

该结构体用于保存 WDT 实例、预警标志和 WDT 操作函数。

```c
static WDTDev g_wdt_dev =
{
    .p_instance    = &g_wdt0,
    .warning_flag  = false,
    .warning_count = 0,
    .Init          = WDTDevInit,
    .Refresh       = WDTDevRefresh,
};
```

### 4. WDT 回调函数

```c
void wdt_callback(wdt_callback_args_t * p_args)
{
    FSP_PARAMETER_NOT_USED(p_args);

    g_wdt_dev.warning_flag = true;
    g_wdt_dev.warning_count++;
}
```

当 WDT 到达 75% 预警点时，会进入该回调函数。回调中只设置标志位，不直接喂狗，使中断处理尽量简短。

### 5. WDT 初始化和刷新

```c
static fsp_err_t WDTDevInit(WDTDev * p_dev)
{
    fsp_err_t err = p_dev->p_instance->p_api->open(
        p_dev->p_instance->p_ctrl,
        p_dev->p_instance->p_cfg);

    if (FSP_SUCCESS != err)
    {
        return err;
    }

    __enable_irq();

    return p_dev->Refresh(p_dev);
}
```

该函数用于打开 WDT，开启全局中断，并第一次刷新 WDT 以启动计数。

```c
static fsp_err_t WDTDevRefresh(WDTDev * p_dev)
{
    return p_dev->p_instance->p_api->refresh(p_dev->p_instance->p_ctrl);
}
```

该函数用于刷新 WDT，也就是喂狗。
## 注意事项

1. 每次修改 `WDT_TEST_MODE` 后，需要重新编译并下载。
2. 不要直接修改 `rcarmcuu5x_gen` 目录下的自动生成文件。
3. WDT 复位时调试器可能会断开，这是正常现象。
4. WDT 回调函数中不建议执行复杂逻辑，应尽量只设置标志位。
5. LED 快闪 2 下表示系统重新启动，如果反复出现说明发生了 WDT 复位。



