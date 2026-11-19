# Flash LP 驱动示例：导入、配置与 Data Flash 实板测试

## 1. 测试目的与验证范围

使用瑞萨官方 Flash LP 示例，由 MCU 程序调用 FSP 的 `r_flash_lp` 驱动，对片内 Data Flash 执行空白检查、按需擦除、写入和读回比较。

本页截图记录的是 **RA2L1 实板测试**，不是客户 RA2E1 板的测试结果。

| 项目 | 本次验证配置 |
| --- | --- |
| MCU | R7FA2L1AB2DFM（64 引脚） |
| 开发环境 | e² studio |
| FSP | 6.5.0 |
| 示例工程 | flash_lp_ek_ra2l1_ep |
| 工作空间 | D:\RenesasWorkspaces\RA2E1（文件夹名不代表当前芯片型号） |
| 调试与交互 | J-Link 调试连接、J-Link RTT Viewer |
| 本次测试项 | 菜单 1：Data Flash Operations |

测试会改写所选 Data Flash 块中的数据，应选择可用于测试的块。本文使用块号 `1`，即从 0 开始编号的 Block 1。

`Load Ancillary File` 是调试器下载文件的功能；本示例运行后，由 MCU 执行 `R_FLASH_LP_*` API 完成 Flash 操作。RTT 负责显示日志、接收菜单输入。

## 2. 准备官方工程与软件

### 2.1 本次 RA2L1 测试资料

官方示例压缩包：

```text
D:\R2AL1\r20an0598eu-ek-ra2l1-exampleprojects.zip
```

解压后，找到以下目录中的 e² studio 工程：

```text
ek_ra2l1/flash_lp/flash_lp_ek_ra2l1_ep/e2studio
```

### 2.2 客户 RA2E1 对应资料

已下载到本机的官方原始压缩包：

```text
D:\R2AL1\r20an0607eu-ek-ra2e1-exampleprojects.zip
```

其中的 Flash LP 工程位于：

```text
ek_ra2e1/flash_lp/flash_lp_ek_ra2e1_ep/e2studio
```

