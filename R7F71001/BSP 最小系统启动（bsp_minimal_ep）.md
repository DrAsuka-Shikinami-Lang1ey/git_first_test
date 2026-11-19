
时钟树配置，系统时钟树达到240Mhz
![](picture/Pasted%20image%2020260701103727.png)


![](picture/Pasted%20image%2020260701093945.png)
Pin脚配置
![](picture/Pasted%20image%2020260701094130.png)
引脚模式选择输出模式，初始低电平
输出类型选择Push-Pull，推挽输出，主动输出高电平或低电平
输入缓冲和双向模式以及反向输出都选择Disable


![](picture/Pasted%20image%2020260701095908.png)
改好配置后需要在Manage configuration里面选择当前的修改后的配置文件，然后点击generate project content



![](picture/Pasted%20image%2020260701101508.png)

g_bsp_pin_cfg 是生成给程序执行的“引脚配置表”。
没有这张表，程序不知道你在 RASC 里配了什么

```c
void hal_entry(void)

{

    /* TODO: add your own code here */

    while(1)

    {

        R_IOPORT_PinWrite(&g_ioport_ctrl,LED4,BSP_IO_LEVEL_HIGH);

        R_BSP_SoftwareDelay(500,BSP_DELAY_UNITS_MILLISECONDS);

        R_IOPORT_PinWrite(&g_ioport_ctrl,LED4,BSP_IO_LEVEL_LOW);

        R_BSP_SoftwareDelay(500,BSP_DELAY_UNITS_MILLISECONDS);

    }
```
`bsp_api.h` 会进一步包含 BSP 里的延时定义，所以能直接使用


![](picture/Pasted%20image%2020260701112721.png)
在Stacks下方配置串口
![](picture/Pasted%20image%2020260701111325.png)Channel: 0
Baud Rate: 115200
Data Bits: 8bits
Parity: None
Stop Bits: 1bit
Operation: Both TX RX
RLIN30TX: P00_2
RLIN30RX: P00_3
注意这几个设置，其他暂时保持默认



错误一
![](picture/Pasted%20image%2020260701135145.png)
原因：系统的实际时钟并不是24MHZ,查看丝印发现晶振是16MHZ

验证：将串口波特率换成76800，能正常显示，它与115200大约为2/3，恰好16MHZ与24MHZ关系也是2/3.
![](picture/Pasted%20image%2020260701141324.png)
改正方法：将时钟树Mainosc改为16MHZ（重点：改完之后一定要在VS code中将文件重新编译生成，再下载启动）


目标：
![](picture/Pasted%20image%2020260701155713.png)
![](picture/Pasted%20image%2020260701150911.png)

代码流程图：

```mermaid
flowchart TD
    A["芯片复位 / 上电"] --> B["进入启动流程"]
    B --> C["调用 R_BSP_WarmStart()"]

    C --> D{"WarmStart 事件类型"}
    D -->|BSP_WARM_START_RESET| E["复位阶段处理<br/>当前为空"]
    D -->|BSP_WARM_START_POST_C| F["C运行环境和系统时钟已初始化"]

    F --> G["R_BSP_GuardEnableAccessAll()<br/>打开寄存器写保护访问"]
    G --> H["R_BSP_ModuleStandbyEnableAll()<br/>打开外设模块时钟"]
    H --> I["IOPORT_CFG_OPEN()<br/>应用 g_bsp_pin_cfg 引脚配置"]

    I --> J["进入 hal_entry()"]

    J --> K["定义串口发送字符串<br/>R-Car U5L4 BSP Boot OK"]
    K --> L["g_uart0.p_api->open()<br/>打开 UART0"]

    L --> M{"UART 打开成功？"}
    M -->|是| N["g_uart0.p_api->write()<br/>发送启动字符串"]
    M -->|否| O["跳过串口发送"]

    N --> P["进入主循环"]
    O --> P

    P --> Q["LED4 输出 HIGH"]
    Q --> R["延时 500 ms"]
    R --> S["LED4 输出 LOW"]
    S --> T["延时 500 ms"]
    T --> P
```




UART代码

