# RA2L1 评估板基础外设测试与集成自检记录

 CPK-RA2L1 评估板从基础 GPIO 到 UART、ADC/DAC、SPI、I²C，以及最终集成自检程序的配置、接线、代码修改和问题排查过程。

![](picture/Pasted%20image%2020260811172317.png)

## 1. LED1 闪烁测试（GPIO 输出）

![](picture/Pasted%20image%2020260812094222.png)

### 1.1 系统时钟选择

![](picture/Pasted%20image%2020260812100132.png)
![](picture/Pasted%20image%2020260812101255.png)

时钟源选择寄存器 `CKSEL` 的含义如下：

| CKSEL | 时钟源 | 说明 |
| --- | --- | --- |
| `000b` | HOCO | 高速片上振荡器 |
| `001b` | MOCO | 中速片上振荡器 |
| `010b` | LOCO | 低速片上振荡器 |
| `011b` | MOSC | 主时钟振荡器 |
| `100b` | SOSC | 子时钟振荡器 |

当前工程使用 HOCO 作为系统时钟源。

### 1.2 LED 引脚配置

![](picture/Pasted%20image%2020260812103049.png)

将红色 LED1 对应的 `P502` 配置为 GPIO 输出，初始电平设置为低电平。

### 1.3 代码修改说明

- 修改文件：`src/hal_entry.c`。
- 修改位置：`hal_entry()` 函数的用户代码区域。
- 使用 `R_IOPORT_PinWrite()` 控制 `P502` 输出高、低电平。
- 使用 `R_BSP_SoftwareDelay()` 在两次电平切换之间延时1秒。
- 将代码放入 `while (1)`，使 LED 持续闪烁。

```c
while (1)
{
    R_IOPORT_PinWrite(&g_ioport_ctrl,
                      BSP_IO_PORT_05_PIN_02,
                      BSP_IO_LEVEL_HIGH);

    R_BSP_SoftwareDelay(1U, BSP_DELAY_UNITS_SECONDS);

    R_IOPORT_PinWrite(&g_ioport_ctrl,
                      BSP_IO_PORT_05_PIN_02,
                      BSP_IO_LEVEL_LOW);

    R_BSP_SoftwareDelay(1U, BSP_DELAY_UNITS_SECONDS);
}
```

### 1.4 测试现象

程序运行后，评估板红色 LED1 每隔1秒切换一次状态，形成周期闪烁。

## 2. S1 按键控制 LED1（GPIO 轮询方式）

![](picture/Pasted%20image%2020260812104427.png)

### 2.1 按键和 LED 引脚配置

S1 按键连接到 `P004`，按下时接地，因此为低电平有效。红色 LED1 使用 `P502` 输出控制。

![](picture/Pasted%20image%2020260812105733.png)

### 2.2 代码修改说明

- 在 `hal_entry()` 中增加 `bsp_io_level_t button_level`，用于保存按键输入电平。
- 循环调用 `R_IOPORT_PinRead()` 读取 `P004`。
- 检测到低电平时，将 `P502` 输出高电平点亮 LED；否则输出低电平熄灭 LED。
- 每次轮询后延时10毫秒，减少 CPU 占用并抑制部分机械抖动。

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

    R_BSP_SoftwareDelay(10U, BSP_DELAY_UNITS_MILLISECONDS);
}
```

### 2.3 测试现象

按住 S1 时红色 LED1 点亮，松开 S1 后 LED1 熄灭。

## 3. S1 外部中断控制 LED1

![](picture/Pasted%20image%2020260812111240.png)

### 3.1 中断触发方式

选择下降沿触发。S1 松开时 `P004` 被电阻上拉为高电平，按下后接地变为低电平，因此按键按下过程为：

```text
HIGH → LOW = Falling Edge
```

![](picture/Pasted%20image%2020260812114447.png)

### 3.2 External IRQ Stack 配置

![](picture/Pasted%20image%2020260812114559.png)

添加 `External IRQ (r_icu)` Stack，使用 IRQ Channel 3，触发方式选择 Falling Edge，回调函数设置为 `button_irq_callback`。

### 3.3 代码修改说明

- 在 `hal_entry()` 中调用 `R_ICU_ExternalIrqOpen()` 打开 IRQ3。
- 调用 `R_ICU_ExternalIrqEnable()` 允许外部中断。
- 中断回调中只置位标志，不在中断里执行延时或复杂操作。
- 主循环检测到标志后延时20毫秒消抖，再确认 `P004` 是否仍为低电平。
- 每次有效按键操作切换一次 LED 状态，并等待按键释放，避免一次按下触发多次。

```c
volatile bool g_button_pressed = false;

