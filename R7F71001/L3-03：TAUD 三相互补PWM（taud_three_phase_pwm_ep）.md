### L3-03：TAUD 三相互补PWM（taud_three_phase_pwm_ep）

  

| 项目 | 描述 |

|------|------|

| **功能目标** | 利用TAUD + PIC实现带死区插入的三相互补PWM，用于电机驱动 |

| **涉及外设** | TAUD、PIC1、TAPA |

| **关键技术点** | ① 三相6路PWM生成 ② 死区时间精确插入（通过PIC） ③ 互补输出（高边/低边） ④ TAUD同步启动 ⑤ 紧急Hi-Z关断（TAPA输入） ⑥ PWM频率与占空比动态调整 ⑦ 与ADCK触发联动 |

| **验证方法** | 示波器观察6路PWM波形，验证死区时间和互补关系；模拟过流触发TAPA Hi-Z关断 |

| **参考现有** | taud_three_phase_ep |
![](picture/Pasted%20image%2020260709175929.png)

![](picture/Pasted%20image%2020260709180000.png)


![](picture/Pasted%20image%2020260709180021.png)


![](picture/Pasted%20image%2020260709180040.png)


## FSP配置
![](picture/Pasted%20image%2020260710094917.png)

![](picture/Pasted%20image%2020260710095034.png)


## 引脚设置
![](picture/Pasted%20image%2020260710095315.png)
## channel2配置
![](picture/Pasted%20image%2020260710095859.png)

![](picture/Pasted%20image%2020260710100005.png)


![](picture/Pasted%20image%2020260710100044.png)

## g_tap0配置
![](picture/Pasted%20image%2020260710100519.png)




![](picture/Pasted%20image%2020260710103714.png)

占空比每秒变化
![](picture/Pasted%20image%2020260710110013.png)



![](picture/Pasted%20image%2020260710110029.png)

## 死区时间
![](picture/Pasted%20image%2020260710110900.png)

## TAPA软件触发Hi-Z关断
![](picture/Pasted%20image%2020260710111440.png)

![](picture/Pasted%20image%2020260710112613.png)


`TAPA0ESO` 分配实际引脚，通过外部电平模拟过流，验证硬件触发关断。
![](picture/Pasted%20image%2020260710113828.png)


![](picture/Pasted%20image%2020260710114030.png)


![](picture/Pasted%20image%2020260710114321.png)


g_tapa_start_done: 1
g_tapa_error_step: 0
g_tapa_hiz_active: 1
g_tapa_hiz_monitor: 7
g_tapa_hiz_trigger_count: 1
g_tapa_hiz_release_attempt_count: 0
g_tapa_hiz_release_count: 0

![](picture/Pasted%20image%2020260710134758.png)

![](picture/Pasted%20image%2020260710140420.png)


![](picture/Pasted%20image%2020260710140322.png)


## ADC
![](picture/Pasted%20image%2020260710142914.png)

![](picture/Pasted%20image%2020260710143054.png)



![](picture/Pasted%20image%2020260710145033.png)


问题
![](picture/Pasted%20image%2020260710151353.png)

FSP 要求 **Trigger 0 和 Trigger 1 两个都不能是 Disable**。  
所以不能只开 `A/D Conversion Trigger 0`，还必须也给 `A/D Conversion Trigger 1` 选一个有效项。

![](picture/Pasted%20image%2020260710152150.png)

![](picture/Pasted%20image%2020260710155819.png)





![](picture/Pasted%20image%2020260713095012.png)
硬件触发改为软件触发

ADCK本身没有问题，问题出在TAPA/PIC触发链路

