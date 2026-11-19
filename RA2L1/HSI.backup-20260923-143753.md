# RA2L1 HSI 引脚配置与自检记录



- UART 收发测试已经通过，SCI0 使用 `P205/TXD0` 和 `P206/RXD0`，流控应设置为 `None`。
- GPIO、ADC、外部 IRQ 的引脚功能已经按引脚表调整；输入测试中的 `PASS` 仅代表读取成功，不等于外部信号状态一定正常。
- P100 比较器输入已经切换为 `CMPIN0`，但比较器参考源和比较阈值尚未明确，因此比较器功能仍未完成。
- 当前 SPI 测试是物理回环测试，不是自适应回环。没有短接 MOSI 与 MISO 时，测试失败属于预期现象。
- IIC0 当前只验证了驱动初始化，尚未与实际 AFE 从设备通信。
- ADC 当前使用 `AVCC0/AVSS0` 作为参考；模拟量换算是否准确，仍需结合原理图和实测电压确认。

---

## 2. 重点引脚配置变更

### 2.1 P004/60：充电器反接检测

| 项目 | 配置 |
| --- | --- |
| 功能 | 充电器反接检测 |
| 信号类型 | GPIO 输入 / IRQ3 |
| 有效电平 | 高有效 |
| 原配置 | IRQ3，下降沿 |
| 修改后 | IRQ3，上升沿 |

P004 可以保留 IRQ3 功能，但由于信号为高有效，触发沿应设置为上升沿。

![](picture/Pasted%20image%2020260921172755.png)

> [!note]
> `CHARGER REVERSE INPUT TEST PASS` 只表示引脚读取成功。正常情况下若没有发生反接，P004 应为低电平；如果显示 `P004=HIGH`，表示反接检测信号当前处于有效状态。

### 2.2 P003/61：过流状态检测

| 项目 | 配置 |
| --- | --- |
| 功能 | 过流状态检测 `OC_DET` |
| 信号类型 | GPIO 输入 |
| 有效电平 | 高有效 |
| 原配置 | ADC0 AN03 |
| 修改后 | 删除 AN03，改为 GPIO 输入 |
| 外部阈值 | 400 A（160 mV，由外部硬件产生） |

软件检测到 P003 为高后，将 P113 拉高并锁存，用于切断放电 MOS。当前代码采用故障锁存策略：即使 P003 随后恢复为低，P113 仍保持使能，直到复位或执行明确的故障清除流程。

### 2.3 P014/53：放电 MOS 温感

| 项目 | 配置 |
| --- | --- |
| 功能 | 放电 MOS 温感 `D_NTC` |
| 原配置 | DAC0 输出 |
| 修改后 | ADC0 AN009 |

![](picture/Pasted%20image%2020260921174350.png)

当前代码可以采集 AN009 并计算 NTC 电阻：

```text
R_D_NTC = V_D_NTC × 10 kΩ / (VCC_3V3 - V_D_NTC)
```

如需进一步换算为温度（℃），还需要客户提供 NTC 型号、`R25/B 值`或温阻表。

### 2.4 P100/48：比较器输入

| 项目 | 配置 |
| --- | --- |
| 功能 | 比较器输入 |
| 原配置 | IIC1 SCL |
| 修改后 | ACMPLP0 `CMPIN0` |

![](picture/Pasted%20image%2020260921180159.png)

已添加比较器模块，但仅根据引脚表无法确定参考源、参考电压和比较阈值。当前 `Input Voltage` 和 `Reference Voltage` 未完成有效配置，因此测试会显示 `ACMPLP0 is not ready`。

### 2.5 P101/47：充放电 MOS 驱动电源开关

| 项目 | 配置 |
| --- | --- |
| 功能 | MOS 驱动电源使能 |
| 信号类型 | GPIO 输出，高有效 |
| 原配置 | IIC1 SDA |
| 修改后 | GPIO 输出，初始低 |

![](picture/Pasted%20image%2020260921182531.png)

系统启动时 P101 保持低电平。通过串口输入以下命令后，P101 才会拉高：

```text
MOS PWR ON
```

关闭命令：

```text
MOS PWR OFF
```

