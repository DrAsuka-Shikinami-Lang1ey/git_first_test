# L4-04：PWM-Diag高精度PWM与ADC触发实验说明

> 本文介绍基于R-Car U5L4 MCU的PWM-Diag高级应用。实验利用PWBA、PWGC、PWSD和ADCK生成三路同步PWM，并在PWM周期指定位置自动触发ADC采样。通过示波器验证多通道PWM相位差以及PWM波形与ADC触发位置的关系。

## 背景介绍

在电机控制系统中，三相PWM通常作为功率驱动的基础时序。电流采样必须与PWM保持准确的时间关系，以避开开关噪声并在指定位置采集U、V、W三相电流。

R-Car U5L4的PWM-Diag子系统由以下模块组成：

- PWBA：产生PWGC使用的计数时钟。
- PWGC：生成PWM波形及ADC触发条件。
- PWSD：接收PWGC触发，向ADCK发送转换请求并保存转换结果。
- ADCK：执行ADC转换。
- PIC：提供部分内部外设信号连接功能。

本实验实现三路同频、相差约120°的PWM，并验证PWGC能够在PWM周期的指定位置通过PWSD自动触发ADCK采样。
![](picture/Pasted%20image%2020260729155321.png)
## 环境说明

- 开发工具：VSCode
- 配置工具：FSP Smart Configurator
- 软件平台：
    - RCAR FSP 1.1.1
- 目标芯片：R-Car MCU U5L4（R7F711001EBFC）
- 验证工具：示波器
- 使用外设：
  - PWBA：提供PWMCLK0计数时钟
  - PWGC0～PWGC2：输出三路PWM波形
  - PWGC3：输出ADC触发位置标记脉冲
  - PWSD0：管理PWM触发并连接ADCK
  - ADCK0：采集ANN00模拟输入



![](../L3-07%EF%BC%9AFreeRTOS%20%E9%9B%86%E6%88%90%E4%B8%8E%E5%A4%9A%E4%BB%BB%E5%8A%A1%E8%B0%83%E5%BA%A6%EF%BC%88freertos_integration_ep%EF%BC%89/picture/d6d1fb3a7cf62ae7949e6dfa76ceecbc_compress%281%29.jpg)

本次实验采用了由霆宝科技开发的 R-Car U5L4评估板。

## 功能设计

本实验实现以下功能：

1. 使用PWMCLK0作为PWGC计数时钟。
2. 生成三路16kHz、50%占空比PWM。
3. 三路PWM相位依次偏移约0°、120°和240°。
4. PWGC0、PWGC1和PWGC2分别产生ADC触发。
5. PWSD将触发和通道信息发送给ADCK0。
6. ADCK0采集ANN00，对应AP0_0模拟输入。
7. PWSD将三路转换结果保存在对应的PWDDIR寄存器中。
8. 使用PWGC3在P10_3输出窄脉冲，标记PWGC0的ADC触发位置。
9. PWGC0～PWGC3通过同一次寄存器写入同步启动。

## 硬件连接

|MCU引脚|功能|外部连接|备注|
|---|---|---|---|
|P10_0|PWGC0O|示波器CH1、AP0_0|PWM0及ADC测试信号|
|P10_1|PWGC1O|示波器通道|PWM1|
|P10_2|PWGC2O|示波器通道|PWM2|
|P10_3|PWGC3O|示波器CH2|ADC触发位置标记|
|AP0_0|ANN00|P10_0|ADC模拟输入|
|A0VREFH|ADC参考高电压|3.3V|必须保持稳定|
|GND|公共地|示波器地夹|所有设备共地|


## 软件设计

### PWM-Diag公共配置

|配置项|设置值|
|---|---|
|PWMCLK0 Prescaler|0|
|PWGC Period 0|`0x07CF`|
|PWSD Queue Full Interrupt Priority|Priority 0|
|PWGC Interrupt Factor 0 Priority|Priority 0|
|PWGC Interrupt Factor 1/2 Priority|Disabled|

实际测得PWM频率约为16kHz，周期约为：

```
T = 1 ÷ 16kHz ≈ 62.5μs
```

### PWGC0配置

|配置项|设置值|
|---|---|
|Name|`g_timer0`|
|Channel|Channel 0|
|Count Clock|PWMCLK0|
|Period|PWGC Period 0|
|Polarity|High|
|Offset|`0x0000`|
|Duty Cycle|`0x4000`，即50%|
|Trigger Delay|`0x6000`，即75%|
|ADCK Unit|ADCK0|
|Analog Input|ANN00|
|PWSD Queue Full Interrupt|Enabled|

### PWGC1配置

|配置项|设置值|
|---|---|
|Name|`g_timer1`|
|Channel|Channel 1|
|Offset|`0x029B`|
|Duty Cycle|`0x4000`|
|Trigger Delay|`0x4000`|
|ADCK Unit|ADCK0|
|Analog Input|ANN00|

`0x029B`约为整个周期的三分之一，对应约120°相位偏移。

### PWGC2配置

|配置项|设置值|
|---|---|
|Name|`g_timer2`|
|Channel|Channel 2|
|Offset|`0x0535`|
|Duty Cycle|`0x4000`|
|Trigger Delay|`0x4000`|
|ADCK Unit|ADCK0|
|Analog Input|ANN00|

`0x0535`约为整个周期的三分之二，对应约240°相位偏移。

### PWGC3触发标记配置

|配置项|设置值|
|---|---|
|Name|`g_timer3`|
|Channel|Channel 3|
|Count Clock|PWMCLK0|
|Period|PWGC Period 0|
|Polarity|High|
|Offset|`0x05DB`|
|Duty Cycle|`0x0400`|
|Interrupt Factor|Disabled|
|PWSD Queue Full Interrupt|Enabled|