void hal_entry(void)
{
    bsp_io_level_t button_level;
    bsp_io_level_t led_level = BSP_IO_LEVEL_LOW;

    R_ICU_ExternalIrqOpen(&g_external_irq3_ctrl, &g_external_irq3_cfg);
    R_ICU_ExternalIrqEnable(&g_external_irq3_ctrl);

    while (1)
    {
        if (g_button_pressed)
        {
            R_BSP_SoftwareDelay(20U, BSP_DELAY_UNITS_MILLISECONDS);
            R_IOPORT_PinRead(&g_ioport_ctrl,
                             BSP_IO_PORT_00_PIN_04,
                             &button_level);

            if (BSP_IO_LEVEL_LOW == button_level)
            {
                led_level = (BSP_IO_LEVEL_LOW == led_level) ?
                            BSP_IO_LEVEL_HIGH : BSP_IO_LEVEL_LOW;

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

                R_BSP_SoftwareDelay(20U, BSP_DELAY_UNITS_MILLISECONDS);
            }

            g_button_pressed = false;
        }
    }
}
```

### 3.4 中断回调函数

```c
void button_irq_callback(external_irq_callback_args_t * p_args)
{
    if (3U == p_args->channel)
    {
        g_button_pressed = true;
    }
}
```

### 3.5 测试现象

每按下一次 S1，红色 LED1 的亮灭状态切换一次。

## 4. UART 串口通信测试

### 4.1 UART Stack 配置

![](picture/Pasted%20image%2020260812145125.png)

添加 `UART (r_sci_uart)` Stack，选择 SCI0，并配置为 Asynchronous UART（异步 UART）。

![](picture/Pasted%20image%2020260812152114.png)

- Asynchronous UART：不需要独立同步时钟线，常用于 USB-TTL 串口通信。
- Synchronous UART：收发双方使用同步时钟，当前测试不使用。

### 4.2 根据客户管脚分配选择 D 组引脚

![](picture/Pasted%20image%2020260812152710.png)

SCI0 使用：

- `P205/TXD0`：MCU 串口发送。
- `P206/RXD0`：MCU 串口接收。

![](picture/Pasted%20image%2020260812160630.png)

### 4.3 USB-TTL 接线

| USB-TTL | 评估板连接位置 |
| --- | --- |
| RXD | J1-23（P205/TXD0） |
| TXD | J1-22（P206/RXD0） |
| GND | J1-8（GND） |
| VCC/3.3V/5V | 不连接 |

### 4.4 UART 参数配置

![](picture/Pasted%20image%2020260812163828.png)

串口助手设置为：115200 波特率、8位数据位、1位停止位、无校验、无硬件流控。

### 4.5 代码修改说明

- 调用 `R_SCI_UART_Open()` 打开 UART0。
- 使用 `R_SCI_UART_Write()` 发送数据。
- 在 `uart0_callback()` 中处理 `UART_EVENT_TX_COMPLETE`，避免上一帧尚未发送完成就覆盖数据。
- 接收命令或连续数据时使用 `UART_EVENT_RX_CHAR`，每收到一个字符立即在回调中保存。
- 最终集成菜单使用命令缓冲区，遇到 `\r` 或 `\n` 后将一整行命令交给主循环解析。

```c
static volatile bool g_uart_tx_complete = false;
static volatile bool g_command_ready = false;
static volatile uint32_t g_command_length = 0U;
static volatile char g_command_buffer[32];

