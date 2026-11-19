# RLIN3 LIN 主从通信示例说明

> 本文介绍基于 Renesas R-Car U5L4 MCU 的 RLIN3 LIN 主从通信。项目使用一块霆宝科技 R-Car U5L4 评估板，分别验证 MCU 作为 LIN 主节点和从节点时的通信功能，并完成无条件帧、诊断帧、睡眠/唤醒、从节点自动波特率以及单从节点事件触发帧验证。

## 背景介绍

LIN（Local Interconnect Network）是一种低成本、单线、主从式汽车通信总线，常用于车窗、车灯、座椅、空调执行器和传感器等对带宽要求较低的车身控制节点。

LIN 网络中只能有一个主节点。主节点负责维护报文调度表并发送帧头，从节点根据收到的帧 ID 决定是否发送响应。

一个标准 LIN 帧主要由以下部分组成：

```
Break → Sync(0x55) → PID → Data → Checksum
```

本项目主要解决以下问题：

- MCU作为LIN主节点，按照调度表发送帧头并收发数据。
- MCU作为LIN从节点，根据ID自动接收或响应。
- 支持诊断帧 `0x3C/0x3D`。
- 支持睡眠命令和总线唤醒。
- 从节点根据主节点Sync字段自动适应波特率。
- 使用单次响应模拟事件触发帧。
- 通过回调事件统计通信状态和错误。
![](picture/Pasted%20image%2020260729155656.png)
## 环境说明

- 开发工具：VSCode
- 配置工具：FSP Smart Configurator
- 软件平台：
    - R-Car MCU U5x FSP 1.1.1
    - TCANLINPro v3.1
- 目标芯片：R-Car MCU U5L4（R7F711001EBFC）
- 调试工具：Renesas E2
- LIN分析设备：图莫斯 UTA0503
- 波形验证设备：示波器
- 使用外设：
    - RLIN35：实现LIN主节点和从节点通信
    - OSTM：为LIN调度表及传输超时提供定时
    - IOPORT：控制LIN收发器的睡眠与唤醒信号



![](../L3-07%EF%BC%9AFreeRTOS%20%E9%9B%86%E6%88%90%E4%B8%8E%E5%A4%9A%E4%BB%BB%E5%8A%A1%E8%B0%83%E5%BA%A6%EF%BC%88freertos_integration_ep%EF%BC%89/picture/d6d1fb3a7cf62ae7949e6dfa76ceecbc_compress%281%29.jpg)

本次实验采用了由霆宝科技开发的 R-Car U5L4评估板。
## 功能设计

本项目按照以下阶段完成验证：

|阶段|MCU角色|图莫斯角色|验证内容|
|---|---|---|---|
|1|从机|主机|从节点ID接收与响应|
|2|主机|从机|主节点调度表及双向通信|
|3|主机|从机|诊断帧 `0x3C/0x3D`|
|4|主机|从机|睡眠命令及总线唤醒|
|5|从机|主机|19200/9600自动波特率|
|6|主机|从机|单从节点事件触发帧|


## 硬件连接

### LIN总线连接

|开发板P2接口|图莫斯接口|备注|
|---|---|---|
|12V|VBAT/12V|LIN收发器供电|
|LIN|LIN|LIN单线总线|
|GND|GND|必须共地|

### MCU与板载LIN收发器连接

|MCU引脚|外设功能|收发器信号|作用|
|---|---|---|---|
|P10_1|RLIN35TX|LIN_TXD|MCU发送数据到LIN收发器|
|P10_2|RLIN35RX|LIN_RXD|MCU接收LIN收发器输出|
|P10_0|GPIO输出|SLP_N|收发器睡眠控制，高电平正常工作|
|P11_7|GPIO输入|WAKE_N|可选外部唤醒检测|

P10_1和P10_2属于收发器逻辑侧信号。外部图莫斯只连接P2接口上的LIN单线，不应直接连接MCU的TX/RX引脚。

## 软件设计

项目使用以下FSP模块：

- `rm_lin_stack`：LIN协议栈和调度表管理。
- `r_rlin3_lin`：RLIN3 LIN底层驱动。
- `r_ostm`：调度及传输层超时定时器。
- `r_ioport`：引脚与收发器控制。

### 时钟配置

RLIN3时钟源选择MOSC，外部晶振频率为16 MHz。

该配置必须与开发板实际晶振一致。

### RLIN3配置


```
LIN Stack：g_lin_stack0
RLIN3：g_lin0
Channel：5
Operation Mode：Master
Transfer Mode：Master schedule continuous
Baud Rate：19200 bit/s
Break Low Width：13 Tbits
Break High Width：0 Tbit
```

主机连续调度模式需要：

- Transport Layer Timer
- Schedule Timer

引脚配置为：

```
P10_1 → RLIN35TX
P10_2 → RLIN35RX
```

### RLIN3从机自动波特率配置

从机自动波特率实验使用：

```
Operation Mode：Slave
Slave Mode：Slave Auto
Transfer Mode：Slave auto mode
Reference Baud Rate：19200 bit/s
```

自动波特率模式下，RLIN3通过检测Break字段并测量Sync字段，自动设置从节点通信波特率。芯片手册第331页说明了该工作方式。

## 无条件帧设计

基本主从通信使用两个无条件帧：

|帧ID|主机操作|从机操作|数据|
|---|---|---|---|
|`0x24`|主机写|从机接收|`01 02 03 04 05 06 07 08`|
|`0x25`|主机读|从机发送|从机返回8字节数据|

普通数据帧采用增强校验。增强校验同时覆盖PID和Data，错误检测能力高于标准校验。

主节点调度表示意：