### 2.6 P502/51：外部温感

| 项目 | 配置 |
| --- | --- |
| 功能 | 外部温感 `TD_A` |
| 原配置 | GPIO 输出，初始低 |
| 修改后 | ADC0 AN018 |

![](picture/Pasted%20image%2020260921182949.png)

---

## 3. 中断与数字输入配置

| 引脚 | 用途 | 最终配置 | 有效电平/说明 |
| --- | --- | --- | --- |
| P001/63 | CAN 唤醒 MCU | IRQ7，上升沿 | 高有效 |
| P000/64 | 4G 唤醒 MCU | IRQ6，上升沿 | 高有效 |
| P411/12 | 充电电压唤醒 | IRQ4，上升沿 | 高有效 |
| P208/20 | ACC 状态检测 | GPIO 输入 | 高有效 |
| P207/21 | 充电 12 V 状态检测 | GPIO 输入 | 高有效 |
| P303/29 | 电池包选中开关检测 | GPIO 输入 | 高有效 |
| P302/30 | 电池包地址线检测 | GPIO 输入 / IRQ5 | 高有效；需要唤醒时使用 IRQ5 |
| P215/6 | 负载状态检测 | GPIO 输入 | 低有效 |
| P204/24 | AFE 报警输入 | GPIO 输入 | 低有效；该脚不能直接配置外部 IRQ |
| P214/7 | 通用输入 | GPIO 输入 | 输入专用端口；有效极性未定义 |

> [!note] GPIO 与 IRQ 的关系
> IRQ 引脚本质上仍然可以读取高低电平，但还具备边沿触发中断能力。需要 MCU 在休眠状态下被外部信号唤醒时，应使用 IRQ；仅需要周期读取状态时，配置为普通 GPIO 输入即可。

由于引脚表没有给出内部上拉要求，当前没有启用内部上拉。是否需要上拉或下拉应以外部电路为准。

---

## 4. GPIO 输出及安全初始状态

未明确给出初始状态的普通输出，暂按“初始低”处理；安全相关信号按照有效电平设置为非使能状态。

| 引脚 | 用途 | 有效电平 | 初始状态 |
| --- | --- | --- | --- |
| P103/45 | 4G DTR / GPIO 输出 | 未明确 | 低 |
| P403/4 | 4G PWRKEY / GPIO 输出 | 未明确 | 低 |
| P112/37 | Flash nWP / GPIO 输出 | 未明确 | 低（建议结合 Flash 手册确认） |
| P400/1 | 4G 模块工作电源使能 | 高有效 | 低 |
| P304/28 | 系统工作电源使能 | 高有效 | 低 |
| P101/47 | MOS 驱动电源使能 | 高有效 | 低 |
| P501/50 | MCU 工作状态指示 | 高有效 | 低 |
| P410/13 | MCU 唤醒 AFE | 高有效 | 低 |
| P104/44 | 保险丝点爆低端 | 低有效 | **高** |
| P500/49 | 保险丝点爆高端 | 高有效 | 低 |
| P105/43 | 过流检测功能开关 | 高有效 | 低 |
| P113/38 | 放电 MOS 切断开关 | 高有效 | 低 |
| P107/41 | 负载检测功能开关 | 高有效 | 低 |
| P106/42 | 充电电压检测开关 | 高有效 | 低 |
| P409/14 | AFE 硬件复位 | 低有效 | **高** |
| P002/62 | 预留 GPIO 输出 | 未明确 | 低；最终用途和安全电平待确认 |

### P409 控制时序

P409 为低有效控制信号，空闲时必须保持高电平：

- 拉低约 12 ms：执行 AFE 复位（满足“超过 8 ms”要求）。
- 拉低约 40 ms：执行 AFE 关断（满足“超过 32 ms”要求）。

串口命令：

```text
AFE RESET
AFE SHUTDOWN
```

---

## 5. 通信与模拟外设配置

### 5.1 SPI1 / Flash 接口

| 引脚 | SPI 功能 | 配置 |
| --- | --- | --- |
| P109/34 | MOSI | SPI1 MOSI |
| P110/35 | MISO | SPI1 MISO |
| P111/36 | CLK | SPI1 RSPCK |
| P301/31 | CS | SPI1 `SSLB2_B` |

