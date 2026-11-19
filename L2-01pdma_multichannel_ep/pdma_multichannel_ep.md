# pDMAC 功能验证说明

本文主要介绍 R-Car MCU U5L4 工程中 pDMAC 的功能验证过程。实验覆盖了内存到内存、外设到内存、内存到外设三类 DMA 传输方向，并验证了 Single transfer、Block transfer 1、Block transfer 2、多通道并发、链式传输和传输完成中断等功能。最后通过 UART DMA 实现电脑发送 `0xAA`、开发板返回 `0xBB`，完成外设到内存和内存到外设的闭环验证。

## 背景介绍

DMA 用于在不大量占用 CPU 的情况下完成数据搬运。普通方式下，CPU 需要从源地址读取数据，再写入目标地址；使用 DMA 后，只需要由 CPU 配置源地址、目标地址和传输长度，后续搬运过程由 DMA 控制器自动完成。

本项目使用 pDMAC 进行以下验证：

```
内存到内存：数组 A -> DMA -> 数组 B
外设到内存：UART RX -> DMA -> 内存变量
内存到外设：内存变量 -> DMA -> UART TX
```
![](picture/Pasted%20image%2020260729154333.png)
## 环境说明

- 开发工具：VSCode
- 配置工具：FSP Smart Configurator
- 目标芯片：R-Car MCU U5L4
- 使用外设：pDMAC、RLIN3 UART
- 调试方式：GDB 调试、变量监视、串口工具验证
- 串口参数：115200、8 数据位、1 停止位、无校验


![](picture/Pasted%20image%2020260729144704.png)  
  
本次实验采用了由霆宝科技开发的 R-Car U5L4评估板。

## 功能设计

本实验分为两部分：

1. pDMAC 内存到内存验证

验证 DMA 是否能正确把一个数组中的数据搬运到另一个数组中，并分别测试：

```
Single transfer
Block transfer 1
Block transfer 2
多通道并发传输
链式传输
```

2. UART DMA 验证

通过 UART 实现：

```
电脑发送 0xAA
开发板通过 UART RX DMA 接收 0xAA
程序判断接收到的数据
开发板通过 UART TX DMA 返回 0xBB
电脑串口工具接收到 0xBB
```

该实验同时验证：

```
外设到内存：UART RX -> DMA -> rx_data
内存到外设：tx_data -> DMA -> UART TX
```

## 硬件连接

UART 使用开发板对应的串口引脚，通过 USB 转串口或板载调试串口与电脑连接。

|板子接口|外设功能|说明|
|---|---|---|
|UART RX|接收 PC 数据|接收 `0xAA`|
|UART TX|发送数据到 PC|返回 `0xBB`|
|GND|地|与串口工具共地|

## 软件设计

### FSP 配置方法

pDMAC 内存到内存测试中，主要配置 Transfer Driver on r_dmac。

典型配置如下：

```
DMA Mode                  Single / Block transfer 1 / Block transfer 2
Transfer Data Size        1 Byte 或 2 Bytes
Source Address Mode       Incremented
Destination Address Mode  Incremented
Activation Source         SOFTWARE_TRIGGER
Transfer Completion IRQ   Enable
```

UART RX DMA 配置：

```
DMA Mode                  Single transfer
Transfer Data Size        1 Byte
Source Address Mode       Fixed
Destination Address Mode  Incremented
DMA Request Type          Hardware DMA Transfer Request
Number of Transfer        1
Activation Source         RLIN3 RX 触发源
```

UART TX DMA 配置：

```
DMA Mode                  Single transfer
Transfer Data Size        1 Byte
Source Address Mode       Incremented
Destination Address Mode  Fixed
DMA Request Type          Hardware DMA Transfer Request
Number of Transfer        1
Activation Source         RLIN3 TX 触发源
```

UART 本体配置：

```
Operation                 Both TX RX
Transfer API Support      Enabled
Baud Rate                 115200
Data Bits                 8
Parity                    None
Stop Bits                 1
Callback                  user_uart_callback
```

### 关键代码编写

#### 1. UART DMA 测试对象

当前代码将 UART DMA 测试封装成一个对象：

```c
typedef struct st_uart_dma_echo uart_dma_echo_t;

struct st_uart_dma_echo
{
    uart_instance_t const * p_uart;

    uint8_t command;
    uint8_t response;
    uint8_t rx_data;
    uint8_t tx_data;

    volatile uint8_t  rx_complete;
    volatile uint8_t  tx_complete;
    volatile uint8_t  error;
    volatile uint8_t  state;
    volatile int      result;
    volatile uint32_t ok_count;

    fsp_err_t (* Init)(uart_dma_echo_t * const p_dev);
    fsp_err_t (* ReadByte)(uart_dma_echo_t * const p_dev);
    fsp_err_t (* WriteByte)(uart_dma_echo_t * const p_dev);
    void      (* Run)(uart_dma_echo_t * const p_dev);
};
```

