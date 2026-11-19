---
title: RH850/U2A-EVA 在 MULTI 环境下的启动流程
aliases:
  - U2A MULTI 启动流程
  - RH850 U2A 第28页后说明
tags:
  - RH850
  - U2A
  - Renesas
  - MULTI
  - 启动流程
  - 多核
  - 嵌入式
source: RH850/U2A-EVA Group Startup Application Note
source_pages: 28-50
---

# RH850/U2A-EVA 在 MULTI 环境下的启动流程

> [!info] 文档范围
> 本笔记整理自《RH850/U2A-EVA Group Startup Application Note》第 28 页之后的内容，重点说明在 Green Hills **MULTI** 开发环境下，RH850/U2A 从上电复位到进入用户 `main()` 函数的完整启动过程。

## 1. 总体概述

从第 28 页开始，文档进入第 3 章，主要介绍：

- MULTI 工程中的启动相关文件；
- 各种程序段的作用和配置方式；
- 芯片上电后的整体启动流程；
- PE0～PE3 的多核启动和同步；
- RAM、寄存器、中断、指针和 FPU 的初始化；
- 最后如何跳转到用户应用程序。

整个过程可以概括为：

```text
芯片上电
  ↓
进入复位向量
  ↓
初始化 CPU 寄存器
  ↓
切换系统时钟到 PLL
  ↓
解除外设模块停止状态
  ↓
由 PE0 启动 PE1～PE3
  ↓
初始化 Local RAM 和 Cluster RAM
  ↓
各个 PE 进行同步
  ↓
设置中断向量表地址
  ↓
设置 gp、tp、sp 等指针
  ↓
配置 FPU
  ↓
进入用户 main() 函数
```

---

# 2. 第 28 页：MULTI 启动相关文件

MULTI 工程中的主要启动文件如下。

| 文件 | 作用 |
|---|---|
| `startup.850` | PE0、PE1 的启动调用、复位入口和向量表 |
| `startup_PE0.850` | PE0 自己的启动程序和向量表 |
| `startup_PE1.850` | PE1 自己的启动程序和向量表 |
| `startup2.850` | PE2、PE3 的启动调用、复位入口和向量表 |
| `startup_PE2.850` | PE2 自己的启动程序和向量表 |
| `startup_PE3.850` | PE3 自己的启动程序和向量表 |
| `main.c` | 公共主程序 |
| `main_pe0.c` | PE0 的主程序 |
| `main_pe1.c` | PE1 的主程序 |
| `main_pe2.c` | PE2 的主程序 |
| `main_pe3.c` | PE3 的主程序 |
| `section.ld` | 链接段和内存地址配置文件 |

## 2.1 PE 的含义

这里的 **PE** 是 Processing Element，可以理解为 CPU 核心。

```text
PE0 = CPU0
PE1 = CPU1
PE2 = CPU2
PE3 = CPU3
```

不同芯片实现的 PE 数量不同：

| 芯片 | PE 数量 |
|---|---:|
| RH850/U2A16 | 4 个：PE0～PE3 |
| RH850/U2A8 | 2 个：PE0、PE1 |
| RH850/U2A6 | 2 个：PE0、PE1 |

因此，U2A8 和 U2A6 不需要执行 PE2、PE3 的启动处理。

---

# 3. 第 29～31 页：程序段配置

程序编译和链接后，不同类型的代码与数据会被放入不同的段中。

## 3.1 主要段及作用