当前程序执行的是物理回环测试：发送固定字节后，要求 MISO 返回完全相同的数据。它不是自适应回环，也不会自动识别 Flash。

根据 CPK-RA2L1 评估板原理图，回环测试时可在断电状态下短接：

```text
J2-2（P109/MOSI） ↔ J2-3（P110/MISO）
```

也可以短接：

```text
J14-1（P109/MOSI） ↔ J15-3（P110/MISO）
```

使用真实 SPI Flash 时不得短接 MOSI 与 MISO，应根据 Flash 型号连接四根信号，并将测试改为读取 JEDEC ID（通常使用命令 `0x9F`）。

### 5.2 ADC0

| 引脚 | 信号 | ADC 通道 |
| --- | --- | --- |
| P012/55 | 充电电压采样 | AN007 |
| P013/54 | 充电 MOS 温感 | AN008 |
| P014/53 | 放电 MOS 温感 | AN009 |
| P015/52 | 3.3 V 模拟电压采样 | AN010 |
| P502/51 | 外部温感 | AN018 |

当前 FSP 配置使用 `AVCC0/AVSS0` 作为 ADC 参考。代码按 3300 mV 参考电压换算，硬件上 AVCC0、P015 和模拟分压关系仍需实测确认。

### 5.3 比较器

| 引脚 | 功能 |
| --- | --- |
| P100/48 | ACMPLP0 CMPIN0 |
| P102/46 | ACMPLP1 CMPIN1 |

比较器参考源、参考电压和输出判定阈值未在引脚表中说明，需要客户或原理图补充后才能完成配置和功能测试。

### 5.4 IIC0 / AFE

| 引脚 | 功能 |
| --- | --- |
| P408/15 | IIC0 SCL |
| P407/16 | IIC0 SDA |

当前测试只确认 IIC0 主机驱动能够初始化。若要验证实际通信，还需要 AFE 型号、I²C 地址、寄存器地址及预期返回值。

### 5.5 SCI0 / UART

| 引脚 | 功能 |
| --- | --- |
| P205/23 | TXD0 |
| P206/22 | RXD0 |

FSP 中应配置为 `No Flow Control`，因为引脚表只分配了 TX/RX，没有分配 RTS/CTS。

---

## 6. 串口工具设置与测试命令

### 6.1 串口参数

```text
波特率：115200
数据位：8
停止位：1
校验：无
流控：无
发送模式：文本（不要勾选“16进制”）
发送后缀：CRLF（\r\n，即 0D 0A）
```

![](picture/Pasted%20image%2020260922114705.png)

### 6.2 UART 测试

![](picture/Pasted%20image%2020260922133017.png)
![](picture/Pasted%20image%2020260922133351.png)

输入：

```text
UART TEST
```

出现以下信息表示 MCU 已成功接收命令并返回结果：

```text
[UART TEST]
UART command received and response transmitted.
UART TEST PASS
```

### 6.3 常用命令

| 命令 | 作用 |
| --- | --- |
| `UART TEST` | 测试串口双向通信 |
| `CHG REV TEST` | 读取 P004 充电器反接状态 |
| `INPUT TEST` | 读取全部数字输入状态 |
| `OC TEST` | 读取 P003 过流状态和 P113 锁存状态 |
| `MOS TEMP TEST` | 读取 P014/AN009 并计算 NTC 电阻 |
| `MOS PWR ON` | P101 拉高，开启 MOS 驱动电源 |
| `MOS PWR OFF` | P101 拉低，关闭 MOS 驱动电源 |
| `AFE RESET` | P409 拉低约 12 ms |
| `AFE SHUTDOWN` | P409 拉低约 40 ms |
| `CMP TEST` | 读取比较器状态 |
| `SPI TEST` | 执行 SPI 物理回环测试 |
| `I2C TEST` | 检查 IIC0 初始化状态 |
| `ALL TEST` | 顺序执行当前菜单内的全部测试 |
| `BACK` | 返回主菜单 |

![](picture/Pasted%20image%2020260922133720.png)

---

## 7. ALL TEST 结果分析

