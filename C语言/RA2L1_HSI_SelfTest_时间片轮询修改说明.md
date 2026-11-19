# RA2L1 HSI SelfTest 时间片轮询版说明

## 1. 最终文件和使用方式

修改后的代码文件：

```text
RA2L1_HSI_SelfTest_hal_entry_timeslice.c
```

原来的 `RA2L1_HSI_SelfTest_hal_entry_patched.c` 没有被覆盖，可以用来对照或恢复。

要在 e² studio 工程中使用新版代码：

1. 先备份工程中的 `src/hal_entry.c`。
2. 用 `RA2L1_HSI_SelfTest_hal_entry_timeslice.c` 的内容替换工程中的 `src/hal_entry.c`。
3. 不需要修改 FSP 自动生成的 `hal_data.c` 和 `hal_data.h`。
4. Clean Project，然后 Build Project。
5. 下载后，在上电或复位后的 10 秒内按一次 USER 按键，让 GPIO 测试通过。

新版程序上电后自动按以下顺序执行一次：

```text
GPIO -> ADC/DAC -> SPI -> I2C -> CAN -> 完成
```

没有串口菜单，也不需要输入命令。

## 2. 怎样查看测试结果

代码只保留了真正有用的自检结果：

```c
volatile test_status_t g_test_status[TEST_ID_COUNT];
volatile bool g_self_test_complete;
volatile bool g_self_test_pass;
```

`g_self_test_complete` 的含义：

- `false`：自检仍在进行。
- `true`：全部项目已经结束。

`g_self_test_pass` 的含义：

- `true`：所有项目全部通过。
- `false`：尚未完成，或者至少有一项失败。

`g_test_status[]` 保存每一项的结果，数组下标与测试项目的对应关系为：

| 数组元素 | 测试项目 |
|---|---|
| `g_test_status[0]` | GPIO |
| `g_test_status[1]` | ADC/DAC |
| `g_test_status[2]` | SPI |
| `g_test_status[3]` | I2C |
| `g_test_status[4]` | CAN |

每个元素的值为：

| 值 | 枚举名称 | 含义 |
|---:|---|---|
| `0` | `TEST_STATUS_NOT_TESTED` | 还没有测试 |
| `1` | `TEST_STATUS_PASS` | 测试通过 |
| `2` | `TEST_STATUS_FAIL` | 测试失败 |

初次调试时，可以把上面三个变量加入 e² studio 的 Expressions 窗口。正式产品中，应用程序可以读取 `g_self_test_complete` 和 `g_self_test_pass`，决定继续运行还是进入故障处理。

例如，全部测试完成后执行正常程序，可以把 `hal_entry()` 中的 `__NOP()` 替换为：

```c
if (g_self_test_pass)
{
    /* 在这里调用正常应用任务。 */
}
else
{
    /* 在这里执行故障指示，例如点亮故障 LED。 */
}
```

## 3. “时间片轮询”是什么意思

### 3.1 原程序的阻塞等待

原来的 SPI 测试大致是：

```c
for (uint32_t timeout = 0U; timeout < 1000U; timeout++)
{
    if (g_spi_transfer_done)
    {
        break;
    }

    R_BSP_SoftwareDelay(1U, BSP_DELAY_UNITS_MILLISECONDS);
}
```

CPU 会一直留在这个 `for` 循环中，直到 SPI 完成或等满 1000 ms。在此期间，主循环不能去运行其他任务。GPIO 原来甚至可能这样等待 10 秒。

### 3.2 新程序的一毫秒时间片

新版主循环只有一个统一的 1 ms 延时：

```c
while (1)
{
    if (BOOT_STATE_READY != g_boot_state)
    {
        poll_hardware_start();
    }
    else if (!g_self_test_complete)
    {
        poll_active_test();
    }
    else
    {
        __NOP();
    }

    R_BSP_SoftwareDelay(TIME_SLICE_MS, BSP_DELAY_UNITS_MILLISECONDS);
}
```

它每隔约 1 ms 询问一次：

1. 主晶振准备好了吗？
2. 当前是哪项测试？
3. 当前测试进行到哪一步？
4. 外设回调是否已经报告完成？
5. 是否超时？

每次只检查和处理一小步，随后立刻返回主循环。这就是这里所说的“时间片轮询”。

可以把它想象成做饭：旧写法是一直站在水壶旁等水烧开；新写法是看一眼水壶，没有烧开就先去做别的事，过一会再回来检查。

### 3.3 状态变量为什么必要

函数每次执行完都会返回，所以程序必须记住“上次做到哪里”。`g_test_step` 就是这张便签。

例如 GPIO 测试有两个步骤：

```text
TEST_STEP_GPIO_WAIT_BUTTON  等按键，最多 10 秒
            |
            | 检测到按键
            v
TEST_STEP_GPIO_LED_ON       LED 点亮 300 ms
            |
            v
测试结束
```

`g_wait_ms` 则是倒计时。主循环每运行一次，倒计时减少 1 ms。它替代了原来每个测试中重复出现的 `timeout` 循环。

## 4. 程序总体流程

程序的主要调用关系是：