```c
static volatile uint8_t g_uart_tx_complete = 0;//标志位UART是否已经发送完成

void user_uart_callback(uart_callback_args_t * p_args)//UART回调函数在FSP配置里
{
    if (UART_EVENT_TX_COMPLETE == p_args->event)
    {
        g_uart_tx_complete = 1;
    }
}

void hal_entry(void)
{
    static uint8_t msg[] = "R-Car U5L4 BSP Boot OK\r\n";//要- 1因为 C 字符串结尾会自动带一个隐藏的 `\0`

    fsp_err_t err = g_uart0.p_api->open(g_uart0.p_ctrl, g_uart0.p_cfg);//```使用 g_uart0 的配置参数g_uart0.p_cfg，初始化 g_uart0 的运行状态 g_uart0.p_ctrl，调用 UART 驱动的 open 函数，并把结果保存到 err。
    if (FSP_SUCCESS == err)
    {
        g_uart_tx_complete = 0;

        err = g_uart0.p_api->write(g_uart0.p_ctrl, msg, sizeof(msg) - 1);

        while ((FSP_SUCCESS == err) && (0 == g_uart_tx_complete))
        {
            /* Wait for UART transmit complete. */
        }
    }

    while (1)
    {
        R_IOPORT_PinWrite(&g_ioport_ctrl, LED4, BSP_IO_LEVEL_HIGH);
        R_BSP_SoftwareDelay(500, BSP_DELAY_UNITS_MILLISECONDS);

        R_IOPORT_PinWrite(&g_ioport_ctrl, LED4, BSP_IO_LEVEL_LOW);
        R_BSP_SoftwareDelay(500, BSP_DELAY_UNITS_MILLISECONDS);
    }
}
```



















# 按照模板来写


# R-Car U5L4 BSP 最小系统启动与串口输出说明

> 本文记录了基于 R-Car U5L4 评估板完成 BSP 最小系统启动、LED 闪烁验证以及 UART 串口输出的过程。实验内容包括 FSP Smart Configurator 工程配置、GPIO 引脚配置、UART 驱动配置、USB 转串口硬件连接、串口输出验证，以及调试过程中遇到的问题和解决方法。
## 背景介绍

>在嵌入式开发中，BSP 最小系统启动是后续外设开发的基础。通过最小工程验证系统时钟、启动流程、GPIO 输出和串口打印，可以确认芯片、调试器、工程配置和基础外设均能正常工作。
  本实验的目标是让 R-Car U5L4 芯片从复位后正常启动，进入 `hal_entry()`，通过 LED 闪烁验证主循环运行，并通过 UART 输出：
```text
R-Car U5L4 BSP Boot OK
```
## 环境说明

- 操作系统：Windows
- 开发工具：Visual Studio Code
- 配置工具：Renesas FSP Smart Configurator
- 构建工具：CMake 3.31.8、Ninja
- 编译器：GCC Arm Toolchain 13.3.1
- 调试器：Renesas E2 Emulator
- 目标芯片：R7F711001EBFC

![](../L1-03%EF%BC%9AGPIO%20%E5%A4%9A%E5%8A%9F%E8%83%BD%E4%B8%8E%E4%B8%AD%E6%96%AD%E6%8E%A7%E5%88%B6%EF%BC%88gpio_multifunction_ep%EF%BC%89/picture/4b28ab98cbf421bd8d36814243b92af7_origin%281%29.jpg)
本次实验采用了由霆宝科技开发的 R-Car U5L4 评估板（型号: R7F711001EBFC）。
## 功能设计

>    本实验实现以下功能：
    1. 芯片从复位向量正常启动。
    2. 初始化系统时钟和必要系统寄存器。
    3. 配置 GPIO 引脚，控制 LED4 闪烁。
    4. 配置 RLIN3 UART 通道，实现串口启动信息输出。
    5. 通过串口工具验证启动打印信息。
## 硬件连接

### LED 连接

| 板子功能 | MCU 引脚 | 备注                |
| ---- | ------ | ----------------- |
| LED4 | AP0_0  | 配置为 GPIO 输出，初始低电平 |

### UART 连接

| MCU 引脚 | USB-UART 信号 | 备注            |
| ------ | ----------- | ------------- |
| P00_2  | TXD_USB     | MCU UART 发送引脚 |
| P00_3  | RXD_USB     | MCU UART 接收引脚 |
| GND    | GND         | 必须共地          |
![](picture/7ae0c6f541e2794b3d5eed584a163d9f_origin%281%29.jpg)
## 软件设计

>  本实验主要用到两个功能：
    1. GPIO 输出：控制 LED4 闪烁，用来验证程序已经正常运行。
    2. UART 串口输出：通过 USB 转串口输出启动信息，验证串口通信是否正常。

### 配置方法
在 FSP Smart Configurator 中完成以下配置：
 1. 配置 LED 引脚：
   - 将 `AP0_0` 配置为 GPIO 输出。
   - 设置 Symbolic Name 为 `LED4`。
   - 输出初始状态选择 `Initial Low`。
 
2. 配置 UART：
   - 添加 `UART Driver on r_rlin3_uart`。
   - 设置实例名为 `g_uart0`。
   - 通道选择 `Channel 0`。
   - 波特率设置为 `115200`。
   - 数据格式设置为 `8 数据位、1 停止位、无校验`。
   - 将 `RLIN30TX` 配置到 `P00_2`。
   - 将 `RLIN30RX` 配置到 `P00_3`。

3. 配置引脚生成：
   - 在 Pins 页面勾选 `Generate data`。
   - 名称填写 `g_bsp_pin_cfg`。
   - 点击 `Generate Project Content` 生成代码。

4. 配置系统时钟：
   - 根据原理图，主晶振为 `16MHz`。
   - 因此 `Main OSC` 应设置为 `16MHz`。

### 代码编写

>程序进入 `hal_entry()` 后，先打开 UART，然后发送启动字符串，最后进入 LED 闪烁循环。
  主要流程如下：
  打开 UART
  发送 "R-Car U5L4 BSP Boot OK"
  循环控制 LED4 闪烁

核心代码如下：

```c
static volatile uint8_t g_uart_tx_complete = 0;
void user_uart_callback(uart_callback_args_t * p_args)
{
    if (UART_EVENT_TX_COMPLETE == p_args->event)

    {
        g_uart_tx_complete = 1;
    }

}
 void hal_entry(void)