| 项目 | 当前结果 | 结论 |
| --- | --- | --- |
| UART | PASS | TX/RX 双向通信正常 |
| CHG REV | PASS（读取成功） | P004 当前为高，表示反接检测有效；若未模拟反接，需要检查外部电路 |
| INPUTS | PASS（读取成功） | 需根据当前台架状态判断各输入电平是否合理 |
| OC INPUT | PASS | P003 低、P113 低，当前无过流；尚未验证实际高电平触发和锁存 |
| MOS TEMP | PASS（采样成功） | VCC_3V3 计算值仅 1592 mV，明显偏低，需要实测 P015 和 AVCC0 |
| CMP | FAIL | 比较器输入/参考源配置未完成 |
| SPI | FAIL | 未连接物理回环线；接真实 Flash 时应改为 JEDEC ID 测试 |
| I2C | PASS（初始化） | 未执行实际 AFE 总线事务 |

### 原始测试输出

<details>
<summary>点击展开</summary>

```text
========== ALL TEST START ==========

[UART TEST]
UART command received and response transmitted.
UART TEST PASS

[CHARGER REVERSE DETECTION]
P004=HIGH: charger reverse detection ACTIVE.
IRQ3 rising edge has not been detected since startup.
CHARGER REVERSE INPUT TEST PASS

[DIGITAL INPUT STATUS]
P001 MCU wake IRQ7: LOW, INACTIVE (high active).
P000 MCU wake IRQ6: LOW, INACTIVE (high active).
P411 charge voltage wake IRQ4: LOW, INACTIVE (high active).
P302 battery address IRQ5: HIGH, ACTIVE (high active).
P208 ACC status: HIGH, ACTIVE (high active).
P207 charge 12V status: HIGH, ACTIVE (high active).
P303 battery selected: LOW, INACTIVE (high active).
P215 load status: LOW, ACTIVE (low active).
P204 AFE alarm: HIGH, INACTIVE (low active).
P214 general input: LOW (active polarity not specified).
DIGITAL INPUT READ PASS

[OVERCURRENT STATUS]
P003=LOW, overcurrent=NORMAL; trigger=400 A (160 mV); P113 cutoff=LOW.

[DISCHARGE MOS TEMPERATURE INPUT]
P014/AN009 raw=1065, V_D_NTC=858 mV, VCC_3V3=1592 mV, R_D_NTC=11689 ohm.

[COMPARATOR INPUT TEST]
ACMPLP0 is not ready. Check Input Voltage and Reference Voltage in FSP.

[SPI TEST]
TX: 00 55 AA 12 34 A5 5A FF
RX: 00 00 00 00 00 00 00 00
SPI LOOPBACK FAIL

[I2C TEST]
IIC1 loopback was removed because P100 is now CMPIN0.
IIC0 master initialization: PASS
Bus transaction skipped: external slave address/protocol is not defined.

========== TEST SUMMARY ============
UART    : TESTED - PASS
CHG REV : TESTED - PASS
INPUTS  : TESTED - PASS
OC INPUT: TESTED - PASS
MOS TEMP: TESTED - PASS
CMP     : TESTED - FAIL
SPI     : TESTED - FAIL
I2C     : TESTED - PASS
====================================
```

</details>

---

## 8. 待确认及后续工作

1. **比较器参数**：确认 P100/P102 的用途、参考源、参考电压和比较阈值。
2. **SPI Flash**：确认 Flash 型号、供电、SPI 模式和 JEDEC ID；当前评估板原理图没有板载 Flash。
3. **ADC 电压**：万用表测量 P015 和 AVCC0。P015 正常情况下应接近 3.3 V 的一半。
4. **P004 状态**：未发生反接时应为低；若持续为高，检查反接检测外围电路和默认上下拉。
5. **AFE 通信**：补充 AFE 型号、I²C 地址、寄存器和期望值，完成真实读写测试。
6. **输入默认状态**：结合实际台架确认 P302、P208、P207、P215 当前有效是否符合预期。
7. **CAN 测试**：当前 `ALL TEST` 汇总未展示 CAN 测试结果，应单独确认 CAN 收发和终端连接。

