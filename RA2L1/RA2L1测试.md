![](picture/Pasted%20image%2020260811172317.png)
## 
![](picture/Pasted%20image%2020260812094222.png)
## 时钟选择
![](picture/Pasted%20image%2020260812100132.png)
![](picture/Pasted%20image%2020260812101255.png)
CKSEL = 000b → HOCO
CKSEL = 001b → MOCO
CKSEL = 010b → LOCO
CKSEL = 011b → 主时钟 MOSC
CKSEL = 100b → 子时钟 SOSC
## 引脚设置
![](picture/Pasted%20image%2020260812103049.png)
## 代码编写
```c
 while (1)
    {
        R_IOPORT_PinWrite(&g_ioport_ctrl,
                          BSP_IO_PORT_05_PIN_02,
                          BSP_IO_LEVEL_HIGH);

        R_BSP_SoftwareDelay(1, BSP_DELAY_UNITS_SECONDS);

        R_IOPORT_PinWrite(&g_ioport_ctrl,
                          BSP_IO_PORT_05_PIN_02,
                          BSP_IO_LEVEL_LOW);

        R_BSP_SoftwareDelay(1, BSP_DELAY_UNITS_SECONDS);
    }
```
## 现象






# 按键 S1 控制红色 LED1,轮询方式
![](picture/Pasted%20image%2020260812104427.png)
## 引脚配置
按下低电平，读到低电平LED闪红色
![](picture/Pasted%20image%2020260812105733.png)
## 代码编写
```c
bsp_io_level_t button_level;

  

while (1)

{

  

R_IOPORT_PinRead(&g_ioport_ctrl,

BSP_IO_PORT_00_PIN_04,

&button_level);

  

if (BSP_IO_LEVEL_LOW == button_level)

{

  

R_IOPORT_PinWrite(&g_ioport_ctrl,

BSP_IO_PORT_05_PIN_02,

BSP_IO_LEVEL_HIGH);

}

else

{

  

R_IOPORT_PinWrite(&g_ioport_ctrl,

BSP_IO_PORT_05_PIN_02,

BSP_IO_LEVEL_LOW);

}

  

  

R_BSP_SoftwareDelay(10, BSP_DELAY_UNITS_MILLISECONDS);

}
```
# 外部中断
![](picture/Pasted%20image%2020260812111240.png)



选择下降沿：S1 松开时 `P004` 为高电平，按下后接地变成低电平，即：

```
HIGH → LOW = Falling Edge
```

![](picture/Pasted%20image%2020260812114447.png)
## Stack配置
![](picture/Pasted%20image%2020260812114559.png)
## 代码编写
```c
bsp_io_level_t button_level;

bsp_io_level_t led_level = BSP_IO_LEVEL_LOW;
R_ICU_ExternalIrqOpen(&g_external_irq3_ctrl,

&g_external_irq3_cfg);
R_ICU_ExternalIrqEnable(&g_external_irq3_ctrl);
while (1)

{
if (g_button_pressed)
{
R_BSP_SoftwareDelay(20, BSP_DELAY_UNITS_MILLISECONDS);
R_OPORT_PinRead(&g_ioport_ctrl,
BSP_IO_PORT_00_PIN_04,
&button_level);
if (BSP_IO_LEVEL_LOW == button_level)
{
if (BSP_IO_LEVEL_LOW == led_level)

{
led_level = BSP_IO_LEVEL_HIGH;
}
else
{
led_level = BSP_IO_LEVEL_LOW;

}
R_IOPORT_PinWrite(&g_ioport_ctrl,

BSP_IO_PORT_05_PIN_02,

led_level);

do
{
R_IOPORT_PinRead(&g_ioport_ctrl,
BSP_IO_PORT_00_PIN_04,
&button_level);
}
while (BSP_IO_LEVEL_LOW == button_level);
R_BSP_SoftwareDelay(20,

BSP_DELAY_UNITS_MILLISECONDS);

}

g_button_pressed = false;

}
```
### 中断函数
```c
void button_irq_callback(external_irq_callback_args_t * p_args)
{
if (3U == p_args->channel)
{
g_button_pressed = true;
}
}
```

# UART

## Stack配置
![](picture/Pasted%20image%2020260812145125.png)
## 
![](picture/Pasted%20image%2020260812152114.png)

Asynchronous UART（异步 UART）
Synchronous UART（同步 UART）
### 根据客户要求选D组引脚
![](picture/Pasted%20image%2020260812152710.png)
## 
![](picture/Pasted%20image%2020260812160630.png)

| USB-TTL     | 评估板              |
| ----------- | ---------------- |
| RXD         | J1-23（P205/TXD0） |
| TXD         | J1-22（P206/RXD0） |
| GND         | J1-8（GND）        |
| VCC/3.3V/5V | 不连接              |

## 
![](picture/Pasted%20image%2020260812163828.png)
## 遇到的问题
### 连续发送时`C1`丢失

发送：

```
AB C1 23 AB C1 23
```

旧程序返回：

```
AB 23 AB 23
```

原因：

旧程序每次只调用一次：

```
R_SCI_UART_Read(..., 1U);
```

收到一个字节后，执行顺序是：

