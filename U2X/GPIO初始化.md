## 达芬奇配置
## Pull Up Option：内部上拉

勾选后，引脚通过一个内部弱电阻连接到电源：

```
VCC
 │
[内部上拉电阻]
 │
引脚
```

当外部没有驱动信号时，引脚默认读取为高电平。

常见用途是低电平有效的按键


## Pull Down Option：内部下拉

勾选后，引脚通过内部弱电阻连接到GND：

```
引脚
 │
[内部下拉电阻]
 │
GND
```

当外部没有驱动信号时，引脚默认读取为低电平。

常见用途是高电平有效的按键：

### 时钟配置
![](picture/Pasted%20image%2020260810141945.png)
外部主晶振按20 MHz配置

![](picture/Pasted%20image%2020260810142057.png)
## 寄存器
![](picture/Pasted%20image%2020260810142611.png)

![](picture/Pasted%20image%2020260810142946.png)

```c
OPBT11 = 0xEFFFFFFF
CKDIVMD[2:0] = 111b
```

![](picture/Pasted%20image%2020260810143926.png)
1. CPU0进入App_Task()
2. 调用R_CLK_Init()
3. 检查MainOSC状态
4. 如果MainOSC未运行，写MOSCE=1启动
5. 等待MOSCS=3，表示MainOSC已经启动并稳定
6. 检查PLL状态
7. 如果PLL未运行，写PLLE=1启动
8. 等待PLLS=3，表示PLL已经启动并稳定

## 其他模块时钟配置
![](picture/Pasted%20image%2020260810144347.png)![](picture/Pasted%20image%2020260810144415.png)
## LED闪烁


![](picture/Pasted%20image%2020260810101829.png)
![](picture/Pasted%20image%2020260810101842.png)

| 选项                          | 当前值          | 作用                           |
| --------------------------- | ------------ | ---------------------------- |
| Bi Direction Control        | 未勾选          | 同时启用输入和输出路径，用于双向引脚           |
| Input Buffer Control        | 未勾选          | 启用数字输入缓冲器                    |
| Input Selection             | `SHMT1`      | 输入缓冲器的施密特触发特性                |
| Ip Control                  | 未勾选          | 控制外设功能的输入路径，即PIPC            |
| Pin Dio Alt Mode Changeable | 未勾选          | 是否允许运行时在DIO和复用功能间切换          |
| Pin Safe State Control      | 未勾选          | 芯片检测到安全故障时，是否由ECM强制该引脚进入安全状态 |
| Pin Safe State Trigger      | `ERROROUTZ0` | 选择哪一个错误输出信号触发安全状态            |
### DIO模块配置
![](picture/Pasted%20image%2020260810145222.png)

`Flip Channel Api`开启生成 `Dio_FlipChannel()`，用于把输出电平直接翻转
### LED闪烁代码
```c
Port_Init(Port_Config);
Dio_WriteChannel(DioConf_DioChannel_EN_HVIL2_NEG, STD_LOW);

if (PE_CPU0 == (Lu4Peid & 0x1FUL))
{
    R_Delay_Ms(1000U);
    (void)Dio_FlipChannel(DioConf_DioChannel_EN_HVIL2_NEG);
    R_FUN_EXEC_CYCLE();
}
```