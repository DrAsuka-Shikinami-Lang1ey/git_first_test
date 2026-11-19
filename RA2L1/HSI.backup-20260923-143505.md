P004/60脚充电器反接检测，GPIO输入，高有效IRQ3 输入、下降沿引脚可以保留 IRQ3，但触发沿应改成上升沿

![](picture/Pasted%20image%2020260921172755.png)


## 

P003/61脚过流状态检测，GPIO输入，高有效ADC0 AN03删除 AN03，改 GPIO 输入

修改配置，删除ADC，然后添加过流检测代码


## 
P014/53脚放电MOS温感，ADC AN009DAC0 输出删除 DAC0，改 ADC0 AN009
![](picture/Pasted%20image%2020260921174350.png)
目前可以完成“采集放电MOS温感信号并计算NTC电阻”的要求。如果还需要输出具体温度（℃），还需要提供 NTC 的型号或 `R25/B值`、温阻表
## 
P100/48脚比较器输入 CMPIN0IIC1 SCL删除 IIC1 功能，改比较器输入
![](picture/Pasted%20image%2020260921180159.png)
添加比较器模块
==比较器模块==
参考源无法仅靠引脚表确定
## 
P101/47脚充放电MOS驱动电源开关，GPIO输出，高有效IIC1 SDA删除 IIC1 功能，改输出、初始低
![](picture/Pasted%20image%2020260921182531.png)
输入`MOS PWR ON`，才能使P101拉高


## 
P502/51脚外部温感，ADC AN018GPIO输出、初始低改 ADC0 AN018
![](picture/Pasted%20image%2020260921182949.png)
## 中断和输入
| 引脚       | 用途                | 建议配置                 |
| -------- | ----------------- | -------------------- |
| P001/63脚 | GPIO输入/IRQ7，唤醒MCU | IRQ7，上升沿             |
| P000/64脚 | GPIO输入/IRQ6，唤醒MCU | IRQ6，上升沿             |
| P411/12脚 | 充电电压唤醒            | IRQ4或GPIO输入，上升沿      |
| P208/20脚 | ACC状态检测           | GPIO输入               |
| P207/21脚 | 充电12V状态检测         | GPIO输入               |
| P303/29脚 | 电池包选中开关检测         | GPIO输入               |
| P302/30脚 | 电池包地址线检测          | GPIO输入；需要中断时可用IRQ5   |
| P215/6脚  | 负载状态检测，低有效        | GPIO输入               |
| P204/24脚 | AFE报警输入，低有效       | GPIO输入；该脚不能直接配置外部IRQ |
| P214/7脚  | GPIO输入            | 输入模式；注意它是输入专用端口      |

不知道具体原理图，所以没有设置内部上拉
IRQ  
## 输出
| 引脚       | 用途             | 建议初始状态         |
| -------- | -------------- | -------------- |
| P103/45脚 | GPIO输出         | 表中未给有效电平，需确认   |
| P403/4脚  | GPIO输出         | 需确认            |
| P112/37脚 | GPIO输出         | 需确认            |
| P400/1脚  | 4G模块工作电源使能，高有效 | 初始低            |
| P304/28脚 | 系统工作电源使能，高有效   | 初始低            |
| P501/50脚 | MCU工作状态指示，高有效  | 初始低            |
| P410/13脚 | MCU唤醒AFE，高有效   | 初始低            |
| P104/44脚 | 保险丝点爆低端，低有效    | **初始高**        |
| P500/49脚 | 保险丝点爆高端，高有效    | 初始低            |
| P105/43脚 | 过流检测功能开关，高有效   | 初始低            |
| P113/38脚 | 放电MOS切断开关，高有效  | 初始低            |
| P107/41脚 | 负载检测功能开关，高有效   | 初始低            |
| P106/42脚 | 充电电压检测开关，高有效   | 初始低            |
| P409/14脚 | AFE硬件复位，低有效    | **初始高**        |
| P002/62脚 | GPIO输出         | 功能和安全电平未写明，需确认 |
P409规定的“下拉超过8 ms复位、超过32 ms关断”属于控制时序，
### SPI、ADC、比较器

| 引脚       | 要求         | 建议配置                       |
| -------- | ---------- | -------------------------- |
| P301/31脚 | SPI CS     | SPI1 `SSLB2_B`，或者GPIO输出初始高 |
| P012/55脚 | 充电电压采样     | ADC0 AN007                 |
| P015/52脚 | 3.3V模拟电压采样 | ADC0 AN010                 |
| P013/54脚 | 充电MOS温感    | ADC0 AN008                 |
| P102/46脚 | 比较器输入      | CMPIN1                     |
ADC参考源仍需客户确认





现象

![](picture/Pasted%20image%2020260922114705.png)


## UART TEST
![](picture/Pasted%20image%2020260922133017.png)
![](picture/Pasted%20image%2020260922133351.png)


## 
![](picture/Pasted%20image%2020260922133720.png)




## 
========== ALL TEST START ==========

  

[UART TEST]

UART command received and response transmitted.

UART TEST PASS

  

[CHARGER REVERSE DETECTION]

P004=HIGH: charger reverse detection ACTIVE.

IRQ3 rising edge has not been detected since startup.

CHARGER REVERSE INPUT TEST PASS

  

[DIGITAL INPUT STATUS]

P001 MCU wake IRQ7: LOW, INACTIVE (high active).

P000 MCU wake IRQ6: LOW, INACTIVE (high active).

P411 charge voltage wake IRQ4: LOW, INACTIVE (high active).

P302 battery address IRQ5: HIGH, ACTIVE (high active).

P208 ACC status: HIGH, ACTIVE (high active).

P207 charge 12V status: HIGH, ACTIVE (high active).

P303 battery selected: LOW, INACTIVE (high active).

P215 load status: LOW, ACTIVE (low active).

P204 AFE alarm: HIGH, INACTIVE (low active).

P214 general input: LOW (active polarity not specified).

DIGITAL INPUT READ PASS

  

[OVERCURRENT STATUS]

P003=LOW, overcurrent=NORMAL; trigger=400 A (160 mV); P113 cutoff=LOW.

  

[DISCHARGE MOS TEMPERATURE INPUT]

P014/AN009 raw=1065, V_D_NTC=858 mV, VCC_3V3=1592 mV, R_D_NTC=11689 ohm.

  

[COMPARATOR INPUT TEST]

ACMPLP0 is not ready. Check Input Voltage and Reference Voltage in FSP.

  

[SPI TEST]

TX: 00 55 AA 12 34 A5 5A FF

RX: 00 00 00 00 00 00 00 00

SPI LOOPBACK FAIL

  

[I2C TEST]

IIC1 loopback was removed because P100 is now CMPIN0.

IIC0 master initialization: PASS

Bus transaction skipped: external slave address/protocol is not defined.

  

========== TEST SUMMARY ============

UART : TESTED - PASS

CHG REV : TESTED - PASS

INPUTS : TESTED - PASS

OC INPUT: TESTED - PASS

MOS TEMP: TESTED - PASS

CMP : TESTED - FAIL

SPI : TESTED - FAIL

I2C : TESTED - PASS

====================================