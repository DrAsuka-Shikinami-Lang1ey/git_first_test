# L3-03：TAUD 三相互补 PWM 说明

> 本文主要记录基于 R-Car U5L4 的 TAUD 三相互补 PWM 实验过程。实验通过 TAUD + PIC + TAPA 实现三相六路 PWM 输出、死区时间插入、占空比动态调整、TAPA Hi-Z 紧急关断，以及 ADCK 软件触发采样验证。通过逻辑分析仪观察六路 PWM 波形，并通过外部引脚模拟故障信号验证 PWM 关断功能。

## 背景介绍

三相互补 PWM 常用于电机驱动、逆变器、电源控制等场景。一个三相桥臂通常需要 6 路 PWM 信号，分别控制 U/V/W 三相的高边和低边开关。

为了防止同一桥臂上下管同时导通，需要在互补 PWM 之间加入死区时间。实际电机控制中还需要具备异常保护功能，例如过流时快速关闭 PWM 输出，使功率器件进入安全状态。本实验不连接真实电机，仅使用逻辑分析仪和示波器观察 PWM 波形，验证外设功能是否正常。
![](picture/Pasted%20image%2020260729154845.png)
## 环境说明

- 开发工具：VSCode
- 配置工具：FSP Smart Configurator
- 目标芯片：R-Car MCU U5L4
- 测量工具：逻辑分析仪或示波器
- 使用外设：
  - TAUD：生成三相 PWM
  - PIC：外设互联与 Hi-Z 控制
  - TAPA：三相 PWM 输出、Hi-Z 保护、AD 触发相关功能
  - ADCK：ADC 采样验证

![](picture/Pasted%20image%2020260729154853.png)
本次实验采用了由霆宝科技开发的 R-Car U5L4评估板
## 功能设计

本实验实现以下功能：

1. 生成三相六路互补 PWM。
2. 为互补 PWM 插入死区时间。
3. 通过 TAPA 输出 U/V/W 三相 PWM。
4. 支持占空比动态变化。
5. 通过 TAPA0ESO 外部输入模拟故障，实现 Hi-Z 紧急关断。
6. 使用 ADCK 软件触发采样，验证 ADC 采样流程。

## 硬件连接

| 板子引脚 | 外设功能 | 说明 |
|---|---|---|
| P10_0 | TAPA0UP | U 相高边 PWM |
| P10_1 | TAPA0UN | U 相低边 PWM |
| P10_2 | TAPA0VP | V 相高边 PWM |
| P10_3 | TAPA0VN | V 相低边 PWM |
| P10_4 | TAPA0WP | W 相高边 PWM |
| P10_5 | TAPA0WN | W 相低边 PWM |
| P08_1 | TAPA0ESO | Hi-Z 外部触发输入 |
| AP0_0 | ADCK0I0 | ADC 输入采样引脚 |

验证时，逻辑分析仪连接 P10_0 到 P10_5，观察六路 PWM 输出。
Hi-Z 验证时，P08_1 接 GND 时 PWM 正常输出；当 P08_1 触发有效电平后，六路 PWM 被关闭。

## 软件设计

### 配置方法

#### 1. TAUD 三相 PWM 配置

在 FSP 中添加 Three-Phase PWM Driver on `r_taud_three_phase`，主要配置如下：

| 配置项                  | 设置值           | 说明            |
| -------------------- | ------------- | ------------- |
| TAUD Unit            | Unit 0        | 使用 TAUD0      |
| TAUD Operation Clock | CK0           | 使用 CK0 作为计数时钟 |
| TAUD Clock Prescaler | PCLK/0        | 不分频           |
| Output Level         | Positive      | 正逻辑输出         |
| Period               | 50 us         | PWM 周期        |
| Dead Time            | 10 Raw counts | 死区时间          |
| U/V/W Duty           | 50%           | 初始占空比         |

TAUD 底层会展开多个通道，用于生成三相高低边 PWM，所以在配置界面中会看到多个 `g_timer0_channelx`，这是正常现象。
#### 2. TAPA 配置
TAPA 用于三相 PWM 输出和 Hi-Z 保护。
主要配置如下：

| 配置项 | 设置值 | 说明 |
|---|---|---|
| Asynchronous Hi-Z Control Function | Enable | 开启异步 Hi-Z 保护 |
| Hi-Z Input Edge Selection | Rising Edge | 上升沿触发 Hi-Z |
| Clear Condition Configuration | Stop Hi-Z output only when input is inactive | 输入无效后才允许解除 Hi-Z |
| TAPA0ESO | P08_1 | 外部 Hi-Z 输入引脚 |

#### 3. PIC 配置
PIC 用于连接 TAPA Hi-Z 输入控制。
主要配置：

| 配置项 | 设置值 |
|---|---|
| Enables Hi-Z Control By TAPA0ESO Pin Input | Enable |

#### 4. ADCK 配置

ADC 最终采用软件触发方式进行采样验证。
主要配置如

| 配置项 | 设置值 | 说明 |
|---|---|---|
| Data Format | 12-bit Unsigned Integer Right Justified | 12 位无符号右对齐 |
| Virtual Channel 0 | Physical Channel AN00 | 使用 AP0_0 |
| Scan Group 0 | Enable | 开启扫描组 0 |
| Trigger Method | Software | 软件触发 |
| Callback | adck_callback | ADC 转换完成回调 |

## 代码编写

### 1. 启动 TAUD、TAPA、PIC 和 ADCK

