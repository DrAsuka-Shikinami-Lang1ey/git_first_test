本文介绍在 R-Car MCU U5L4 上使用 CSIH 实现 SPI 主机通信。实验采用单板物理环回，将 CSIH0SO 与 CSIH0SI 短接，依次验证直接访问、FIFO 和 DMA 传输。最终完成 1024 字节 × 1000 轮 DMA 压力测试，累计校验 1,024,000 字节，无传输失败及数据不一致。

## 背景介绍

SPI 是一种同步串行通信接口，常用于 MCU 与 Flash、传感器、ADC 和通信模块之间的数据交换。

R-Car MCU 的 CSIH（Clocked Serial Interface H）支持主机/从机、多个片选、直接访问、FIFO 和 DMA 联动。本项目重点验证：

- CSIH 主机模式通信
- SPI 时钟及数据采样配置
- 直接访问模式
- FIFO 缓冲传输
- DMA 收发联动
- 连续传输和数据一致性检查
- 时钟、数据和片选波形测量
- ![](picture/Pasted%20image%2020260729155906.png)

## 环境说明

- 开发工具：VS Code
- 配置工具：FSP Smart Configurator
- 软件平台：
    - RCAR FSP 1.1.1
- 目标芯片：R-Car MCU U5L4（R7F711001EBFC）
- 验证工具：
    - 示波器
    - ATK-Logic 逻辑分析仪
    - 调试器
- 使用外设：
    - CSIH0：SPI 时钟及全双工收发
    - DMAC 通道0：CSIH0 发送 DMA
    - DMAC 通道1：CSIH0 接收 DMA


![](../L3-07%EF%BC%9AFreeRTOS%20%E9%9B%86%E6%88%90%E4%B8%8E%E5%A4%9A%E4%BB%BB%E5%8A%A1%E8%B0%83%E5%BA%A6%EF%BC%88freertos_integration_ep%EF%BC%89/picture/d6d1fb3a7cf62ae7949e6dfa76ceecbc_compress%281%29.jpg)

本次实验采用了由霆宝科技开发的 R-Car U5L4评估板。

## 功能设计

程序分为三个验证阶段：

1. 直接访问环回
    
    发送 `0xAA`、`0xBB`，验证基础时钟、引脚和收发流程。
    
2. FIFO 环回
    
    使用 FIFO 连续传输64字节，等待接收完成事件并逐字节比较。
    
3. DMA 环回及压力测试
    
    - 先进行64字节 DMA 环回；
    - 再进行1024字节 × 1000轮压力测试；
    - 每轮生成不同的数据；
    - 同时等待 TX DMA 和 RX DMA 完成；
    - 每轮逐字节比较发送和接收缓冲区；
    - 记录成功次数、失败轮次和错误位置。

## 硬件连接

### CSIH0 引脚

|MCU引脚|CSIH0功能|方向|驱动能力|用途|
|---|---|---|---|---|
|P00_1|CSIH0SI|输入|Low|SPI接收数据|
|P00_2|CSIH0SC|输出|Medium|SPI时钟|
|P00_3|CSIH0SO|输出|Medium|SPI发送数据|
|P08_0|CSIH0CSS0|输出|High|低有效片选CS0|

### 环回连接

|板端信号|连接对象|说明|
|---|---|---|
|P00_3 / CSIH0SO|P00_1 / CSIH0SI|使用杜邦线短接|
|P00_2 / CSIH0SC|逻辑分析仪CLK|测量SPI时钟|
|P00_1 / CSIH0SI|逻辑分析仪MISO|解码环回数据|
|P08_0 / CSIH0CSS0|逻辑分析仪CS#|观察低有效片选|
|板端GND|逻辑分析仪GND|共地|

片选引脚不需要使用跳线帽短接。环回测试只需将 CSIH0SO 与 CSIH0SI 短接。

## 软件设计

### CSIH0公共配置

|配置项|设置|
|---|---|
|通道|CSIH0|
|工作模式|Master mode|
|传输方向|TX/RX|
|Job mode|Disabled|
|Handshake|Disabled|
|内部Loop-back|Disabled|
|Buffer Memory|根据测试阶段选择Direct access或FIFO|
|片选|CS0，低有效|
|波特率|1,000,000 Hz|
|时钟空闲电平|Low|
|数据采样|奇数边沿采样、偶数边沿变化|
|SPI模式|Mode 0（CPOL=0、CPHA=0）|
|位序|MSB first|
|回调函数|`csih_callback`|
|通信错误中断|Priority 14|

当前工程的实际波特率是1 MHz。

### DMA配置

使用 DMA 时：

