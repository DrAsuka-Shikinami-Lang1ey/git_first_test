### L1-03：GPIO 多功能与中断控制（gpio_multifunction_ep）

  

| 项目 | 描述 |

|------|------|

| **功能目标** | 全面验证IOPORT的多种工作模式、复用功能、中断输入能力 |

| **涉及外设** | IOPORT、INTC（外部中断）、Noise Filter & Edge/Level Detector |

| **关键技术点** | ① 批量端口配置（方向/上下拉/开漏/驱动强度） ② S/W I/O控制复用模式 vs Direct I/O控制复用模式 ③ 外部中断输入（边沿/电平触发） ④ 数字噪声滤波器配置 ⑤ 输入数据读取与输出数据写入 ⑥ 未使用引脚处理 |

| **验证方法** | 按键触发外部中断→LED翻转；用示波器验证Direct模式下PWM直接输出（不经CPU） |

| **参考现有** | ioport_ep, icu_ep, eint_swint_ep |



## 修改达芬奇配置
![](picture/Pasted%20image%2020260810172231.png)




