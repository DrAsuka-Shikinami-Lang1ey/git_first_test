---
title: U2A8 SPI通信项目开发流程
tags:
  - RH850
  - U2A8
  - SPI
  - MCAL
  - DaVinci
  - GHS-MULTI
aliases:
  - U2A8 SPI开发
---

# U2A8 SPI通信项目开发流程

> [!summary]
> U2A8项目中，**DaVinci Configurator**负责配置并生成MCAL底层驱动，作用类似U5L4的FSP Configurator；**GHS MULTI**负责管理工程、编写代码、编译和在线调试；**Renesas E2**是连接电脑与目标板的硬件调试器。

## 1. 软件和硬件的职责

| 工作内容 | U5L4常用工具 | U2A8当前工具 |
|---|---|---|
| 配置引脚、时钟和SPI参数 | FSP Configurator | DaVinci Configurator |
| 生成底层驱动代码 | Generate Project Content | DaVinci Generate生成MCAL代码 |
| 编写C代码 | e² studio或VS Code | GHS MULTI；VS Code可辅助编辑 |
| 编译和链接 | GCC/LLVM工具链 | GHS Compiler 2022.1.4 |
| 工程管理 | e² studio工程 | MULTI Project Manager的`.gpj`工程 |
| 下载和在线调试 | e² studio Debug | MULTI Debugger |
| 电脑与板卡间的调试硬件 | E2/E2 Lite/J-Link | Renesas E2 Emulator |
| 查看SPI真实波形 | 逻辑分析仪或示波器 | 逻辑分析仪或示波器 |
| 查看打印、输入调试命令 | 串口终端 | 可选，不是自动SPI通信必需环节 |

> [!important]
> `e² studio`是IDE软件，`E2 Emulator`是硬件调试器，两者不是同一个东西。对于当前U2A8工程，GHS MULTI承担了e² studio的大部分职责。

```mermaid
flowchart LR
    A[DaVinci配置Port和SPI] --> B[Generate生成MCAL代码]
    B --> C[GHS MULTI编写应用代码]
    C --> D[GHS Compiler编译链接]
    D --> E[生成u2a_mcal.elf]
    E --> F[MULTI Debugger]
    F --> G[E2调试器]
    G --> H[U2A8目标板]
    H --> I[逻辑分析仪观察SPI波形]
```

## 2. 当前工程信息

工程目录：

```text
D:\MCAL\Project\U2A8_Manual_Practice
```

主要文件：

| 文件 | 作用 |
|---|---|
| `default.gpj` | GHS MULTI顶层工程 |
| `tools\Mcal\u2a.dpa` | DaVinci工程 |
| `u2a_mcal\app\App_Task.c` | 系统初始化和任务代码 |
| `u2a_mcal\app\app_tst\App_Spi_Test.c` | 原有SPI测试代码 |
| `u2a_mcal\bsp\mcal\modules\spi\include\Spi_Cfg.h` | DaVinci生成的SPI配置宏 |
| `output\u2a_mcal.elf` | 编译后用于下载的程序文件 |

当前工程已经配置了三组SPI：

| ID | SPI接口 | 原用途 | SCLK | MOSI | MISO | CS |
|---|---|---|---|---|---|---|
| 0 | MSPI0 | 板载PMIC RAA271084 | P2_12 | P2_13 | P2_11 | P2_14 |
| 1 | MSPI1 | AFE1扩展接口 | P6_8 | P5_2 | P6_9 | P6_6 |
| 2 | MSPI2 | AFE2扩展接口 | P10_11 | P10_10 | P10_12 | P10_9 |

> [!warning]
> ID 0连接板载PMIC。不要向ID 0发送没有经过PMIC手册确认的随机数据，否则可能修改电源管理寄存器，引起复位、掉电或电压变化。

## 3. SPI项目的标准开发顺序

### 第1步：确定外部SPI器件和需求

先确定准备连接的器件，例如：

- SPI Flash；
- EEPROM；
- ADC；
- 传感器；
- 另一块MCU；
- 暂时没有外设，先做MOSI与MISO回环测试。

从器件数据手册中确认：

- 供电电压和SPI I/O电压；
- SPI Mode 0、1、2或3；
- CPOL和CPHA；
- 最大允许时钟频率；
- 数据宽度；
- MSB First或LSB First；
- CS有效电平；
- CS建立时间、保持时间和帧间间隔；
- 读写命令格式；
- 寄存器地址长度；
- 读取时是否需要Dummy Byte。