| 段名称 | 作用 |
|---|---|
| `.cintvect` | PE0、PE1 的复位或中断向量入口 |
| `.coldboot` | PE0、PE1 的启动控制代码 |
| `.intvect_PE0` | PE0 的中断向量表 |
| `.intvect_PE1` | PE1 的中断向量表 |
| `.cintvect2` | PE2、PE3 的复位或中断向量入口 |
| `.coldboot2` | PE2、PE3 的启动控制代码 |
| `.intvect_PE2` | PE2 的中断向量表 |
| `.intvect_PE3` | PE3 的中断向量表 |
| `.text` | 可执行程序代码 |
| `.rodata` | 只读常量 |
| `.romdata` | 存放在 ROM 中的带初值数据 |
| `.data` | 运行时位于 RAM 中的带初值变量 |
| `.bss` | 未显式初始化变量，启动时清零 |
| `.sdabase` | Small Data Area 的基地址 |
| `.stack` | 栈空间 |
| `.heapbase` | 堆区基地址 |
| `.heap` | 堆空间 |

通常可以简单理解为：

```text
.text、.rodata、.romdata → 通常放在 Flash/ROM
.data、.bss、.stack、.heap → 通常放在 RAM
```

## 3.2 MULTI 汇编中的段定义

在汇编语言中，可以使用：

```asm
.section 段名称, "属性"
```

常用属性如下：

| 属性 | 含义 |
|---|---|
| `a` | 为该段分配存储空间 |
| `b` | 具有 BSS 语义，只记录大小，启动时清零 |
| `w` | 该段可写 |
| `z` | 该段包含可执行代码 |

例如，一个既可分配、可写，又具有 BSS 语义的段，可以组合使用多个属性。

## 3.3 C 语言中的段定义

MULTI 的 C 语言代码中，可使用：

```c
#pragma ghs section
```

将变量、常量或函数放入指定段。

> [!warning] 注意
> 如果新增了自定义段，除了修改源代码，还需要同步修改 `section.ld`，否则链接器可能不知道该段应该放到哪个地址。

---

# 4. 第 32～36 页：MULTI 启动总流程

这一部分通过流程图和处理表，说明 MULTI 下 RH850/U2A 的启动顺序。

## 4.1 只由 PE0 执行的处理

以下公共硬件初始化通常只允许 PE0 执行：

- 时钟 Gearup；
- 模块待机设置；
- 启动 PE1～PE3。

原因是这些操作影响整个芯片，而不是只影响某个单独 CPU 核心。

## 4.2 每个 PE 分别执行的处理

以下操作会根据当前 PE 编号分别执行：

- 各自负责的 RAM 初始化；
- 各自的指针初始化；
- 各自的中断向量设置；
- 各自的 FPU 设置；
- 进入各自的用户主程序。

---

# 5. 第 37 页：上电和 RESET 复位

芯片上电后，程序计数器 PC 首先指向复位向量地址。

## 5.1 PE0、PE1 的复位入口

PE0、PE1 的复位入口位于：

```text
startup.850
```

复位后会跳转到：

```asm
_RESET
```

示例逻辑：

```asm
_RESETVECT:
    jr _RESET
```

## 5.2 PE2、PE3 的复位入口

PE2、PE3 的复位入口位于：

```text
startup2.850
```

随后跳转到：

```asm
_RESET2
```

## 5.3 上电时哪个 PE 先运行

芯片正常上电时，首先启动的是：

```text
PE0
```

之后由 PE0 根据条件宏决定是否启动其他 PE：

```text
ENABLE_PE1_BY_PE0
ENABLE_PE2_BY_PE0
ENABLE_PE3_BY_PE0
```

含义如下：

| 设置值 | 含义 |
|---:|---|
| `0` | 不启动对应 PE |
| `1` | 允许 PE0 启动对应 PE |

---

# 6. 第 38～39 页：初始化 CPU 寄存器

启动程序需要把 CPU 寄存器设置到一个明确、可预测的状态。

## 6.1 初始化的寄存器类型

主要包括：

- 通用寄存器 `r1～r31`；
- 异常和中断相关寄存器；
- `EIPC`；
- `FEPC`；
- `CTPC`；
- `EBASE`；
- `INTBP`；
- FPU 相关寄存器；
- MPU 相关寄存器；
- Cache 相关寄存器；
- 虚拟化相关寄存器；
- Guest Context 相关寄存器。