{
    /* TODO: add your own code here */

    static uint8_t message[] = "R-Car U5L4 BSP Boot OK\r\n";
    fsp_err_t err = g_uart0.p_api->open(g_uart0.p_ctrl, g_uart0.p_cfg);
    if (FSP_SUCCESS == err)

    {

        g_uart_tx_complete = 0;

        err = g_uart0.p_api->write(g_uart0.p_ctrl, message, sizeof(message) - 1);

        while ((FSP_SUCCESS == err) && (0 == g_uart_tx_complete))

        {

        }

    }

    while(1)

    {
        R_IOPORT_PinWrite(&g_ioport_ctrl,LED4,BSP_IO_LEVEL_HIGH);
        R_BSP_SoftwareDelay(500,BSP_DELAY_UNITS_MILLISECONDS);
        R_IOPORT_PinWrite(&g_ioport_ctrl,LED4,BSP_IO_LEVEL_LOW);
        R_BSP_SoftwareDelay(500,BSP_DELAY_UNITS_MILLISECONDS);
    }  

#if BSP_TZ_SECURE_BUILD

    /* Enter non-secure code */

    R_BSP_NonSecureEnter();

#endif

}
```


## 功能验证

### LED 验证

>程序下载后，观察 LED4 是否以约 1Hz 频率闪烁。如果 LED 正常闪烁，说明程序已经进入 `hal_entry()` 主循环，GPIO 配置和基本启动流程正常。

### 串口验证
>正常情况下，复位后串口工具应显示：
```text
R-Car U5L4 BSP Boot OK
```
![](picture/Pasted%20image%2020260701171129.png)
## 注意事项
>
    1. `g_bsp_pin_cfg` 必须生成，否则 FSP 中配置的引脚不会真正应用到代码中。
    2. UART 通道和引脚必须匹配：`Channel 0` 对应 `RLIN30TX/RLIN30RX`。
    3. 硬件晶振频率必须和 FSP Main OSC配置一致。
    4. 如果串口乱码，优先检查波特率、主晶振配置、TX/RX 跳线和 UART 引脚配置。