void uart0_callback(uart_callback_args_t * p_args)
{
    if (UART_EVENT_TX_COMPLETE == p_args->event)
    {
        g_uart_tx_complete = true;
    }
    else if (UART_EVENT_RX_CHAR == p_args->event)
    {
        uint8_t value = (uint8_t) p_args->data;

        if (('\r' == value) || ('\n' == value))
        {
            if (!g_command_ready && (g_command_length > 0U))
            {
                g_command_buffer[g_command_length] = '\0';
                g_command_ready = true;
            }
        }
        else if (!g_command_ready && (g_command_length < 31U))
        {
            g_command_buffer[g_command_length++] = (char) value;
        }
    }
}
```

### 4.6 问题排查：连续发送时数据丢失

#### 4.6.1 出错现象

发送：

```text
AB C1 23 AB C1 23
```

旧程序返回：

```text
AB 23 AB 23
```

#### 4.6.2 问题原因

旧程序每次只调用一次：

```c
R_SCI_UART_Read(..., 1U);
```

收到一个字节后的执行过程为：

```text
收到 AB
→ 停止本次接收
→ 回传 AB
→ 等待发送完成
→ 再次启动接收
```

等待回传期间，电脑已经发送了 `C1`，但 MCU 尚未重新启动接收，因此 `C1` 被漏掉。

#### 4.6.3 解决方法

改在回调中连续接收，不再为每个字节反复启动 `R_SCI_UART_Read()`。如果用于高速连续数据流，可进一步使用64字节或更大的环形缓冲区。

### 4.7 正确测试现象

![](picture/Pasted%20image%2020260812171105.png)

串口能够完整接收并回传连续发送的数据，不再丢失中间字节。

## 5. ADC/DAC 模拟量回环测试

### 5.1 ADC/DAC 引脚连接

| 测试信号 | MCU 端口 | MCU 芯片脚号 | 评估板位置 |
| --- | --- | --- | --- |
| DAC0 输出 | P014 | 53脚 | J2-21 |
| ADC0 输入 | P003/AN003 | 61脚 | J2-29 |

正确连接：

```text
J2-21（P014/DAC0） → J2-29（P003/AN003）
```

### 5.2 ADC Stack 配置

![](picture/Pasted%20image%2020260812174311.png)

添加 `ADC (r_adc)` Stack，模块名称为 `g_adc0`。

### 5.3 ADC 扫描通道配置

![](picture/Pasted%20image%2020260812174543.png)
![](picture/Pasted%20image%2020260812174830.png)

当前 ADC 输入使用 `P003/AN003`，因此 `Channel Scan Mask` 只勾选 `Channel 3`。扫描掩码决定一次 ADC 扫描实际转换哪些模拟通道；未勾选的通道不会被转换。

### 5.4 DAC 引脚和 Stack 配置

![](picture/Pasted%20image%2020260813093327.png)

RA2L1 当前封装只有一路 DAC 输出，固定使用 `P014`。为了避免 DAC 与 ADC 使用同一个引脚，ADC 最终改为直接可用的 `P003/AN003`。

![](picture/Pasted%20image%2020260813094706.png)

### 5.5 ADC 关键参数调整

![](picture/Pasted%20image%2020260813100814.png)

将 ADC 配置为单次扫描，`Clear after read` 设置为 Off，使读取结果后寄存器数据不会立即被清除。该选项只影响 ADC 结果寄存器，与 DAC 是否持续输出无关。

### 5.6 代码修改说明

- 打开 UART0，用于打印每个 DAC 设置值和 ADC 采样值。
- 打开 ADC0，并调用 `R_ADC_ScanCfg()`应用 Channel 3 扫描配置。
- 打开 DAC0，先写入初始值，再调用 `R_DAC_Start()`启动输出。
- 依次输出 `0、512、1024、2048、3072、4095` 六个 DAC 数字量。
- 每次改变 DAC 后延时100毫秒，等待输出电压稳定。
- 调用 `R_ADC_ScanStart()`启动一次采样，并通过 `R_ADC_StatusGet()`等待扫描结束。
- 使用 `R_ADC_Read(..., ADC_CHANNEL_3, ...)`读取 P003 的结果。

```c
R_ADC_Open(&g_adc0_ctrl, &g_adc0_cfg);
R_ADC_ScanCfg(&g_adc0_ctrl, &g_adc0_channel_cfg);
R_DAC_Open(&g_dac0_ctrl, &g_dac0_cfg);
R_DAC_Write(&g_dac0_ctrl, 0U);
R_DAC_Start(&g_dac0_ctrl);

R_DAC_Write(&g_dac0_ctrl, dac_value);
R_BSP_SoftwareDelay(100U, BSP_DELAY_UNITS_MILLISECONDS);
R_ADC_ScanStart(&g_adc0_ctrl);