PWGC0触发延迟为 `0x6000`，对应周期75%。PWGC3的Offset设置为：

```
CTDR = 0x07CF × 0x6000 ÷ 0x8000
     ≈ 0x05DB
```

因此，P10_3窄脉冲的上升沿与PWGC0的ADC触发比较位置一致。

PWGC3占空比为：

```
0x0400 ÷ 0x8000 = 3.125%
```

对应脉宽约为：

```
62.5μs × 3.125% ≈ 1.95μs
```

> PWGC3仅作为示波器标记。程序不调用`R_PWM_DIAG_ADTriggerEnable(g_timer3.p_ctrl)`，因此它不会触发ADC。

### ADCK配置

|配置项|设置值|
|---|---|
|ADC模块|ADCK0|
|PWM-Diag Setting|Enable|
|普通Scan Group 4|Disable|
|Data Format|12-bit Signed Fixed-point|
|Sampling Period|18 States|
|Callback|不用于PWM-Diag完成判断|

PWM-Diag使用专用转换路径。转换完成后，结果返回PWSD并存入：

```
PWSDnPWDDIRx
```

程序使用以下API读取结果：

```c
R_PWM_DIAG_ADConvResultRead(g_timer0.p_ctrl, &result);
R_PWM_DIAG_ADConvResultRead(g_timer1.p_ctrl, &result);
R_PWM_DIAG_ADConvResultRead(g_timer2.p_ctrl, &result);
```

### 程序初始化流程

核心初始化流程如下：

```c
R_ADCK_Open(&g_adc0_ctrl, &g_adc0_cfg);
R_ADCK_ScanCfg(&g_adc0_ctrl, &g_adc0_sg_cfg);
R_ADCK_ScanGroupStart(&g_adc0_ctrl, ADC_GROUP_MASK_PWM_DIAG);

g_timer0.p_api->open(g_timer0.p_ctrl, g_timer0.p_cfg);
g_timer1.p_api->open(g_timer1.p_ctrl, g_timer1.p_cfg);
g_timer2.p_api->open(g_timer2.p_ctrl, g_timer2.p_cfg);
g_timer3.p_api->open(g_timer3.p_ctrl, g_timer3.p_cfg);

R_PWM_DIAG_ADTriggerEnable(g_timer0.p_ctrl);
R_PWM_DIAG_ADTriggerEnable(g_timer1.p_ctrl);
R_PWM_DIAG_ADTriggerEnable(g_timer2.p_ctrl);
```

四个PWGC通道通过一次寄存器写入同步启动：

```c
R_SLPW00->SLPWGC[0].SLPWGC |=
    (1UL << 0) |
    (1UL << 1) |
    (1UL << 2) |
    (1UL << 3);
```

主循环持续读取三路ADC结果：

```c
while (1)
{
    pwm_diag_results_read(g_adc_result_live);
}
```

## ADC结果说明

调试器中观察：

```c
g_adc_result_live[0]
g_adc_result_live[1]
g_adc_result_live[2]
```

由于ADCK使用12位Signed Fixed-point格式，ADC码需要右移3位：

```c
adc_code = raw_result >> 3;
```

电压换算公式：

```c
Vin ≈ (adc_code ÷ 4095) × A0VREFH
```

### 多通道PWM相位差验证

三路PWM频率均约为16kHz，占空比约为50%。

PWM周期：

```
T ≈ 62.5μs
```

相邻两路PWM理论相位差为120°，对应时间差：

```
Δt = 62.5μs × 120° ÷ 360°
   ≈ 20.83μs
```

示波器测得相邻PWM时间差约20.6μs～20.8μs，与理论结果基本一致。

验证结果：

```
PWM0：约0°
PWM1：约120°
PWM2：约240°
```

多通道PWM同步和相位配置正确。

### ADC触发位置功能验证

为了验证ADC确实在指定PWM位置采样，将P10_0连接到AP0_0。

P10_0为50%占空比PWM：

```
0%～50%   ：高电平
50%～100% ：低电平
```

分别设置PWGC0触发延迟：

#### 25%位置测试

```
Trigger Delay = 0x2000
```

ADC在PWM高电平区间采样，结果为：

```
g_adc_result_live = [32760, 8, 32752]
```

其中：

```
g_adc_result_live[0] = 32760
```

表示PWGC0在25%位置采到了约3.3V高电平。

#### 75%位置测试

```
Trigger Delay = 0x6000
```

ADC在PWM低电平区间采样，结果为：

```
g_adc_result_live = [8, 8, 32752]
```

其中：

```
g_adc_result_live[0] = 8
```

表示PWGC0在75%位置采到了约0V低电平。

验证结果证明，修改触发延迟后，ADC采样结果按照预期从高电平变为低电平，因此PWGC→PWSD→ADCK自动触发链路工作正常。

### 示波器观察PWM与ADC触发点关系

示波器连接：

```
CH1 → P10_0，PWM0
CH2 → P10_3，ADC触发位置标记
```

当前Trigger Delay为 `0x6000`，理论触发位置为PWM周期75%：

```
62.5μs × 75% = 46.9μs
```

示波器图中，粉色窄脉冲位于黄色PWM低电平区间中部，符合75%位置。
考虑光标位置误差，测量结果与理论75%一致。

## 注意事项
1. ADC输入电压不得超过A0VREFH允许范围。
2. P10_0连接AP0_0仅用于本实验验证，实际电机控制中应连接真实电流采样信号。
3. P10_3已经配置为PWGC3O，不能继续作为GPIO输出3.3V。