```
收到AB
→ 停止本次接收
→ 回传AB
→ 等待发送完成
→ 再次启动接收
```

在等待回传期间，电脑已经发送了`C1`，但MCU尚未重新启动接收，所以`C1`被漏掉。

解决方法：

改用FSP的：

```
UART_EVENT_RX_CHAR
```

不再为每个字节反复调用`R_SCI_UART_Read()`，而是在回调中连续接收数据，并加入64字节环形缓冲区：

```
static uint8_t g_uart_rx_buffer[64];
```


## 现象
![](picture/Pasted%20image%2020260812171105.png)
# ADC数模转换
## 引脚连接
| 测试       | MCU端口      | MCU芯片脚号 | 评估板位置 |
| -------- | ---------- | ------- | ----- |
| DAC输出    | P014       | 53脚     | J2-21 |
| ADC输入    | P003/AN003 | 61脚     | J2-29 |

## Stack配置
![](picture/Pasted%20image%2020260812174311.png)
## ADC Scan通道选择
![](picture/Pasted%20image%2020260812174543.png)
![](picture/Pasted%20image%2020260812174830.png)
## 配置DAC
![](picture/Pasted%20image%2020260813093327.png)
因为DAC只有一路，引脚是P014与之前设置的ADC冲突所以切换一路ADC引脚，AN008 P013
## DAC配置
![](picture/Pasted%20image%2020260813094706.png)
## ADC需要调整的地方
![](picture/Pasted%20image%2020260813100814.png)

## 串口打印现象

![](picture/Pasted%20image%2020260813104831.png)
## 出错现象已经问题解决办法
### 出错现象

DAC输出电压正常变化，万用表测量J2-21和J2-22的电压也正常，但ADC采样值没有跟随DAC变化：

```
DAC=0       ADC约2000
DAC=2048    ADC仍约2000
DAC=4095    ADC仍约2000
```

ADC结果一直在约1.4～1.8V之间波动。

### 问题原因

不是ADC采样频率问题。
评估板上的J2-22虽然标为P013，但它与MCU的P013/AN008之间被焊桥 `E12` 隔开；MCU的P013默认通过 `E2` 接到电容触摸按键TS2。
![](picture/Pasted%20image%2020260813105633.png)
所以：
```
P014 → 杜邦线 → J2-22
```
只能把DAC电压送到J2-22，并没有送到MCU的P013。MCU的ADC输入实际处于悬空状态，因此采样值随机波动。
相关电路见评估板原理图第1、2、5页。
改用直接连接MCU的P003/AN003：
# SPI
## Stack配置
![](picture/Pasted%20image%2020260813113908.png)
## SPI配置
|SPI信号|MCU引脚|芯片管脚号|开发板接口|当前回环连接|作用|
|---|---|---|---|---|---|
|MOSIB（MOSI）|P109|34|J2-2|连接到 J2-3|SPI主机发送数据|
|MISOB（MISO）|P110|35|J2-3|连接到 J2-2|SPI主机接收数据|
|RSPCKB（SCK）|P111|36|J2-4|不外接|SPI1输出同步时钟|
|FLASH_CS|P301|31|J1-31|不外接、暂未配置|外部Flash片选，低电平有效|
|GND|VSS|—|任意可用GND|同一块板无需另接|电平参考地|
## 引脚配置
![](picture/Pasted%20image%2020260813133808.png)
## 现象
![](picture/Pasted%20image%2020260813142453.png)
# I2C
## Stack配置 添加主机和从机
![](picture/Pasted%20image%2020260813163010.png)

## 引脚接线表
|功能|MCU引脚|芯片管脚号|开发板接口|连接到|
|---|---|---|---|---|
|IIC0主机SCL|P408/SCL0|15|J1-15，也可用J14-10|P100/SCL1|
|IIC0主机SDA|P407/SDA0|16|J1-16，也可用J14-9|P101/SDA1|
|IIC1从机SCL|P100/SCL1|48|J2-16，也可用J14-5|P408/SCL0|
|IIC1从机SDA|P101/SDA1|47|J2-15，也可用J14-4|P407/SDA0|
## 已经接了上拉电阻
![](picture/Pasted%20image%2020260813172245.png)
## 现象
![](picture/Pasted%20image%2020260814095355.png)
# 整合所有的测试
# 接线表格
| 测试      | 接线                              |
| ------- | ------------------------------- |
| UART    | P205→USB串口RX，P206→USB串口TX，GND共地 |
| ADC/DAC | P014与P003连接                     |
| SPI     | P109与P110连接                     |
| I²C时钟   | P408与P100连接                     |
| I²C数据   | P407与P101连接                     |
## 问题
![](picture/Pasted%20image%2020260814111626.png)
![](picture/Pasted%20image%2020260814111802.png)
### 栈溢出
![](picture/Pasted%20image%2020260814112441.png)

原来只有：

```
0x400 = 1024 字节
```

运行 `ALL TEST` 时调用层次更深，又叠加 I²C 中断和回调，造成栈溢出，最终跳到异常地址 `0xEFFFFFFE`。

改成：

```
0x1000 = 4096 字节
```
## 正确现象
![](picture/Pasted%20image%2020260814135034.png)
![](picture/Pasted%20image%2020260814135024.png)