- CSIH Buffer Memory 设置为 `Direct access mode`
- 启用 `Enable Support for using a transfer API`
- CSIH状态发送和接收中断关闭
- 数据搬运请求交给 DMAC

#### TX DMA：g_transfer0

|配置项|设置|
|---|---|
|DMAC通道|0|
|DMA模式|Single transfer|
|数据宽度|1 Byte|
|源地址|Incremented|
|目标地址|Fixed|
|请求类型|Hardware DMA Transfer Request|
|触发源|CSIH_INTCSIHTIC_0|
|完成中断|Enable，Priority 12|
|回调|csih_tx_transfer_callback|

#### RX DMA：g_transfer1

|配置项|设置|
|---|---|
|DMAC通道|1|
|DMA模式|Single transfer|
|数据宽度|1 Byte|
|源地址|Fixed|
|目标地址|Incremented|
|请求类型|Hardware DMA Transfer Request|
|触发源|CSIH_INTCSIHTIR_0|
|完成中断|Enable，Priority 12|
|回调|csih_rx_transfer_callback|

TX DMA 的源地址递增，是因为 DMAC 需要依次读取发送数组；目标地址固定，是因为所有数据都写入 CSIH 发送寄存器。

RX DMA 的源地址固定在 CSIH 接收寄存器，目标地址在接收数组中逐字节递增。


### DMA压力测试

测试参数为：

```
#define CSIH_DMA_DATA_LENGTH (1024U)
#define CSIH_DMA_TEST_ROUNDS (1000U)
```

每轮根据数组位置和当前轮次生成不同测试数据：

```
g_tx_data[i] =
    (uint8_t) (((i * 17U) +
                (round * 29U) +
                0x5AU) & 0xFFU);
```

随后调用：

```
R_CSIH_WriteRead(&g_csih0_ctrl,
                 g_tx_data,
                 (void *) g_rx_data,
                 CSIH_DMA_DATA_LENGTH,
                 SPI_BIT_WIDTH_8_BITS);
```

回调函数分别记录 TX DMA 和 RX DMA 完成：

```
if (SPI_EVENT_TRANSFER_COMPLETE == p_args->event)
{
    g_dma_tx_complete = 1U;
}
else if (SPI_EVENT_RECEIVE_COMPLETE == p_args->event)
{
    g_dma_rx_complete = 1U;
}
else
{
    g_dma_transfer_error = 1U;
}
```

程序等待两个方向都完成，再比较1024字节数据。任何一轮出现 API 错误、超时、回调错误或数据不一致时，程序都会停止并记录错误位置。

## 功能验证

### 直接访问环回

直接访问阶段发送 `0xAA`、`0xBB`，接收数据与发送数据一致。

### FIFO环回

FIFO模式完成64字节环回：

- 接收完成标志为1
- 传输错误为0
- 没有数据不一致
- CSIH返回 `FSP_SUCCESS`

逻辑分析仪能够观察到连续时钟、数据和低有效片选信号：

### DMA压力测试

最终测试结果：

|监视变量|结果|含义|
|---|---|---|
|g_test_round|1000|完成1000轮|
|g_pass_count|1000|1000轮全部通过|
|g_fail_count|0|没有失败|
|g_total_bytes|1024000|累计校验1,024,000字节|
|g_first_fail_round|4294967295|没有失败轮次|
|g_mismatch_index|4294967295|没有数据不一致|
|g_dma_tx_complete|1|TX DMA完成|
|g_dma_rx_complete|1|RX DMA完成|
|g_dma_transfer_error|0|DMA无错误|
|g_loopback_result|1|环回测试通过|
|g_csih_error|FSP_SUCCESS|CSIH无错误|

### 验证结论

|测试项目|数据量|结果|
|---|---|---|
|直接访问物理环回|2字节（AA、BB）|通过|
|FIFO物理环回|64字节|通过|
|DMA基础环回|64字节|通过|
|DMA连续压力测试|1024字节 × 1000轮|通过|

在当前1 MHz配置下，单板 CSIH0 的直接访问、FIFO、DMA联动和连续数据一致性验证均已完成。累计验证1,024,000字节，没有检测到传输失败或数据错误。

## 注意事项

1. CSIH0CSS0 是硬件片选输出，不需要用跳线帽连接到其他引脚。
2. 逻辑分析仪应配置为CPOL=0、CPHA=0、MSB first、8 bit和CS active-low。
3. 使用DMA时，CSIH应设置为Direct access mode，并启用Transfer API。
4. 不要直接修改自动生成的 `hal_data.c`。
5. 数据手册建议CSIHnSO和CSIHnSC采用High输出驱动能力。当前1 MHz环回在Medium设置下已经通过；提高频率时应改为High并重新检查信号完整性。