**理由：** SPI没有自动协商机制，主机和从机的时钟模式、位序或数据宽度只要有一项不一致，就可能有波形但无法得到正确数据。

### 第2步：规划使用的SPI接口和引脚

初次练习建议使用MSPI1，即CN21扩展接口。CN21主要引脚如下：

| CN21引脚 | 信号 | 说明 |
|---|---|---|
| 1 | AFE.PWR.5V | 5V电源，使用前必须核对外设电压 |
| 4 | AFE1.CS | SPI片选 |
| 5 | AFE1.SCLK | SPI时钟 |
| 6 | AFE1.SDI | MCU的MOSI |
| 7 | AFE1.SDO | MCU的MISO |
| 10 | GND | 公共地 |

如果连接外部器件：

```text
CN21-5  SCLK  → 外设SCLK
CN21-6  MOSI  → 外设MOSI/SDI/DIN
CN21-7  MISO  ← 外设MISO/SDO/DOUT
CN21-4  CS    → 外设CS/NSS/SS
CN21-10 GND   ↔ 外设GND
```

> [!danger]
> 所有接线必须在板卡断电时完成。不得在没有确认电压的情况下连接CN21第1脚5V，也不得把外部5V逻辑信号直接送入MCU引脚。

### 第3步：用DaVinci配置Port模块

启动DaVinci：

```text
D:\Vector\CBD2100679_D00_U2A8_292_V430300_Practice\
DaVinciConfigurator\Core\DaVinciCFG.exe
```

在DaVinci中打开：

```text
D:\MCAL\Project\U2A8_Manual_Practice\tools\Mcal\u2a.dpa
```

在`Port`模块中设置：

1. SCLK引脚的外设复用功能；
2. MOSI引脚的外设复用功能和输出方向；
3. MISO引脚的外设复用功能和输入方向；
4. CS引脚的外设复用功能或GPIO输出功能；
5. CS的安全初始电平，通常为未选中状态。

如果继续使用工程现有的MSPI1/CN21，当前引脚复用通常已经配置完成，不需要重新选择引脚。

**理由：** SPI模块即使已经启动，如果Port复用仍然是普通GPIO，SPI信号也不会出现在芯片引脚上。

### 第4步：用DaVinci配置Spi模块

在`Spi`模块中配置以下对象：

```text
SpiChannel → SpiJob → SpiSequence
```

- `SpiChannel`：描述数据宽度、位序、默认数据和缓冲区类型；
- `SpiJob`：描述使用哪个MSPI硬件、哪个CS和具体时序；
- `SpiSequence`：描述一次通信需要执行哪些Job。

建议使用可识别的名称：

```text
SpiChannel_MySensor
SpiJob_MySensor
SpiSequence_MySensor
```

需要核对的参数：

- Master模式；
- MSPI硬件单元；
- CS通道；
- SPI时钟频率；
- CPOL和CPHA；
- 8位、16位或其他数据宽度；
- MSB First或LSB First；
- CS有效电平；
- CS建立、保持和帧间延时；
- IB或EB缓冲区；
- 同步或异步传输等级。

**理由：** Port模块只决定信号从哪个引脚输出，Spi模块才决定实际产生什么时序。

### 第5步：Generate生成MCAL代码

在DaVinci中点击`Generate`，确认相关模块右侧显示绿色对勾。

生成结果通常包括：

```text
Spi_Cfg.h
Spi_PBcfg.c
Spi_Lcfg.c
Port_Cfg.h
Port_PBcfg.c
```

> [!warning]
> 不要直接修改DaVinci生成的`Spi_Cfg.h`、`Spi_PBcfg.c`和`Spi_Lcfg.c`，下一次Generate会覆盖手工改动。SPI参数应该回到DaVinci中修改。

### 第6步：用GHS MULTI编写应用代码

双击打开：

```text
D:\MCAL\Project\U2A8_Manual_Practice\default.gpj
```

DaVinci负责生成驱动，但不会替项目编写“读取哪个寄存器、发送什么命令”的业务逻辑。这些代码需要在GHS MULTI中编写。

当前工程已经在`App_Task.c`中执行：

```c
Spi_Init(Spi_Config);
Spi_SetAsyncMode(SPI_POLLING_MODE);
```

因此，应用代码中通常不需要再次调用`Spi_Init()`。

正式项目建议新建：

```text
u2a_mcal\app\app_tst\App_MySpi.c
u2a_mcal\app\app_tst\App_MySpi.h
```

新建文件后，需要把`.c`文件添加到`app_tst.gpj`中，否则GHS不会编译它。