do
{
    R_ADC_StatusGet(&g_adc0_ctrl, &adc_status);
}
while (ADC_STATE_SCAN_IN_PROGRESS == adc_status.state);

R_ADC_Read(&g_adc0_ctrl, ADC_CHANNEL_3, &adc_value);
```

电压换算方式：

```text
DAC_mV = DAC数字量 × 3300 ÷ 4096
ADC_mV = ADC数字量 × 3300 ÷ 4095
```

### 5.7 串口打印结果

![](picture/Pasted%20image%2020260813104831.png)

### 5.8 问题排查：ADC 采样值不跟随 DAC 变化

#### 5.8.1 出错现象

DAC 输出电压正常变化，万用表测量 J2-21 与 J2-22 的电压也正常，但 ADC 采样值没有跟随 DAC 变化：

```text
DAC=0       ADC约2000
DAC=2048    ADC仍约2000
DAC=4095    ADC仍约2000
```

ADC 结果一直在约1.4～1.8 V之间波动。

#### 5.8.2 问题原因

这不是 ADC 采样频率问题。评估板的 J2-22 虽标为 `P013`，但它与 MCU 的 `P013/AN008` 之间被焊桥 `E12` 隔开；MCU 的 P013 默认通过 `E2` 连接到电容触摸按键 TS2。

![](picture/Pasted%20image%2020260813105633.png)

因此：

```text
P014 → 杜邦线 → J2-22
```

只能把 DAC 电压送到 J2-22，并没有送到 MCU 的 P013。MCU 的 ADC 输入实际处于悬空状态，所以采样值随机波动。相关电路见评估板原理图第1、2、5页。

#### 5.8.3 解决方法

ADC 改用能够直接连接 MCU 的 `P003/AN003`，并将 DAC 输出 J2-21 连接到 ADC 输入 J2-29。修改 Channel Scan Mask，只勾选 Channel 3。修改后 ADC 数字量能够随 DAC 设置值同步变化。

## 6. SPI 主机回环测试

### 6.1 SPI Stack 配置

![](picture/Pasted%20image%2020260813113908.png)

添加 `SPI (r_spi)` Stack，模块名称使用 `g_spi1`，配置为主机、全双工、模式0、MSB First，实际时钟约1 MHz。当前回环测试不使用 DMA，也不使用外部片选。

### 6.2 SPI 接线与信号说明

| SPI 信号 | MCU 引脚 | 芯片管脚号 | 开发板接口 | 当前回环连接 | 作用 |
| --- | --- | --- | --- | --- | --- |
| MOSIB（MOSI） | P109 | 34 | J2-2 | 连接到 J2-3 | SPI 主机发送数据 |
| MISOB（MISO） | P110 | 35 | J2-3 | 连接到 J2-2 | SPI 主机接收数据 |
| RSPCKB（SCK） | P111 | 36 | J2-4 | 不外接 | SPI1 输出同步时钟 |
| FLASH_CS | P301 | 31 | J1-31 | 不外接、暂未配置 | 外部 Flash 片选，低电平有效 |
| GND | VSS | — | 任意可用 GND | 同一块板无需另接 | 电平参考地 |

回环测试只需要：

```text
J2-2（P109/MOSI） → J2-3（P110/MISO）
```

### 6.3 SPI 引脚配置

![](picture/Pasted%20image%2020260813133808.png)

### 6.4 代码修改说明

- 调用 `R_SPI_Open()`打开 SPI1。
- 准备固定的8字节发送数组和同长度接收数组。
- 调用 `R_SPI_WriteRead()`进行一次全双工传输。
- 在 `spi_callback()` 中保存 SPI 事件，并置位传输完成标志。
- 主循环设置超时等待，避免中断异常时永久卡死。
- 传输完成后使用 `memcmp()`比较发送和接收数组，完全一致则判定回环通过。

```c
static volatile bool g_spi_transfer_done = false;
static volatile spi_event_t g_spi_event = SPI_EVENT_TRANSFER_ABORTED;

R_SPI_WriteRead(&g_spi1_ctrl,
                transmit_data,
                receive_data,
                sizeof(transmit_data),
                SPI_BIT_WIDTH_8_BITS);

while (!g_spi_transfer_done)
{
    __NOP();
}