```c
static lin_stack_frame_info_t frames[] =
{
    {
        LIN_STACK_FRAME_TYPE_UNCD,
        LIN_STACK_FRAME_DIR_PUB,
        50U,
        {0x24U, {tx_data}, 8U, LIN_CHECKSUM_TYPE_ENHANCED}
    },
    {
        LIN_STACK_FRAME_TYPE_UNCD,
        LIN_STACK_FRAME_DIR_SUB,
        50U,
        {0x25U, {rx_data}, 8U, LIN_CHECKSUM_TYPE_ENHANCED}
    }
};
```

## 诊断帧设计

LIN诊断使用固定帧ID：

|帧ID|名称|方向|
|---|---|---|
|`0x3C`|Master Request Frame|主机发送诊断请求|
|`0x3D`|Slave Response Frame|从机发送诊断响应|

本实验使用的诊断请求为：

```
0F 06 B2 00 FF 7F FF FF
```

图莫斯从机返回：

```
0F 06 F2 34 12 21 43 FF
```

字段含义：

- `0F`：NAD，从节点地址。
- `06`：PCI，单帧中包含6字节诊断数据。
- `B2`：请求SID。
- `F2`：肯定响应SID。
- 其余字节：供应商ID、功能ID、变体等测试数据。

诊断帧 `0x3C/0x3D` 必须使用标准校验，这是LIN协议规定，不使用增强校验。

## 睡眠与唤醒设计

### 睡眠命令

主节点通过 `0x3C` 发送：

```
00 FF FF FF FF FF FF FF
```

其中首字节为 `0x00`，表示让总线节点进入睡眠状态。

发送睡眠命令后，程序通过P10_0将板载LIN收发器的SLP_N拉低，使收发器进入睡眠。

### 唤醒信号

程序重新拉高SLP_N，并通过RLIN3发送7 Tbits的显性低电平唤醒脉冲。

在19200 bit/s下：

```
1 Tbit = 1 / 19200 ≈ 52.08 μs
7 Tbits ≈ 364.6 μs
```

实际测得唤醒低电平约360.6 μs，与理论值接近。

唤醒后等待约100 ms，再发送验证帧，确认从节点恢复通信。

## 自动波特率设计

图莫斯作为主机，MCU作为从机。

验证步骤：

1. 图莫斯以19200 bit/s发送 `0x24` 和 `0x25`。
2. MCU正常接收和响应。
3. 保持MCU运行，不复位、不重新下载程序。
4. 停止图莫斯LIN通道。
5. 将图莫斯波特率修改为9600 bit/s。
6. 重新启动并发送相同帧。
7. MCU继续正常通信。

该现象证明RLIN3从节点能够根据新的Break和Sync字段重新测量并调整波特率。

## 事件触发帧设计

当前FSP LIN Stack只提供：

```
LIN_STACK_FRAME_TYPE_UNCD
LIN_STACK_FRAME_TYPE_DIAG
```

没有原生事件触发帧类型。因此本实验通过应用层方式模拟单从节点事件触发。

定义：

- 关联无条件帧ID：`0x26`
- `0x26`对应PID：`0xA6`
- 事件槽ID：`0x30`
- 事件响应：`A6 11 22 33 44 55 66 77`

工作过程：

```
无新事件：
MCU发送0x30帧头 → 图莫斯不响应

产生事件：
MCU发送0x30帧头 → 图莫斯单次响应
A6 11 22 33 44 55 66 77

事件响应完成：
后续0x30轮询 → 图莫斯重新保持静默
```

图莫斯配置为“单次响应”，因此第一次轮询成功，后续轮询显示 `SLAVE_NO_RESPONSE`。这属于无新事件时的预期现象。

## 功能验证

### 基本主从通信

MCU从机模式验证结果：

```
初始化状态：FSP_SUCCESS
接收计数：3
发送计数：3
错误计数：0
```

MCU主机连续调度验证结果：

```
调度计数：206
发送计数：104
接收计数：103
错误计数：0
接收数据匹配：true
```

说明主机调度表和从机ID响应功能正常。

### Break、Sync和PID波形

在19200 bit/s下：

```
13 Tbits理论Break宽度 ≈ 677.1 μs
实测Break宽度 ≈ 672.72 μs
```

误差处于允许范围内，Break、Sync和PID结构正确。

### 诊断帧

```
g_lin_diag_tx_count = 1
g_lin_diag_rx_count = 1
g_lin_diag_error_count = 0
g_lin_diag_response_matches = true
```

诊断请求和响应均正确。

### 睡眠与唤醒

```
g_lin_sleep_wakeup_phase = LIN_SLEEP_WAKEUP_PHASE_COMPLETE
g_lin_sleep_command_tx_count = 1
g_lin_wakeup_tx_count = 1
g_lin_verify_rx_count = 1
g_lin_error_count = 0
g_lin_verify_response_matches = true
g_lin_sleep_wakeup_complete = true
```

睡眠、唤醒及唤醒后通信验证成功。

### 自动波特率

MCU未复位的情况下，图莫斯从19200 bit/s切换至9600 bit/s后，MCU接收和发送计数继续增加，数据匹配且错误计数保持为0，自动波特率验证成功。

### 事件触发帧

```
g_lin_event_rx_count = 1
g_lin_event_detected = true
g_lin_event_response_matches = true
g_lin_event_response_data =
    A6 11 22 33 44 55 66 77
g_lin_event_bus_error_count = 0
```

单从节点事件触发响应验证成功。



## 注意事项
1. P10_1和P10_2是LIN收发器逻辑侧TX/RX，不应直接连接图莫斯LIN端口。
    
2. SLP_N必须保持高电平，收发器才能正常通信。
    
3. 普通LIN数据帧使用增强校验；诊断帧 `0x3C/0x3D`必须使用标准校验。
    
4. 自动波特率测试切换波特率时不能复位MCU，否则无法证明运行中的自适应能力。