```text
hal_entry()
    |
    +-- start_hardware()        打开 GPIO、ADC、DAC、SPI、I2C，启动主晶振
    |
    +-- while (1)               每约 1 ms 循环一次
          |
          +-- poll_hardware_start()
          |       等主晶振稳定，然后打开 CAN
          |
          +-- start_all_tests()
          |       从 GPIO 测试开始
          |
          +-- poll_active_test()
                  根据 g_active_test 调用当前项目的 poll 函数
                          |
                          +-- finish_test()
                                  保存结果并启动下一项
```

所有测试结束后，`finish_all_tests()` 检查五项结果。只有五项全部是 `PASS`，`g_self_test_pass` 才会变为 `true`。

## 5. 每项测试具体怎样改

### 5.1 GPIO

原写法：在一个 10000 次的循环中等待按键，每次延时 1 ms。

新写法：

1. `start_test(TEST_ID_GPIO)` 清除旧按键标志。
2. 设置 10000 ms 倒计时。
3. 每个时间片由 `poll_gpio_test()` 检查一次 `g_button_pressed`。
4. 按键中断回调只负责把 `g_button_pressed` 设为 `true`。
5. 检测到按键后点亮 LED，切换到 `TEST_STEP_GPIO_LED_ON`。
6. 再经过 300 ms 后熄灭 LED，GPIO 测试完成。

为什么这样改：等待按键是最慢的操作，不能让 CPU 在这里空等 10 秒。

注意：已经取消串口提示，因此下载或复位后要在 10 秒内按 USER 键。如果不想要人工参与，应当把 GPIO 自检改成输出脚到输入脚的硬件回环，而不是简单删除超时。

### 5.2 ADC/DAC

每个 DAC 测试值分成以下步骤：

1. 写入一个 DAC 值。
2. 设置 100 ms 稳定时间，立即返回主循环。
3. 时间到后启动 ADC 扫描。
4. 后续时间片不断调用 `R_ADC_StatusGet()` 检查扫描状态。
5. 扫描完成后读取 ADC，并比较 ADC 与 DAC 的差值。
6. 开始下一个 DAC 值。

`g_adc_index` 记住当前测试到第几个 DAC 值；`g_adc_pass` 记住前面是否出现过超差。

为什么保留 `g_adc_pass`：ADC/DAC 一共测试 6 个值。即使第 2 个值失败，程序仍会继续测完后面的值，因此必须记住前面已经失败过。这是算法状态，不是调试变量。

### 5.3 SPI

SPI 测试启动 `R_SPI_WriteRead()` 后立即返回。SPI 中断回调完成时设置：

```c
g_spi_event = p_args->event;
g_spi_transfer_done = true;
```

主循环中的 `poll_spi_test()` 每个时间片检查一次完成标志。完成后同时确认：

1. 回调事件是 `SPI_EVENT_TRANSFER_COMPLETE`。
2. 收到的 8 字节与发出的 8 字节完全一致。

如果 1000 ms 内没有完成，就判定失败。

原来的局部变量 `transfer_finished` 被删除，因为 `g_spi_transfer_done` 已经表达同一件事，没有必要保存两份相同信息。

### 5.4 I2C

I2C 自回环实际包含两段传输：

```text
主机发送 -> 从机接收 -> 比较数据
从机发送 -> 主机接收 -> 比较数据
```

因此保留两个状态：

- `TEST_STEP_I2C_WAIT_WRITE`
- `TEST_STEP_I2C_WAIT_READ`

主机和从机回调只记录“完成”“中止”和实际字节数。真正的数据比较和下一步启动都放在主循环中执行。

为什么不在中断回调中做全部工作：中断回调应该尽量短。回调停留太久会影响其他中断的响应，也会让程序更难排查。

### 5.5 CAN

CAN 使用外部主晶振，所以启动也分成时间片：

1. `R_CGC_ClockStart()` 只负责开始启动主晶振。
2. `poll_hardware_start()` 每 1 ms 调用一次 `R_CGC_ClockCheck()`。
3. 晶振稳定后才打开 CAN 并进入外部回环模式。
4. CAN 测试发送一帧，随后由 `poll_can_test()` 等待 TX 和 RX 两个完成标志。
5. 最后比较 ID、DLC 和 8 字节数据。

如果主晶振、CAN 打开或模式切换失败，其他测试仍会继续，最终 CAN 项显示 `FAIL`。

## 6. 为什么回调共享变量必须使用 volatile

例如：

```c
static volatile bool g_spi_transfer_done;
```

主循环读取这个变量，SPI 中断回调会在主循环不知道的时刻修改它。如果没有 `volatile`，编译器可能认为变量没有变化，重复使用寄存器里的旧值，主循环就可能永远看不到回调写入的 `true`。

`volatile` 的作用是告诉编译器：每次使用这个变量时都要重新访问内存。

但是，`volatile` 不是“所有全局变量都要加”。只在中断与主循环共享，或硬件可能改变的变量上使用。`g_active_test`、`g_test_step` 等只在主循环中使用，所以不需要 `volatile`。

## 7. 为什么异步收发缓冲区改成 static