大部分寄存器会先被清零。

## 6.2 PSW 示例值

文档给出的 PSW 示例值为：

```text
0x00010020
```

其中主要包含：

- `ID = 1`：禁止接收 EI 级异常或中断；
- `CU0 = 1`：允许使用 FPU。

> [!note] 注意
> 文档中的寄存器值只是示例，实际项目应根据系统需求设置。

## 6.3 MPU 的 32 个 Entry

MPU 包含 32 个 Entry。

启动代码需要通过 `MPIDX` 从 0 遍历到 31，并依次初始化：

```text
MPLA
MPUA
MPAT
```

也就是说，不能只初始化一个 MPU 区域，而应该明确处理全部 32 个 Entry。

---

# 7. 第 40～41 页：时钟 Gearup

PE0 启动后，可以将系统时钟切换到 PLL，以提高 CPU 的运行频率。

## 7.1 执行条件

只有以下条件全部满足时才执行：

```text
ENABLE_CLOCK_GEARUP = 1
当前运行的 PE 为 PE0
主振荡器和 PLL 已稳定
PLLS = 0x00000003
```

## 7.2 基本处理步骤

启动代码大致执行以下操作：

1. 等待主振荡器和 PLL 稳定；
2. 解锁时钟保护寄存器；
3. 配置 PLL 分频；
4. 将 CPU 时钟源切换到 PLL；
5. 等待时钟切换完成；
6. 设置最终分频；
7. 再次确认状态；
8. 重新锁定时钟保护寄存器。

## 7.3 Clock Gearup 的含义

这里的 Clock Gearup 可以理解为：

> 芯片先以较安全、较低的启动时钟运行，等 PLL 稳定后，再切换到较高的正常工作频率。

---

# 8. 第 41 页：模块待机设置

U2A 的部分外设上电后可能处于模块停止状态，或者没有获得模块时钟。

要使用某个外设，通常需要解除对应模块的停止状态。

## 8.1 执行条件

只有以下条件满足时才执行：

```text
ENABLE_MODULE_STANDBY_SET = 1
当前运行的 PE 为 PE0
```

## 8.2 基本处理步骤

1. 解锁模块停止保护寄存器 `MSRKCPROT`；
2. 修改目标模块的停止寄存器；
3. 给对应外设提供时钟；
4. 重新锁定 `MSRKCPROT`。

## 8.3 RS-CANFD 示例

文档以 RS-CANFD 为例，修改：

```text
MSR_RSCFD
```

从而解除 RS-CANFD 模块的停止状态。

> [!tip] 容易误解的地方
> “Module Standby Settings” 并不一定表示让模块进入待机。
> 在这个启动流程中，它更多是指配置模块待机寄存器，并解除需要使用的外设的停止状态。

---

# 9. 第 42 页：启动 PE1～PE3

PE0 通过 `BOOTCTRL` 寄存器启动其他 PE。

## 9.1 BOOTCTRL 位定义

| 位 | 名称 | 对应 PE |
|---:|---|---|
| bit1 | `BC1` | PE1 |
| bit2 | `BC2` | PE2 |
| bit3 | `BC3` | PE3 |

概念上相当于：

```c
BOOTCTRL |= 0x02;  // 启动 PE1
BOOTCTRL |= 0x04;  // 启动 PE2
BOOTCTRL |= 0x08;  // 启动 PE3
```

实际是否执行，还取决于：

```text
ENABLE_PE1_BY_PE0
ENABLE_PE2_BY_PE0
ENABLE_PE3_BY_PE0
```

> [!important]
> 启动其他 PE 的操作只由 PE0 执行。

---

# 10. 第 43～44 页：初始化 RAM

启动程序需要初始化：

- 每个 CPU 独立的 Local RAM；
- 多个 PE 可访问的 Cluster RAM；
- Retention RAM。

## 10.1 为什么要分配给多个 PE 初始化