关键流程如下：

```c
g_pic_result = R_PIC_Open(&g_pic_ctrl, &g_pic_cfg);

g_adc_result = R_ADCK_Open(&g_adc0_ctrl, &g_adc0_cfg);
g_adc_result = R_ADCK_ScanCfg(&g_adc0_ctrl, &g_adc0_sg_cfg);
g_adc_result = R_ADCK_ScanGroupStart(&g_adc0_ctrl, ADC_GROUP_MASK_0);

g_taud_result = R_TAUD_THREE_PHASE_Open(&g_taud_three_phase0_ctrl,
                                        &g_taud_three_phase0_cfg);

g_tapa_result = R_TAPA_Start(&g_tapa0_ctrl);

g_taud_result = R_TAUD_THREE_PHASE_Start(&g_taud_three_phase0_ctrl);
```
### 2. 动态修改三相占空比

程序中设置了 4 组占空比模式，并周期性切换：
```c
static const uint8_t duty_patterns[4][3] =
{
    {25U, 50U, 75U},
    {75U, 25U, 50U},
    {50U, 75U, 25U},
    {50U, 50U, 50U}
};
```

通过以下函数更新占空比：

```c
static void TaudThreePhaseSetDuty(uint8_t u_duty, uint8_t v_duty, uint8_t w_duty)
{
    three_phase_duty_cycle_t duty_cycle;

    duty_cycle.duty[THREE_PHASE_CHANNEL_U] = u_duty;
    duty_cycle.duty[THREE_PHASE_CHANNEL_V] = v_duty;
    duty_cycle.duty[THREE_PHASE_CHANNEL_W] = w_duty;

    g_taud_result =
        R_TAUD_THREE_PHASE_DutyCycleSet(&g_taud_three_phase0_ctrl, &duty_cycle);
}
```

### 3. TAPA Hi-Z 状态检测

程序通过读取 TAPA 标志位判断当前是否进入 Hi-Z 状态：

```c
uint32_t tapa_hiz_monitor =
    ((uint32_t) R_TAPA0->FLG >> TAPA_HIZ_MONITOR_SHIFT) & TAPA_HIZ_MONITOR_MASK;

g_tapa_hiz_active = (0U != tapa_hiz_monitor) ? 1U : 0U;
```

当外部输入触发 Hi-Z 后，`g_tapa_hiz_active` 会变为 1，PWM 输出关闭。

### 4. ADC 软件触发采样

程序周期性启动 ADC 扫描：

```c
g_adc_result = R_ADCK_ScanGroupStart(&g_adc0_ctrl, ADC_GROUP_MASK_0);

if (FSP_SUCCESS == g_adc_result)
{
    g_adc_software_trigger_count++;
}
else
{
    g_adc_software_trigger_fail_count++;
    g_adc_error_step = 6U;
}
```

ADC 转换完成后，在回调函数中读取结果：

```c
void adck_callback(adc_callback_args_t * p_args)
{
    uint16_t adc_value = 0U;

    g_adc_callback_count++;

    if (ADC_EVENT_SCAN_COMPLETE_GROUP_0 != p_args->event)
    {
        return;
    }

    g_adc_read_result = R_ADCK_Read(&g_adc0_ctrl, ADC_CHANNEL_0, &adc_value);

    if (FSP_SUCCESS == g_adc_read_result)
    {
        g_adc_last_value = adc_value;
        g_adc_read_count++;
    }
}
```

## 功能验证

### 1. 六路 PWM 输出验证

使用逻辑分析仪连接 P10_0 到 P10_5，可以观察到 6 路 PWM 波形。

验证现象：

- 6 路 PWM 持续输出。
- U/V/W 三相均有对应的高边和低边 PWM。
- PWM 周期符合配置比例。
- 高低边之间存在死区时间。

### 2. 死区时间验证

在逻辑分析仪中测量互补 PWM 的边沿间隔，可以看到高边关闭和低边打开之间存在延迟。

实测死区约为 `7.9 us`，说明死区插入功能有效。

### 3. Hi-Z 关断验证

P08_1 接 GND 时，PWM 正常输出。

当 P08_1 输入有效触发信号后，六路 PWM 被关闭。

关键观察变量：

```c
g_tapa_start_done = 1
g_tapa_error_step = 0
g_tapa_hiz_active = 1
g_tapa_hiz_trigger_count = 1
```

说明 TAPA Hi-Z 功能已经生效。

### 4. ADC 软件触发验证

观察 Watch 变量：

```c
g_adc_open_done = 1
g_adc_scan_cfg_done = 1
g_adc_scan_start_done = 1
g_adc_error_step = 0
g_adc_callback_count 持续增加
g_adc_read_count 持续增加
g_adc_software_trigger_count 持续增加
g_adc_software_trigger_fail_count = 0
```

说明 ADCK 软件触发采样正常。

`g_adc_last_value` 表示 AP0_0 当前输入电压对应的 ADC 采样值。若 AP0_0 接 GND，数值接近 0；若接 3.3V，数值会明显增大。

## 注意事项

1. 逻辑分析仪只能观察数字 PWM 波形，ADC 输入电压变化需要通过 Watch 变量或示波器辅助判断。
2. Hi-Z 验证时不要直接长时间短接 3.3V 和 GND。
3. 如果没有电阻，不建议将 P08_1 同时硬接 GND 和 3.3V。
4. 本实验最终采用软件触发 ADC，保证 ADC 采样流程可验证。