SPI、I2C 和 CAN 传输是异步的。调用启动函数后，外设可能仍在后台使用缓冲区，而启动函数已经返回。

如果缓冲区是普通局部数组：

```c
static void bad_example(void)
{
    uint8_t receive[8];
    R_SPI_WriteRead(..., receive, ...);
} /* 函数返回后，receive 的有效期结束 */
```

函数返回后，该局部数组的栈空间可以被其他函数覆盖，外设继续写它会产生不可预期的结果。

新版把异步传输缓冲区定义成 `static`，它们在程序整个运行期间一直存在，因此外设可以安全使用。

## 8. 删除了哪些内容，为什么删除

### 8.1 整套串口界面

删除内容包括：

- `#include <stdio.h>` 和所有 `snprintf()`。
- UART 打开、UART 写函数和 UART 回调。
- 菜单和状态文本。
- 命令接收缓冲区。
- 命令大小写转换和空格整理。
- `HELP`、`BACK`、`ALL TEST` 等命令处理。
- 页面状态 `PAGE_MAIN`、`PAGE_RESULT`。
- 原 UART 测试。

原 UART 测试只是“收到 UART 命令后又发送了一段文本”，没有把发送数据接回并比较，因此并不是严格的 UART 自回环测试。既然不需要串口打印，连同它一起删除可以减少代码量和误解。

如果将来确实要测试 UART，建议连接 TXD 与 RXD，然后像 SPI 一样发送固定数据、接收并逐字节比较。

### 8.2 CAN 调试快照变量

删除了：

```text
g_can_test_done
g_can_test_pass
g_can_test_state
g_can_failure_reason
g_can_last_event
g_can_last_error
g_can_last_mailbox
g_can_last_fsp_error
```

这些变量主要用来在调试器中观察过程，而且有的信息互相重复。新版只保留算法真正需要的：

```text
g_can_tx_complete
g_can_rx_complete
g_can_error
g_can_rx_frame
```

CAN 的最终结果统一保存在 `g_test_status[TEST_ID_CAN]`，不再维护另一套结果变量。

### 8.3 重复的临时结果变量和打印缓冲区

删除了 `message[]`、`length`、`write_length`、`transfer_finished` 等只服务于打印或重复表达状态的变量。

## 9. 为什么代码行数没有按删除比例大幅下降

时间片程序通常会比最简单的阻塞程序多一些“状态管理”代码。这不是多余代码，而是把原来隐藏在 `for + delay` 中的执行过程明确写出来。

原来的思路是：

```text
启动 -> 一直等 -> 完成
```

现在是：

```text
启动 -> 保存状态 -> 返回主循环 -> 下个时间片检查 -> 完成
```

虽然多了状态枚举和 `poll_xxx_test()`，却带来了重要好处：主循环不会因为某项外设等待而停止，将来可以很容易在每个时间片中加入看门狗、通信、按键扫描或其他应用任务。

本次在去掉串口后，文件大小已经从约 29 KB 降到约 21 KB。

## 10. 仍然保留的特殊死循环

`stop_on_error()` 中仍有一个 `while (1)`：

```c
while (1)
{
    __NOP();
}
```

它只用于 GPIO、ADC、DAC、SPI、I2C 等基础驱动初始化失败的致命情况，不用于正常等待。基础外设都没有成功打开时，继续执行测试没有意义，所以程序把总结果设为失败后停住。

CAN 初始化失败是例外：程序会把 CAN 项判为失败，但仍然继续测试其他模块。

在正式产品中，`stop_on_error()` 可以替换成故障 LED、复位或安全状态处理。

## 11. 硬件条件不能省略

代码简化不会改变自回环测试所需的硬件连接：

- GPIO：需要在上电后 10 秒内按 USER 按键。
- ADC/DAC：DAC 输出必须实际连接到 ADC 输入通道 3。
- SPI：MOSI 与 MISO 必须按原工程的回环方式连接。
- I2C：主从实例和上拉电阻必须保持原测试工程配置。
- CAN：外部收发器、供电、共地、终端电阻和跳帽必须正确，CAN0 TX/RX 不能接反。

## 12. 时间精度说明

当前版本使用 `R_BSP_SoftwareDelay(1 ms)` 作为简单时间片，适合学习和当前裸机自检。实际一个循环还包含少量函数执行时间，因此 1000 次循环会略长于严格的 1000 ms，不适合作为高精度时基。

如果以后需要准确的 1 ms 调度，建议使用 AGT/GPT 硬件定时器中断产生系统节拍。状态机和各个 `poll_xxx_test()` 不需要推倒重写，只需把软件延时换成定时器节拍标志。

## 13. 编译检查结果

新版代码已经使用该 RA2L1 工程的 FSP 6.5.0 生成文件和 ARM GNU Toolchain 13.2 做过 `-fsyntax-only` 检查，并启用了工程原有的 `-Wall`、`-Wextra`、`-Wconversion`、`-Wshadow` 等警告选项。

检查结果：零错误、零警告。

这只能证明代码与当前 FSP 接口和 C 语法匹配。由于硬件回环条件无法在文件环境中模拟，最终仍需在开发板上分别确认五项结果。
