![](picture/Pasted%20image%2020260709142144.png)

![](picture/Pasted%20image%2020260709142204.png)


```
g_ostm_started
g_ostm_error_count
g_ostm_irq_count
g_ostm_last_event
```




### 2，把period改为1ms
做GPIO 高低电平翻转周期
![](picture/Pasted%20image%2020260709161734.png)
```
OSTM 作为 1ms 系统时基，实现 ms 级延时函数
```

具体来说，现在代码完成了三件事：

1. **OSTM0 周期中断**
    
    - FSP 配置为 `Period = 1 ms`
    - 所以理论上每 1ms 进入一次 `ostm_callback()`
2. **1ms 系统 tick**
    
    - 每次进入中断，执行：
    
    ```
    g_ostm_irq_count++;
    ```
    
    - 这个变量就相当于系统运行了多少个 ms
3. **基于 tick 的毫秒延时**
    
    - 主循环里调用：
    
    ```
    DelayMs(500U);
    Led4Toggle();
    ```
    
    - 意思是每等待 500 个 tick 后翻转一次 LED

###  us级精度
FSP配置
![](picture/Pasted%20image%2020260709162710.png)

Counter Reload Disable ```做自由运行计时：Disable


**Interval timer mode**  
你前面的 `OSTM0` 用的就是这个。

它的工作方式是：

```
装入一个周期值
开始倒计时
到 0 或到比较点
产生中断
重新装入周期值
继续下一轮
```

**ree-running comparison mode**  
你现在给 `OSTM1` 配的就是这个。

它的工作方式更像秒表：

```
计数器启动后一直跑
不会每次都重新开始
程序可以随时读取当前计数值
用“现在的计数值 - 开始的计数值”计算经过了多久
```
start = 当前计数值;

while ((当前计数值 - start) < 目标计数)
{
}



| 对比项    | Interval timer mode              | Free-running comparison mode |
| ------ | -------------------------------- | ---------------------------- |
| 通俗理解   | 周期闹钟                             | 连续秒表                         |
| 计数方式   | 按设定周期计数，到点后重新开始                  | 计数器持续运行，不停地走                 |
| 典型用途   | 周期中断、系统 tick、定时任务                | us 延时、时间测量、代码运行时间统计          |
| 当前项目用途 | OSTM0 做 1ms tick                 | OSTM1 做 DelayUs              |
| 代码思路   | 每 1ms 进一次中断，`g_ostm_irq_count++` | 读取当前计数值，等待计数差达到目标            |
| 优点     | 周期稳定，适合系统节拍                      | 精度高，适合短时间测量                  |
| 缺点     | 短 us 延时会受中断开销影响                  | 需要代码自己计算计数差                  |
| 类比     | 定好时间自动响的闹钟                       | 一直运行的秒表                      |
![](picture/Pasted%20image%2020260709165524.png)


![](picture/Pasted%20image%2020260709165549.png)


delayus（100U）改为（50U）
同时完成了```
本实验使用 OSTM0 和 OSTM1 两个单元。OSTM0 配置为 interval timer mode，用于 1ms 系统节拍；OSTM1 配置为 free-running comparison mode，用于 us 级延时。两个 OSTM 实例初始化并启动后，通过状态查询确认均处于 TIMER_STATE_COUNTING 状态，并通过 GPIO 输出验证 ms/us 延时功能同时工作。



### 