### 第7步：编写自动SPI通信程序

下面是外部缓冲区EB的同步传输模板：

```c
#include "Spi.h"
#include "Spi_Cfg.h"

static uint8 SpiTxData[2] = {0x9FU, 0x00U};
static uint8 SpiRxData[2] = {0U, 0U};

void App_MySpi_Test(void)
{
    Std_ReturnType result;

    result = Spi_SetupEB(
        SpiConf_SpiChannel_SpiChannel_MySensor,
        SpiTxData,
        SpiRxData,
        2U);

    if (result == E_OK)
    {
        result = Spi_SyncTransmit(
            SpiConf_SpiSequence_SpiSequence_MySensor);

        if (result == E_OK)
        {
            /* 根据器件协议处理SpiRxData */
        }
    }
}
```

配置宏必须以生成后的`Spi_Cfg.h`为准，不能自己猜测名称或数值。

异步发送模板：

```c
uint32 timeout = 1000000UL;

if (Spi_SetupEB(channel, tx_data, rx_data, length) == E_OK)
{
    if (Spi_AsyncTransmit(sequence) == E_OK)
    {
        while ((Spi_GetSequenceResult(sequence) == SPI_SEQ_PENDING) &&
               (timeout > 0UL))
        {
            timeout--;
        }
    }
}
```

> [!important]
> 等待异步传输结束时必须加入超时保护，不能无限循环等待。

### 第8步：决定SPI通信的执行时机

常见方式如下。

#### 初始化完成后发送一次

```text
系统初始化 → Spi_Init → App_MySpi_Test
```

适合读取器件ID、初始化寄存器。

#### 周期任务中发送

```text
系统初始化 → 每10ms或100ms执行一次SPI任务
```

适合周期采集传感器或ADC数据。应确保上一次传输已经结束，避免重复启动同一个Sequence。

#### 串口命令触发

```text
串口输入命令 → 执行SPI通信
```

适合开发调试，不是正式业务代码所必需。

### 第9步：用GHS MULTI编译

在MULTI Project Manager中：

1. 选择顶层`default.gpj`；
2. 点击`Build`；
3. 选择`Rebuild`或`Build All`；
4. 等待编译结束；
5. 确认显示`Build successful`和`0 Errors`。

输出文件：

```text
D:\MCAL\Project\U2A8_Manual_Practice\output\u2a_mcal.elf
```

**理由：** MULTI Debugger最终下载的是编译链接后的ELF文件，不是DaVinci工程或ARXML文件。

### 第10步：连接E2并下载程序

硬件操作顺序：

1. 板卡断电；
2. E2目标排线连接板卡CN12，核对1脚方向；
3. E2通过USB连接电脑；
4. 板卡DC1接入规定的12V电源；
5. 给板卡上电；
6. 检查板卡电源指示灯和Windows设备管理器。

MULTI操作顺序：

1. 编译成功后点击`Debug`；
2. 在MULTI Debugger中点击`Connect`；
3. 选择E2连接；
4. 点击`Download/Prepare Target`；
5. 下载`u2a_mcal.elf`；
6. 设置断点；
7. 点击Run运行。

工具关系：

```text
MULTI Debugger：控制下载、运行、断点和变量观察
E2 Emulator：完成电脑与U2A8芯片之间的物理调试连接
```

### 第11步：验证SPI通信

推荐同时使用以下方法。

#### 逻辑分析仪

至少测量：

- CS；
- SCLK；
- MOSI；
- MISO；
- GND。

重点检查：

1. CS是否在传输前进入有效状态；
2. SCLK空闲电平是否符合CPOL；
3. 数据采样边沿是否符合CPHA；
4. 时钟频率是否在器件范围内；
5. MOSI命令和地址是否正确；
6. MISO是否有有效响应；
7. 数据宽度和字节顺序是否正确；
8. CS保持时间是否满足器件要求。

#### MULTI变量窗口

在传输完成后设置断点并观察：

```c
SpiTxData
SpiRxData
result
Spi_GetSequenceResult(sequence)
```

#### 串口打印（可选）

```c
PRINTF("SPI RX: %02X %02X\n", SpiRxData[0], SpiRxData[1]);
```

如果程序自动完成SPI传输，串口不是必需的。串口只用于输出结果或输入人工测试命令。

## 4. 无外设时的MSPI1回环测试

如果暂时没有SPI从设备，可以先验证工具链和板卡信号。

### 硬件连接

板卡断电并确认CN21没有连接AFE子板，然后短接：

