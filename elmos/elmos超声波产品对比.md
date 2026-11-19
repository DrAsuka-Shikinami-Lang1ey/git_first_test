## Sync 同步信号：是 **Master（E524.20A 主收发器）与 ECU/MCU 控制器之间的一条专用同步线**，与 SPI 总线并行存在。它的核心作用是：

**1. 触发 DSI3 通信周期的起始**
从底部时序图可以看到，DSI3 总线上的通信分为几个阶段：

- **CRM**（Communication Resource Management）— 总线资源管理阶段
- **PDCM**（Partial Discharge / Data Communication Mode）— 数据通信阶段
- **BRC**（Bus Resource Control）— 总线资源控制

MCU 通过 **Sync 信号** 告知 Master "现在开始一轮 DSI3 总线扫描"，Master 收到 Sync 后启动 CRM → PDCM → BRC 的完整通信周期。

**2. 确保 MCU 与 DSI3 总线时序对齐**
- MCU 不知道 DSI3 总线上各 Slave 何时响应
- Sync 让 MCU 精确知道"这一轮通信从何时开始"，从而在正确的时间窗口通过 SPI 读取结果数据
- 避免 MCU 在总线还在通信时就去读数据（读到脏数据）
**可以直接拿一个普通的GPIO口去做**

## DSI3 的 CRM 数据结构
- 寄存器读取；
- 向传感器发送命令；
- 主机命令帧与从机响应帧的位字段结构。

CRM 是“一问一答”式通信：主控发送命令，指定传感器返回寄存器值或执行结果。

## 为什么新一代 ADAS ECU 需要 E521.44：
- 传统方案在 DSI3 主控与高性能 ADAS MCU 之间增加一颗专用 MCU。
- E521.44 支持直接连接 NVIDIA、Qualcomm 等高性能控制器，减少中间 MCU。
- SPI I/O 电压扩展到 1.8–5.0 V，可以省去电平转换器。
- 芯片内置数据缓存，降低 ADAS MCU 的实时读取负担和周期中断次数。
- 内部产生模拟、数字电压，减少外部电源器件。
- DSI3 码片时间缩短到 2 μs，速率提升到 666 kbit/s，比上一代快约 50%。

## 三种拓扑方式
菊花链、并联总线和点对点连接。