其中：

```
command  = 0xAA
response = 0xBB
rx_data  保存接收到的数据
tx_data  保存要发送的数据
```

#### 2. 启动 UART RX DMA 接收

```c
static fsp_err_t UartDmaEchoReadByte(uart_dma_echo_t * const p_dev)
{
    p_dev->state = UART_DMA_STATE_WAIT_RX;

    return p_dev->p_uart->p_api->read(p_dev->p_uart->p_ctrl,
                                      &p_dev->rx_data,
                                      1U);
}
```

该函数启动 UART 接收 1 字节。由于 FSP 已经给 UART RX 挂载 DMA，所以实际数据路径为：

```
UART RX 寄存器 -> pDMAC -> rx_data
```

#### 3. 启动 UART TX DMA 发送

```c
static fsp_err_t UartDmaEchoWriteByte(uart_dma_echo_t * const p_dev)
{
    p_dev->tx_complete = 0U;
    p_dev->state       = UART_DMA_STATE_WAIT_TX;

    return p_dev->p_uart->p_api->write(p_dev->p_uart->p_ctrl,
                                       &p_dev->tx_data,
                                       1U);
}
```

该函数启动 UART 发送 1 字节。实际数据路径为：

```
tx_data -> pDMAC -> UART TX 寄存器
```

#### 4. UART 回调函数


```c
void user_uart_callback(uart_callback_args_t * p_args)
{
    g_uart_dma_echo.Callback(&g_uart_dma_echo, p_args);
}
```

回调中根据事件设置完成标志：

```c
case UART_EVENT_RX_COMPLETE:
{
    p_dev->rx_complete = 1U;
    p_dev->rx_count++;
    break;
}

case UART_EVENT_TX_COMPLETE:
{
    p_dev->tx_complete = 1U;
    p_dev->tx_count++;
    break;
}
```

如果 TX 阶段出现未知事件，也进行兼容处理，避免程序卡死：

```c
default:
{
    if (UART_DMA_STATE_WAIT_TX == p_dev->state)
    {
        p_dev->tx_complete = 1U;
        p_dev->tx_count++;
        p_dev->unexpected_tx_event_count++;
    }
    break;
}
```

#### 5. 主流程

```c
while (1)
{
    UartDmaEchoPrepareRx(p_dev);

    p_dev->ReadByte(p_dev);
    p_dev->WaitRx(p_dev);

    if (p_dev->command != p_dev->rx_data)
    {
        p_dev->wrong_data_count++;
        continue;
    }

    p_dev->WriteByte(p_dev);
    p_dev->WaitTx(p_dev);

    p_dev->ok_count++;
    p_dev->result = 0;
}
```

逻辑为：

```
等待接收 1 字节
如果收到 0xAA
发送 0xBB
成功计数加 1
继续下一轮
```

## 功能验证

### 内存到内存验证

源数组填充为：

```
1, 2, 3 ... 20
```

DMA 传输完成后，目标数组也变为：

```
1, 2, 3 ... 20
```

正确现象：

```
g_dmac_complete = 1
g_dmac_error = 0
g_dmac_result = 0
```

### 多通道并发验证

通道 0 和通道 1 同时传输不同数组。

正确现象：

```
g_dmac0_complete = 1
g_dmac1_complete = 1
g_dmac0_request_count = 1
g_dmac1_request_count = 1
```

### 链式传输验证

只手动启动通道 0，由通道 0 自动触发通道 1。

正确现象：

```
g_dmac0_complete = 1
g_dmac1_complete = 1
g_dmac0_request_count = 1
g_dmac1_request_count = 0
```

其中 `g_dmac1_request_count = 0` 说明通道 1 不是软件启动，而是由 DMA 链式功能自动触发。

### UART DMA 验证

串口工具设置为 HEX 发送。

发送：`AA`

接收：`BB`

调试变量正确现象：

```
g_uart_dma_echo.rx_data = 0xAA
g_uart_dma_echo.tx_data = 0xBB
g_uart_dma_echo.result = 0
g_uart_dma_echo.ok_count 持续增加
g_uart_dma_echo.wrong_data_count = 0
g_uart_dma_echo.error_count = 0
```

## 注意事项

1. 串口工具发送时选择 HEX 模式，只发送：`AA`不要添加 `\r\n`，否则实际发送会变成：`AA 0D 0A`。

2. UART 的 `read()` 和 `write()` 只是启动传输，不代表传输已经完成，必须通过 callback 设置完成标志。
   
3. RX DMA 的源地址应为 Fixed，因为 UART 接收寄存器地址固定。

4. TX DMA 的目标地址应为 Fixed，因为 UART 发送寄存器地址固定。
   