```text
CN21第6脚MOSI/SDI ↔ CN21第7脚MISO/SDO
```

逻辑分析仪连接：

| 逻辑分析仪 | CN21 |
|---|---|
| GND | 第10脚GND |
| CH1 | 第4脚CS |
| CH2 | 第5脚SCLK |
| CH3 | 第6脚MOSI |
| CH4 | 第7脚MISO |

### 软件验证

可以保留现有测试程序，并通过串口输入：

```text
spi list
spi get -d 1
spi transmit 1 0xA55A
spi get -s 1
```

成功标准：

- SPI状态为空闲；
- 传输结果为`SPI_SEQ_OK`；
- CS产生有效电平；
- SCLK产生时钟；
- MOSI与MISO波形一致；
- 接收数据与发送数据对应。

该串口方式只是快速验证现有驱动。如果已经编写自动调用`Spi_SetupEB()`和`Spi_SyncTransmit()`的程序，就不必使用串口命令。

## 5. 常见问题

| 现象 | 优先检查 |
|---|---|
| CS、SCLK都没有波形 | 程序是否运行、SPI是否初始化、Sequence是否正确、Port复用是否正确 |
| CS变化但没有时钟 | Job与Channel关联、MSPI模块时钟、发送长度 |
| MOSI正常，MISO一直为1 | 从设备未供电、CS错误、MISO断路、命令格式错误 |
| MISO一直为0 | 接线短路、器件处于复位、接口电平错误 |
| 数据整体错一位 | CPHA设置错误 |
| SCLK空闲电平错误 | CPOL设置错误 |
| 字节顺序相反 | CPU小端内存顺序或器件协议字节序 |
| 编译找不到SPI配置宏 | DaVinci未重新Generate，或者应用代码中的宏名称错误 |
| 新建`.c`文件没有参与编译 | 文件未添加到对应`.gpj`工程 |
| E2无法连接 | CN12排线方向、板卡供电、E2驱动、Device File和LPD配置 |
| MULTI提示许可证错误 | 公司网络/VPN、`GHS_LMHOST`、许可证占用情况 |

## 6. 每次修改后的固定流程

```text
确定器件通信参数
        ↓
DaVinci配置Port和Spi
        ↓
Generate生成MCAL代码
        ↓
GHS MULTI编写应用程序
        ↓
Rebuild并确认0 Errors
        ↓
E2连接目标板
        ↓
下载u2a_mcal.elf
        ↓
运行和断点调试
        ↓
逻辑分析仪检查波形
        ↓
核对接收数据
```

## 7. 上板前检查清单

- [ ] 已确认芯片为U2A8 292-pin/FABG
- [ ] 已确认外设供电电压和IO电压
- [ ] 已确认SPI Mode、频率、位宽和位序
- [ ] 板卡断电状态下完成接线
- [ ] MCU与外设、逻辑分析仪已经共地
- [ ] DaVinci Port引脚复用正确
- [ ] DaVinci Channel、Job、Sequence关联正确
- [ ] DaVinci Generate成功
- [ ] 应用代码使用的是生成后的配置宏
- [ ] 新建源文件已经加入`.gpj`
- [ ] GHS编译结果为`0 Errors`
- [ ] E2排线方向和目标供电正确
- [ ] 已准备逻辑分析仪观察CS、SCLK、MOSI和MISO
- [ ] 没有向板载PMIC发送未经确认的随机命令

## 8. 相关资料

- 板卡原理图：`D:\MCAL\RH850-U2A Demo板资料包\1.U2A文档\0_Hardware\StarterKit U2A G2 Rev2.4.pdf`
- GHS在线调试说明：`C:\Users\TPSZ-1300194\Desktop\RH850_U2x 使用GHS MULTI在线调试.pdf`
- 当前DaVinci工程：`D:\MCAL\Project\U2A8_Manual_Practice\tools\Mcal\u2a.dpa`
- 当前MULTI工程：`D:\MCAL\Project\U2A8_Manual_Practice\default.gpj`
- SPI测试代码：`D:\MCAL\Project\U2A8_Manual_Practice\u2a_mcal\app\app_tst\App_Spi_Test.c`
- SPI生成配置：`D:\MCAL\Project\U2A8_Manual_Practice\u2a_mcal\bsp\mcal\modules\spi\include\Spi_Cfg.h`

## 总结

U2A8的开发方式可以简单理解为：**DaVinci负责底层配置和代码生成，GHS MULTI负责编写、编译和调试应用程序，E2负责连接和下载，逻辑分析仪负责确认SPI真实波形。**
