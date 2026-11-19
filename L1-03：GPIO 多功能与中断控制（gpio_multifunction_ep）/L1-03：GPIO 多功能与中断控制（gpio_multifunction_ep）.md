# L1-03 GPIO 多功能与中断控制实验说明

> 本文档主要说明基于 R-Car U5L4 开发板的 GPIO 多功能与中断控制实验。实验内容包括：使用外部中断检测按键输入并控制 LED 翻转，以及使用 PWGC 输出 PWM 波形，通过示波器验证 Direct 模式下 PWM 由硬件直接输出，不依赖 CPU 在循环中翻转引脚。
  
## 背景介绍

> GPIO 是嵌入式系统中最基础、最常用的外设资源之一，常用于 LED 控制、按键输入、片选信号、状态检测等场景。除了普通输入输出模式外，GPIO 引脚还可以复用为外部中断、定时器输出、串口、I2C、SPI 等外设功能。
   本实验围绕 GPIO 的复用能力进行验证：一方面使用外部中断响应按键动作，避免主循环一直轮询按键状态；另一方面使用 PWGC 外设输出 PWM 波形，验证 PWM 可以由硬件自动输出，CPU 只负责初始化和启动。
![](picture/Pasted%20image%2020260729154003.png)
## 环境说明

- 开发板：R-Car U5L4 开发板

- MCU 型号：R7F711001EBFC

- 开发工具：Visual Studio Code

- 配置工具：Renesas FSP Smart Configurator

- 构建工具：CMake + Ninja

- 编译工具链：GCC 13.3.1

- 调试下载器：Renesas E2

- 验证工具：示波器、薄膜矩阵键盘、杜邦线

![](picture/Pasted%20image%2020260729144658.png)  
  
本次实验采用 R-Car U5L4 开发板，通过 FSP Smart Configurator 完成引脚、外部中断和 PWM 外设配置，再在 `hal_entry.c` 中编写用户代码。

## 功能设计
  
本实验实现以下功能：

1. LED 输出控制  
   将开发板上的 LED4 配置为 GPIO 输出，用于显示按键触发结果。
  
2. 按键外部中断  
   使用薄膜键盘的一个按键作为输入信号，将按键接到 `P00_10`。按键按下时产生下降沿，触发外部中断 `IRQ3`。

3. 按键控制 LED 翻转  
   外部中断回调函数中记录按键触发次数，主循环检测到触发后进行简单消抖，然后翻转 LED 状态。

4. PWM 直接输出  
   使用 `PWGC2` 通过 `P10_2` 输出 PWM 波形，使用示波器验证该波形由硬件直接输出。
  
## 硬件连接

| 板子引脚         | 外设引脚      | 作用     | 备注                 |
| ------------ | --------- | ------ | ------------------ |
| AP0_0 / LED4 | 板载 LED    | LED 输出 | 配置为 GPIO 输出，初始低电平  |
| P00_10       | 薄膜键盘 pin5 | 按键中断输入 | 复用为 `IRQ3`，内部上拉    |
| GND          | 薄膜键盘 pin1 | 按键公共端  | 按键按下时将 `P00_10` 拉低 |
| P10_2        | 示波器探头     | PWM 输出 | 复用为 `PWGC2O`       |
| GND          | 示波器 GND   | 示波器参考地 | 必须与开发板共地           |
![](picture/Pasted%20image%2020260703094044.png)

## 软件设计
  
本实验主要使用以下 FSP 功能：

- IOPORT：配置 LED、按键引脚和 PWM 输出引脚。
- ICU External IRQ：配置 `IRQ3` 外部中断，用于检测按键下降沿。
- PWM-Diag：配置 `PWGC2`，通过 `P10_2` 输出 PWM 波形。
- BSP Delay：用于按键触发后的简单软件消抖。

### 配置方法

#### 1. Pin 配置

在 FSP Smart Configurator 的 `Pins` 页面中配置：

| 引脚       | 配置                                     | 说明       |
| -------- | -------------------------------------- | -------- |
| `AP0_0`  | `GPIO` / Output mode / Initial Low     | LED4 输出  |
| `P00_10` | `IRQ_IRQ3` / Peripheral mode / Pull-up | 按键外部中断输入 |
| `P10_2`  | `PWGC2_PWGC2O` / Peripheral mode       | PWM 输出   |

同时勾选：  

```text

Generate data: g_bsp_pin_cfg

```

这样 FSP 会生成 `g_bsp_pin_cfg`，程序启动时通过 `IOPORT_CFG_OPEN(&IOPORT_CFG_CTRL, &IOPORT_CFG_NAME);` 自动加载引脚配置。

#### 2. 外部中断配置