data_matches = (SPI_EVENT_TRANSFER_COMPLETE == g_spi_event) &&
               (0 == memcmp(transmit_data,
                            receive_data,
                            sizeof(transmit_data)));

void spi_callback(spi_callback_args_t * p_args)
{
    g_spi_event = p_args->event;
    g_spi_transfer_done = true;
}
```

### 6.5 测试现象

![](picture/Pasted%20image%2020260813142453.png)

串口打印的 TX 与 RX 数据完全一致，并显示 `SPI LOOPBACK PASS`。

## 7. I²C 主从机回环测试

### 7.1 I²C Master/Slave Stack 配置

![](picture/Pasted%20image%2020260813163010.png)

在同一个工程中添加：

- `I2C Master (r_iic_master)`：IIC0，7位从机地址 `0x50`。
- `I2C Slave (r_iic_slave)`：IIC1，7位本机地址 `0x50`。
- 主机和从机速率均使用 Standard Mode，约100 kHz。

### 7.2 I²C 引脚接线表

| 功能 | MCU 引脚 | 芯片管脚号 | 开发板接口 | 连接到 |
| --- | --- | --- | --- | --- |
| IIC0 主机 SCL | P408/SCL0 | 15 | J1-15，也可用 J14-10 | P100/SCL1 |
| IIC0 主机 SDA | P407/SDA0 | 16 | J1-16，也可用 J14-9 | P101/SDA1 |
| IIC1 从机 SCL | P100/SCL1 | 48 | J2-16，也可用 J14-5 | P408/SCL0 |
| IIC1 从机 SDA | P101/SDA1 | 47 | J2-15，也可用 J14-4 | P407/SDA0 |

使用 J1/J2 时正确连接为：

```text
J1-15（P408/SCL0） → J2-16（P100/SCL1）
J1-16（P407/SDA0） → J2-15（P101/SDA1）
```

### 7.3 I²C 上拉电阻

![](picture/Pasted%20image%2020260813172245.png)

评估板的 P408、P407 侧已经通过 R25、R26 上拉到 VCC。回环连接在同一块评估板上时，不需要重复增加外部上拉电阻。

### 7.4 代码修改说明

- 分别调用 `R_IIC_SLAVE_Open()`和 `R_IIC_MASTER_Open()`打开 IIC1 从机及 IIC0 主机。
- 主机首先发送8字节测试数据，从机在 `I2C_SLAVE_EVENT_RX_REQUEST` 事件中调用 `R_IIC_SLAVE_Read()`准备接收。
- 从机接收完成后，将收到的数据复制到从机发送缓冲区。
- 主机再发起读取，从机在 `I2C_SLAVE_EVENT_TX_REQUEST` 中调用 `R_IIC_SLAVE_Write()`返回相同数据。
- 主机和从机回调分别更新完成、异常及实际字节数标志。
- 等待函数加入1秒超时；即使接线错误或没有应答，也应返回 FAIL，而不是永久阻塞。
- 最后分别比较 `Master TX、Slave RX、Master RX` 三组数据。

```c
R_IIC_SLAVE_Open(&g_i2c_slave1_ctrl, &g_i2c_slave1_cfg);
R_IIC_MASTER_Open(&g_i2c_master0_ctrl, &g_i2c_master0_cfg);

R_IIC_MASTER_Write(&g_i2c_master0_ctrl,
                   master_transmit,
                   sizeof(master_transmit),
                   false);

memcpy(g_i2c_slave_transmit,
       g_i2c_slave_receive,
       sizeof(g_i2c_slave_transmit));

R_IIC_MASTER_Read(&g_i2c_master0_ctrl,
                  master_receive,
                  sizeof(master_receive),
                  false);