为了缩短启动时间，文档没有让 PE0 单独清零全部 RAM，而是将不同 RAM 地址范围分给不同 PE。

这样多个 PE 可以并行执行 RAM 初始化。

例如：

```text
PE0 → 初始化 CPU0 Local RAM 和部分 Cluster RAM
PE1 → 初始化 CPU1 Local RAM 和另一部分 Cluster RAM
PE2 → 初始化 CPU2 Local RAM 和另一部分 Cluster RAM
PE3 → 初始化 CPU3 Local RAM 和另一部分 Cluster RAM
```

## 10.2 RAM 清零核心逻辑

清零代码的核心类似：

```asm
st.w  r0, [r6]
add   4, r6
cmp   r6, r7
```

含义是：

1. 将 `r0` 中的 0 写入当前 RAM 地址；
2. 地址增加 4 字节；
3. 判断是否到达结束地址；
4. 未到达则继续写 0。

因此每次清零一个 32 位字。

## 10.3 未启动所有 PE 时的注意事项

如果项目只启动部分 PE，就必须重新调整 RAM 初始化范围。

例如，PE2 没有启动时，不能让原本分配给 PE2 的 RAM 一直不初始化，应把这部分 RAM 分给其他正在运行的 PE。

> [!warning]
> 否则某些 RAM 区域可能保留不确定值，导致程序出现随机异常。

---

# 11. 第 45～46 页：多核同步

不同 PE 完成初始化的速度可能不同。

为了避免某个 PE 提前进入用户程序，文档设置了一个多核同步点，也可以称为 Barrier。

## 11.1 同步标志地址

文档使用 Cluster RAM 中的一个共享地址：

```text
0xFE400000
```

并用其中的 bit0～bit3 表示各个 PE 是否到达同步点。

| 位 | 含义 |
|---:|---|
| bit0 | PE0 已到达同步点 |
| bit1 | PE1 已到达同步点 |
| bit2 | PE2 已到达同步点 |
| bit3 | PE3 已到达同步点 |

## 11.2 同步过程

每个 PE 的处理过程如下：

1. 设置代表自己的标志位；
2. 检查其他 PE 的标志位；
3. 如果还有 PE 没到达，则进入等待；
4. 等待期间执行 `snooze`；
5. 所有 PE 都到达后，继续执行后续启动流程。

示例逻辑：

```asm
set1  0, [r10]     ; PE0 设置自己的到达标志

.L.wait_PE1:
tst1  1, [r10]     ; 检查 PE1
bnz   .L.next
snooze
br    .L.wait_PE1
```

## 11.3 `snooze` 的作用

`snooze` 用于处理器等待时降低无意义的忙循环开销。

它不会让程序结束，只是让 CPU 以更适合等待的方式停留在当前循环中。

---

# 12. 第 47～48 页：设置中断处理地址

U2A 支持两种主要的中断向量方式：

1. 表引用方式；
2. 直接向量方式。

## 12.1 表引用方式

表引用方式使用：

```text
INTBP
```

保存中断向量表的基地址。

只有满足以下条件时才配置：

```text
USE_TABLE_REFERENCE_METHOD = 1
```

不同 PE 使用各自对应的中断向量表。

## 12.2 直接向量方式

直接向量方式会根据 `PSW.EBV` 决定使用：

- `RBASE`；
- 或 `EBASE`。

判断关系：

| `PSW.EBV` | 使用的基址寄存器 |
|---:|---|
| `0` | `RBASE` |
| `1` | `EBASE` |

文档示例中：

```text
PE0、PE1 的 RBASE 初始区域：0x00000000
PE2、PE3 的 RBASE 初始区域：0x00800000
```

> [!note]
> 表引用方式主要通过 `INTBP` 找到向量表；
> 直接向量方式则根据 `RBASE` 或 `EBASE` 计算异常入口地址。

---