在 `Stacks` 页面添加：

```text

Input -> External IRQ Driver on r_icu

```

配置如下：

| 项目                     | 设置                      |
| ---------------------- | ----------------------- |
| Name                   | `g_external_irq0`       |
| Channel                | `Channel 3`             |
| Trigger                | `Falling Edge`          |
| Callback               | `external_irq_callback` |
| Pin                    | `P00_10`                |
| Pin Interrupt Priority | `Priority 12`           |

按键一端接 GND，另一端接 `P00_10`。由于 `P00_10` 使用内部上拉，未按下时为高电平，按下时被拉低，因此选择下降沿触发。

#### 3. PWM 配置
在 `Pins` 页面中将 `P10_2` 设置为：

```text

P10_2 -> PWGC2_PWGC2O

```

在 `Stacks` 页面添加：

```text

Timers -> Timer Driver on r_pwm_diag

```

配置如下：

| 项目                      | 设置              |
| ----------------------- | --------------- |
| Name                    | `g_timer0`      |
| Channel                 | `2`             |
| Count Clock Select      | `PWMCLK0`       |
| Period Select           | `PWGC Period 0` |
| PWM Polarity            | `High`          |
| Default Offset          | `0`             |
| Default Duty Cycle      | `0x4000`        |
| Interrupt Factor Enable | `Disabled`      |
  
其中 `Default Duty Cycle = 0x4000` 表示约 50% 占空比。PWM 启动后由 PWGC 硬件自动输出，不需要在 `while(1)` 中反复写 GPIO。
### 代码编写

#### 1. LED 操作函数  
```c

static bsp_io_level_t g_led_level = BSP_IO_LEVEL_LOW;

static void led_write(bsp_io_level_t level)

{

    g_led_level = level;

    R_IOPORT_PinWrite(&g_ioport_ctrl, LED4, g_led_level);

}

  
static void led_toggle(void)

{

    bsp_io_level_t next_level = (BSP_IO_LEVEL_LOW == g_led_level) ? BSP_IO_LEVEL_HIGH : BSP_IO_LEVEL_LOW;

    led_write(next_level);

}

```

`led_write()` 用于设置 LED 电平，同时记录当前 LED 状态。`led_toggle()` 根据当前状态取反，实现 LED 翻转。

#### 2. 外部中断回调函数
```c

enum

{

    KEY_IRQ_CHANNEL = 3U,

    KEY_DEBOUNCE_MS = 50U,

};

  

static volatile uint32_t g_key_irq_count = 0;

  

void external_irq_callback(external_irq_callback_args_t * p_args)

{

    if (KEY_IRQ_CHANNEL == p_args->channel)

    {

        g_key_irq_count++;

    }

}

```

中断回调中不直接翻转 LED，而是只增加计数值。这样可以让中断函数保持简短，真正的 LED 操作放到主循环中完成。

## 功能验证

### 1. 按键控制 LED
验证步骤：
  

1. 编译并下载程序到开发板。

2. 将薄膜键盘 pin1 接 GND，pin5 接 `P00_10`。

3. 按下薄膜键盘对应按键。

4. 观察 LED4 状态是否翻转。

预期现象：

```text

每按下一次按键，LED4 状态翻转一次。

```
### 2. PWM 输出验证

验证步骤：

1. 示波器探头接 `P10_2`。

2. 示波器 GND 接开发板 GND。

3. 下载程序并运行。

4. 在示波器上观察 `P10_2` 输出波形。

预期现象：

```text

P10_2 输出稳定 PWM 方波，电平约为 0 V 到 3.3 V，占空比约为 50%。

```

该波形由 PWGC 硬件自动输出，主循环中没有反复执行 `R_IOPORT_PinWrite()` 控制 `P10_2`，因此可以验证 Direct 模式下 PWM 不经过 CPU 循环翻转引脚。

## 注意事项

1. `P00_10` 配置为按键中断输入时，不能再作为普通 GPIO 输出使用。

2. `P10_2` 已配置为 `PWGC2O`，代码中的 PWM 通道必须对应 `Channel 2`。

3. 按键接线必须共地，否则外部中断可能无法稳定触发。

4. PWM 使用示波器测量时，探头 GND 必须接开发板 GND。

5. 本实验中 PWM 不需要中断，因此 PWM 相关中断保持 `Disabled`。

6. 如果按键没有反应，应优先检查 `P00_10` 是否仍配置为 `IRQ_IRQ3`，以及 `g_external_irq0` 的回调函数是否为 `external_irq_callback`。

7. 如果示波器没有波形，应检查 `P10_2` 是否配置为 `PWGC2_PWGC2O`，以及代码中是否调用了 `g_timer0.p_api->start()`。

  

