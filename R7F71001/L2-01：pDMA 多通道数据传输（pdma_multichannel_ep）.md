
| 项目        | 描述                                                                                                               |
| --------- | ---------------------------------------------------------------------------------------------------------------- |
| **功能目标**  | 实现pDMAC的内存到内存、内存到外设、外设到内存传输，覆盖全部3种传输模式                                                                           |
| **涉及外设**  | pDMAC（16通道）                                                                                                      |
| **关键技术点** | ① 通道优先级与仲裁 ② Single transfer（单次） ③ Block transfer 1（按传输次数） ④ Block transfer 2（按地址计数） ⑤ 传输完成中断 ⑥ 多通道并发传输 ⑦ 链式传输配置 |
| **验证方法**  | 内存填充测试（验证速度和正确性）；外设到内存（如UART RX DMA）验证；传输时间测量                                                                    |
| **参考现有**  | dmac_ep                                                                                                          |
第一步配置 DMAC Stack 属性
![](picture/Pasted%20image%2020260706133758.png)
第二步 配置
![](picture/Pasted%20image%2020260706140037.png)
可靠性/访问保护相关功能不用修改

![](picture/Pasted%20image%2020260706140213.png)
内存到内存
源和目的都要递增
![](picture/Pasted%20image%2020260707100810.png)
# 添加断点

![](picture/Pasted%20image%2020260707162518.png)
数组传输成功
Memory to Memory + Block transfer 1 + Software trigger + Completion interrupt：PASS



# ![](picture/Pasted%20image%2020260707165615.png)

Single transfer + Software trigger + Completion interrupt：PASS



# 几种模式的区别
singer transfer 
block transfer 1
block transfer 2
模式                一次请求搬多少              适合场景

Single transfer     1 个数据                    UART/ADC 这种一个个来的数据

Block transfer 1    一整块数据                  内存复制、一次性搬 buffer

Block transfer 2    按组搬，多组数据             分段 buffer、多行数据、二维数组、帧数据


Reload Number of Transfer：下一轮重新装载“每组搬几个”
Reload Address Reload Counter：下一轮重新装载“搬几组”



## 有错误版
g_dmac_complete: 1 '\001'
g_dmac_error: 0 '\000'
g_dmac_result: -2
g_dmac_src: [20]
g_dmac_dst: [20]
g_dmac_request_count: 2
g_dmac_reset_count: 5
g_dmac_expected_total: 20
g_dmac_remaining: 16


## 正确版
g_dmac_complete: 1 '\001'
g_dmac_error: 0 '\000'
g_dmac_result: 0
g_dmac_src: [20]
g_dmac_dst: [20]
g_dmac_request_count: 1
g_dmac_reset_count: 20
g_dmac_expected_total: 20
g_dmac_remaining: 20

![](picture/Pasted%20image%2020260708093001.png)


# 多通道Block transfer 1
g_dmac0_complete: 1 '\001'
g_dmac_error: 0 '\000'
g_dmac_result: 0
g_dmac0_src: [20]
g_dmac0_dst: [20]
g_dmac0_request_count: 1
g_dmac1_complete: 1 '\001'
g_dmac1_src: [20]
g_dmac1_dst: [20]
g_dmac1_request_count: 1

# 链式传输配置 Chain Function
Always chain 每次传输请求都触发链式动作,Chain at the last transfer最后一次传输完成后才触发下一通道
![](picture/Pasted%20image%2020260708095535.png)

g_dmac0_complete: 1 '\001'
g_dmac_error: 0 '\000'
g_dmac_result: 0
g_dmac0_src: [20]
g_dmac0_dst: [20]
g_dmac0_request_count: 1
g_dmac1_complete: 1 '\001'
g_dmac1_src: [20]
g_dmac1_dst: [20]
g_dmac1_request_count: 0





## UART配置
![](picture/Pasted%20image%2020260708102929.png)

让 **UART 下面挂 DMA**，这一项必须启用：
Transfer API Support = Enabled


## stacks页面
![](picture/Pasted%20image%2020260708110847.png)
## 有问题，，发送两个AA之后就没有反应了
![](picture/Pasted%20image%2020260708112911.png)


g_uart_dma_echo.rx_data: 170 '\252'
g_uart_dma_echo.tx_data: 187 '\273'
g_uart_dma_echo.rx_complete: 1 '\001'
g_uart_dma_echo.tx_complete: 0 '\000'
g_uart_dma_echo.rx_complete: 1 '\001'
g_uart_dma_echo.ok_count: 1
g_uart_dma_echo.wrong_data_count: 0
g_uart_dma_echo.error_count: 0


## 改正之后
![](picture/Pasted%20image%2020260708114243.png)


![](picture/Pasted%20image%2020260708141721.png)




