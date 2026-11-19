
flowchart TD

    A["系统启动"] --> B["WarmStart 中打开 g_bsp_pin_cfg"]

    B --> C["LED4 输出低电平"]

    C --> D["打开并启动 g_timer0 PWM"]

    D --> E["打开并使能 g_external_irq0"]

    E --> F["开启全局中断"]

    F --> G["进入主循环"]

    G --> H{"是否检测到按键中断计数变化"}

    H -- "是" --> I["延时 50 ms 消抖"]

    I --> J["翻转 LED4"]

    J --> G

    H -- "否" --> G

    D --> K["P10_2 持续输出 PWM"]






#### 3. 主函数初始化流程

  

```c

void hal_entry(void)

{

    fsp_err_t err = FSP_SUCCESS;

    uint32_t handled_irq_count = 0;

  

    led_write(BSP_IO_LEVEL_LOW);

  

    err = g_timer0.p_api->open(g_timer0.p_ctrl, g_timer0.p_cfg);

    if (FSP_SUCCESS != err)

    {

        while (1) {}

    }

  

    err = g_timer0.p_api->start(g_timer0.p_ctrl);

    if (FSP_SUCCESS != err)

    {

        while (1) {}

    }

  

    err = g_external_irq0.p_api->open(g_external_irq0.p_ctrl, g_external_irq0.p_cfg);

    if (FSP_SUCCESS != err)

    {

        while (1) {}

    }

  

    err = g_external_irq0.p_api->enable(g_external_irq0.p_ctrl);

    if (FSP_SUCCESS != err)

    {

        while (1) {}

    }

  

    __enable_irq();

  

    while (1)

    {

        if (handled_irq_count != g_key_irq_count)

        {

            handled_irq_count = g_key_irq_count;

            R_BSP_SoftwareDelay(KEY_DEBOUNCE_MS, BSP_DELAY_UNITS_MILLISECONDS);

            led_toggle();

        }

    }

}

```

  

程序启动后先初始化 LED 状态，再打开并启动 PWM，然后打开并使能外部中断。主循环中通过比较 `handled_irq_count` 和 `g_key_irq_count` 判断是否有新的按键事件，如果有则延时 50 ms 消抖并翻转 LED。

  

#### 4. WarmStart 引脚初始化

  

```c

void R_BSP_WarmStart (bsp_warm_start_event_t event)

{

    if (BSP_WARM_START_POST_C == event)

    {

        R_BSP_GuardEnableAccessAll();

        R_BSP_ModuleStandbyEnableAll();

        IOPORT_CFG_OPEN(&IOPORT_CFG_CTRL, &IOPORT_CFG_NAME);

    }

}

```

  

`WarmStart` 是 BSP 在系统启动过程中自动调用的函数。本实验在 `BSP_WARM_START_POST_C` 阶段打开模块访问权限、解除模块待机，并加载 FSP 生成的引脚配置。




## QA

  

**Q1：为什么按键中断回调里不直接翻转 LED？**

  

中断回调函数应该尽量短，只记录事件即可。LED 翻转和消抖放到主循环中处理，更容易维护，也能减少中断中执行较长延时或复杂逻辑带来的问题。

  

**Q2：为什么 PWM 输出说是不经过 CPU？**

  

CPU 只在初始化阶段调用 `open()` 和 `start()` 启动 PWM。启动后，PWGC 外设根据寄存器配置自动产生周期波形，CPU 不需要在主循环里不断拉高、拉低引脚。

  

**Q3：为什么要配置 `g_bsp_pin_cfg`？**

  

`g_bsp_pin_cfg` 是 FSP 根据 Pin 配置生成的引脚配置表。程序启动时调用 `IOPORT_CFG_OPEN()` 加载这张表，引脚的 GPIO、IRQ、PWM 等功能才会真正生效。