# 13. 第 49 页：初始化重要指针

MULTI 启动代码会设置以下重要指针：

- `gp`；
- `tp`；
- `sp`。

## 13.1 全局指针 gp

`gp` 指向：

```text
.sdabase 段起始地址
```

主要用于访问 Small Data Area 中的数据。

对应的链接器符号类似：

```text
___ghsbegin_sdabase
```

## 13.2 文本指针 tp

`tp` 指向：

```text
.robase 段起始地址
```

用于访问相关只读数据区域。

对应的链接器符号类似：

```text
___ghsbegin_robase
```

## 13.3 栈指针 sp

`sp` 指向：

```text
.stack 段末尾
```

并进行必要的地址对齐。

对应的链接器符号类似：

```text
___ghsend_stack
```

## 13.4 为什么必须先设置这些指针

在进入 C 语言程序以前，编译器生成的代码可能已经依赖：

- 栈；
- 小数据区；
- 只读数据区。

因此，`gp`、`tp`、`sp` 必须在进入用户代码之前正确初始化。

---

# 14. 第 50 页：启用 FPU 并进入用户程序

## 14.1 启用 FPU

通过设置：

```text
PSW/FEPSW.CU0 = 1
```

允许 CPU 执行浮点运算指令。

如果项目不使用 FPU，可以将 `CU0` 设置为 0。

> [!warning]
> 如果程序中使用了浮点指令，但 `CU0` 没有开启，CPU 可能产生异常。

## 14.2 进入用户程序

启动初始化完成后，代码会进入 Green Hills 运行库启动流程，例如：

```asm
jr __start
```

运行库随后完成必要的 C 语言运行环境初始化，并最终调用用户的：

```c
main()
```

因此，启动代码的最终目的就是：

> 在用户 C 程序执行前，把寄存器、时钟、RAM、中断、多核、栈、指针和 FPU 等运行环境全部准备好。

---

# 15. 第 51 页以后：修订历史

后面的内容主要是文档修订记录，不再介绍新的启动功能。

修订内容可能包括：

- 增加 U2A8 和 U2A6 支持；
- 修改寄存器初始化表；
- 修改 PLL 启动条件；
- 修改多核启动代码；
- 修改 RAM 初始化范围；
- 修正 FPU 设置；
- 修正用户程序跳转方式；
- 更新工具版本和文档日期。

---

# 16. 一句话总结

> [!summary]
> 第 28 页之后主要说明：在 MULTI 环境中，RH850/U2A 上电后如何从复位向量开始，由 PE0 完成公共硬件初始化和其他 PE 的启动，各个 PE 再分别完成 RAM 初始化、同步、中断和指针配置，最后进入用户的 `main()` 函数。

---

# 17. 关键术语速查

| 术语 | 含义 |
|---|---|
| MULTI | Green Hills Software 的集成开发环境 |
| PE | Processing Element，可理解为一个 CPU 核心 |
| RESET Vector | 芯片复位后 PC 首先取得指令的地址 |
| Clock Gearup | 将启动时钟切换到 PLL 高速时钟 |
| Module Standby | 外设模块停止或时钟控制机制 |
| Local RAM | 某个 CPU 核心主要使用的本地 RAM |
| Cluster RAM | 多个 PE 可访问的共享或集群 RAM |
| Retention RAM | 特定状态下可保持数据的 RAM |
| Barrier | 多核同步点 |
| INTBP | 表引用中断方式的向量表基址寄存器 |
| RBASE | 直接向量方式使用的复位基址 |
| EBASE | 扩展异常基址寄存器 |
| gp | Global Pointer，全局小数据区指针 |
| tp | Text Pointer，文本或只读区相关指针 |
| sp | Stack Pointer，栈指针 |
| FPU | Floating Point Unit，浮点运算单元 |
| PSW | Program Status Word，程序状态字 |
| CU0 | PSW/FEPSW 中控制 FPU 使用的位 |