```

从机回调中的核心处理：

```c
if (I2C_SLAVE_EVENT_RX_REQUEST == p_args->event)
{
    R_IIC_SLAVE_Read(&g_i2c_slave1_ctrl,
                     g_i2c_slave_receive,
                     sizeof(g_i2c_slave_receive));
}
else if (I2C_SLAVE_EVENT_TX_REQUEST == p_args->event)
{
    R_IIC_SLAVE_Write(&g_i2c_slave1_ctrl,
                      g_i2c_slave_transmit,
                      sizeof(g_i2c_slave_transmit));
}
```

### 7.5 测试现象

![](picture/Pasted%20image%2020260814095355.png)

`Master TX、Slave RX、Master RX` 三组8字节数据一致，串口显示 `I2C LOOPBACK PASS`。

## 8. 所有模块集成自检

### 8.1 集成测试接线汇总

| 测试项目 | 接线 |
| --- | --- |
| UART | P205 → USB串口 RX，P206 → USB串口 TX，GND 共地 |
| ADC/DAC | P014（J2-21）→ P003（J2-29） |
| SPI | P109（J2-2）→ P110（J2-3） |
| I²C 时钟 | P408（J1-15）→ P100（J2-16） |
| I²C 数据 | P407（J1-16）→ P101（J2-15） |

### 8.2 集成程序代码修改说明

- 将 UART、GPIO、ADC/DAC、SPI、I²C 五项测试保留在同一工程中。
- 定义 `NOT TESTED、TESTED - PASS、TESTED - FAIL` 三种状态。
- 上电后通过 UART 显示主菜单及所有测试项目的当前状态。
- 单项测试完成后进入结果页面，输入 `BACK` 返回主菜单并查看更新后的状态。
- `ALL TEST` 按顺序执行五项测试，最后打印汇总结果。
支持的命令：

```text
UART TEST
GPIO TEST
ADC TEST
SPI TEST
I2C TEST
ALL TEST
HELP
BACK
```

状态结构示例：

```c
typedef enum e_test_status
{
    TEST_STATUS_NOT_TESTED = 0,
    TEST_STATUS_PASS,
    TEST_STATUS_FAIL
} test_status_t;

static test_status_t g_test_status[TEST_ID_COUNT];
```

命令分发的核心逻辑：

```c
if (0 == strcmp(command, "UART TEST"))
{
    run_test(TEST_ID_UART);
}
else if (0 == strcmp(command, "GPIO TEST"))
{
    run_test(TEST_ID_GPIO);
}
else if (0 == strcmp(command, "ADC TEST"))
{
    run_test(TEST_ID_ADC_DAC);
}
else if (0 == strcmp(command, "SPI TEST"))
{
    run_test(TEST_ID_SPI);
}
else if (0 == strcmp(command, "I2C TEST"))
{
    run_test(TEST_ID_I2C);
}
else if (0 == strcmp(command, "ALL TEST"))
{
    run_all_tests();
}
```

### 8.3 问题排查：ALL TEST 卡在 I²C

![](picture/Pasted%20image%2020260814111626.png)
![](picture/Pasted%20image%2020260814111802.png)

#### 8.3.1 出错现象

单独运行 ADC/DAC、SPI 和 I²C 均可通过，但执行 `ALL TEST` 时程序进入 I²C 后停止输出。调试器显示程序跳转到无效地址 `0xEFFFFFFE`。

#### 8.3.2 原因：主栈空间不足

![](picture/Pasted%20image%2020260814112441.png)

原 Main Stack Size 只有：

```text
0x400 = 1024 字节
```

执行 `ALL TEST` 时函数调用层次更深，同时叠加 I²C 中断、主从机回调和局部缓冲区，最终造成栈溢出并破坏中断返回地址。

#### 8.3.3 解决方法

在 FSP Configuration 的 BSP 配置中，将 `Main stack size (bytes)` 改为：

```text
0x1000 = 4096 字节
```

随后点击 `Generate Project Content`，重新编译并下载程序。不要直接修改自动生成的 `bsp_cfg.h`，否则下一次生成工程内容时会被覆盖。

### 8.4 正确测试现象

![](picture/Pasted%20image%2020260814135034.png)
![](picture/Pasted%20image%2020260814135024.png)

所有测试执行完成后，UART、GPIO、ADC/DAC、SPI、I²C 均显示 `TESTED - PASS`。输入 `BACK` 可返回主菜单，再次查看各模块的最终状态。

## 9. 总结

本次测试完成了 RA2L1 的 GPIO 输出、按键轮询、外部中断、UART、ADC/DAC、SPI 和 I²C 功能验证，并将各模块整合为可通过串口命令运行的自检程序。调试过程中解决了 UART 连续接收丢字节、ADC 引脚未实际连通以及集成测试主栈不足等问题。最终各模块均可独立测试或通过 `ALL TEST` 顺序执行，测试结果能够在串口菜单中统一显示，为后续增加其他外设和客户板功能验证建立了基础工程。