[瑞萨官方 RA2E1 示例包下载（FSP 6.5.0）](https://github.com/renesas/ra-fsp-examples/releases/download/v6.5.0.example.1/r20an0607eu-ek-ra2e1-exampleprojects.zip)

客户现有 MCUboot 工程配置为 `R7FA2E1A92DFM`。客户使用 RA2E1 示例时，应按实板核对器件型号，不能照抄下文 RA2L1 的型号设置。该 RA2E1 压缩包仅完成下载与完整性校验，尚未在客户板上验证。

### 2.3 RTT 软件

在 Windows 开始菜单中查找 **J-Link RTT Viewer**。如果已经安装，可直接使用；未安装时，安装 [SEGGER J-Link Software and Documentation Pack](https://www.segger.com/downloads/jlink/)。

本机程序路径为：

```text
C:\Program Files\SEGGER\JLink_V942\JLinkRTTViewer.exe
```

客户电脑的版本及安装路径可能不同。建议发送官方安装包链接，不单独复制一个 EXE。

本文的 RTT 连接步骤基于 J-Link。如果客户使用 E2/E2 Lite，需要另行确认交互方式。

## 3. 导入 Flash LP 工程

1. 解压官方示例包。
2. 在 e² studio 中选择 **文件 → 导入 → 常规 → 现有项目到工作空间**。
3. 选择“选择根目录”，浏览到示例中的 `e2studio` 目录；也可选择能搜索到该工程的上级目录。
4. 在工程列表中勾选 `flash_lp_ek_ra2l1_ep`。
5. 勾选“将项目复制到工作空间中”，点击“完成”。
6. 在项目资源管理器中展开工程，双击 `configuration.xml`。

![导入 Flash LP 示例工程](picture/Pasted%20image%2020261014100517.png)

![打开工程的 FSP 配置页面](picture/Pasted%20image%2020261014101550.png)

## 4. 将器件配置改为实板型号

官方 EK-RA2L1 示例默认器件为 `R7FA2L1AB2DFP`；本次实板是 `R7FA2L1AB2DFM`，需要调整。

1. 在 `configuration.xml` 下方点击 **BSP**。
2. 确认 **FSP version** 为 `6.5.0`。
3. 将 **Board** 改为 **Custom User Board (Any Device)**，或界面中对应的 Custom User Board 选项。
4. 点击 **Device** 右侧的 `…`，选择 `R7FA2L1AB2DFM`。
5. 核对完整型号后保存。

![将器件切换为实板型号](picture/Pasted%20image%2020261014101906.png)

## 5. 检查引脚和时钟

### 5.1 引脚配置

1. 点击下方 **Pins**。
2. 在 **Select Pin Configuration** 下拉框中选择 `R7FA2L1AB2DFM.pincfg`。
3. 保持 **Generate data** 勾选，符号名为 `g_bsp_pin_cfg`。
4. 如果旧的 `RA2L1 EK`、`R7FA2L1AB2DFP.pincfg` 配置仍有红叉，进入 **Manage configurations…**，取消旧配置的生成或移除不再使用的旧配置，保留当前 DFM 配置。
5. 保存后确认“问题”窗口为 **0 个错误**。

仅测试片内 Flash 不需要增加 Flash 外设引脚；仍需保留板上使用的调试接口配置。

![选择与实板一致的引脚配置](picture/Pasted%20image%2020261014103216.png)

### 5.2 时钟配置

点击 **Clocks**，本次使用以下参数：

| 项目 | 设置 |
| --- | --- |
| Clock Src | HOCO |
| HOCO | 48 MHz |
| ICLK | Div /1，48 MHz |
| PCLKB | Div /2，24 MHz |
| PCLKD | Div /1，48 MHz |
| CLKOUT | Disabled |

该系统时钟选择内部 HOCO，无需依靠外部晶振提供系统时钟。Flash LP 编程、擦除所用时钟需满足器件要求；本次配置已完成实板测试。

![HOCO 48 MHz 时钟配置](picture/Pasted%20image%2020261014103221.png)

## 6. 检查 Flash 驱动配置

点击 **Stacks**，选中 `g_flash Flash (r_flash_lp)`，在下方“属性”中确认：

| 属性 | 设置 |
| --- | --- |
| Code Flash Programming | Enabled |
| Data Flash Programming | Enabled |
| Data Flash Background Operation Support | Enabled |
| Name | g_flash |
| Data Flash Background Operation | Enabled |
| Callback | bgo_callback |
| Flash Ready Interrupt Priority | Priority 2 |

BGO 表示后台操作：API 发起操作后，完成状态通过中断和回调通知程序。因此应保留示例的回调名称与中断配置。

示例同时包含 Code Flash 功能，但本流程只选择 Data Flash 测试。

![Flash LP 驱动属性设置](picture/Pasted%20image%2020261014103412.png)

## 7. 生成代码并编译

1. 按 `Ctrl + S` 保存配置。
2. 点击右上角 **Generate Project Content**，等待生成完成。
3. 查看“问题”窗口，先处理红色错误。
4. 右键当前工程，执行 **Clean Project**，然后执行 **Build Project**。
5. 在“控制台”查看构建结果。

本次实际输出：

```text
Building target: flash_lp_ek_ra2l1_ep.elf
Build Finished. 0 errors, 1 warnings.
```

这表示编译成功，生成了 `.elf` 和 `.hex` 文件。

本次唯一警告来自 `SEGGER_RTT_printf.c`：

```text
conversion to 'unsigned int' from 'int' may change the sign of the result
```

该警告未阻止编译，本次 RTT 与 Data Flash 实板测试也已通过。若出现其他警告，应单独查看，不能一概忽略。

导入后出现的“包含路径没有找到”警告，可能与尚未生成 FSP 文件有关；生成并重新构建后若仍存在，应检查具体路径。

## 8. 下载程序并继续运行

1. 连接板卡与调试器，给板卡上电。
2. 在当前工程的调试配置中核对目标器件和本次编译的 `.elf`。
3. 启动调试，下载程序。
4. 程序暂停时，点击 **Resume / 继续运行**，或按 `F8`。

下载本示例会替换板上的应用程序。后续测试使用的是独立 Flash LP 示例。

如果暂停后看到程序位于 `SEGGER_RTT_HasKey()`，调用栈中有 `hal_entry()`，这与示例等待 RTT 菜单输入的流程一致，不应仅凭停在这里就判断程序卡死。

截图中的 **Suspended** 表示 CPU 处于暂停状态；需要继续运行后，程序才能处理输入。

![程序暂停在 RTT 输入检查函数](picture/Pasted%20image%2020261014104142.png)

## 9. 打开 RTT Viewer 并连接

1. 保持 e² studio 调试连接。
2. 打开 **J-Link RTT Viewer**。
3. 在连接窗口中设置：
   - **Connection to J-Link**：`Existing Session`。
   - **RTT Control Block**：`Auto Detection`。
4. 点击 **OK / Connect**。
5. 确保 e² studio 中程序已继续运行。
6. 在 RTT Viewer 的 **Terminal 0** 或 **All Terminals** 中查看输出。

`Existing Session` 表示使用当前调试会话建立的 J-Link 连接。[SEGGER 官方连接说明](https://kb.segger.com/J-Link_RTT_Viewer)

![RTT Viewer 连接与交互窗口](picture/Pasted%20image%2020261014104558.png)

正常应出现：

```text
Enter 1 for Data Flash Operations
Enter 2 for Code Flash Operations
```

如果 RTT 已连接但没有看到菜单，在输入框发送 `x` 可请求重新显示主菜单。若仍无响应，先检查程序是否处于运行状态。

## 10. 执行 Data Flash 测试

本流程两次输入的数字都是 `1`，含义不同，应分开发送。

1. 在 RTT Viewer 底部输入框填写 `1`，点击 **Enter** 按钮，选择 **Data Flash Operations**。
2. 等待出现以下提示：

   ```text
   Enter valid block number for Data-Flash operations:
   ```

3. 再填写 `1` 并发送，选择 **Data Flash Block 1**。
4. 等待日志输出完成。

不要一次发送 `11`。本次不选择主菜单 `2`，该选项会执行 Code Flash 测试，涉及代码区和访问窗口设置。

程序执行顺序：

| 阶段 | 操作 |
| --- | --- |
| 初始化 | `R_FLASH_LP_Open()` |
| 空白检查 | `R_FLASH_LP_BlankCheck()` 检查待写区域 |
| 按需擦除 | 若区域非空，调用 `R_FLASH_LP_Erase()` 擦除所在块，并再次检查 |
| 写入 | `R_FLASH_LP_Write()` 写入 64 字节测试数据 `00～3F` |
| 读回比较 | 读取写入区域，通过 `memcmp()` 与原数据比较 |
| 关闭驱动 | `R_FLASH_LP_Close()` |

## 11. 判断测试结果

本次实板输出包括：

```text
FLASH_LP INIT SUCCESSFUL
Performing Data Flash operations
Write successful
...
30 31 32 33 34 35 36 37
38 39 3A 3B 3C 3D 3E 3F
Flash_LP Data FLASH Operations Successful
FLASH_LP DE-INIT SUCCESSFUL
```

其中：

- `30～3F` 是读回数据的末尾部分，以十六进制显示。
- `Data FLASH Operations Successful` 表示本次 Data Flash 流程与数据比较成功。
- `DE-INIT SUCCESSFUL` 表示驱动正常关闭。

![Data Flash 测试成功日志](picture/Pasted%20image%2020261014104733.png)

