## 1、1.09V 的 VDD 在低功耗时外部关掉，其他 5V、3.3V 保持
进入 DeepSTOP
      ↓
PWRCTL = L
      ↓
外部电源关闭 VDD
      ↓
发生有效唤醒事件
      ↓
PWRCTL = H
      ↓
外部电源重新提供 VDD
      ↓
等待 VDD 稳定，再解除 DeepSTOP 复位
## 2、模拟电源、数字电源及模拟地、数字地建议分开
目的在于减少数字开关噪声对 ADC 和模拟参考的干扰
## 3、ERROROUT、VMONOUT、PWRCTL 的用途及是否可以悬空
| 信号       | 用途                   | 典型接收端        |
| -------- | -------------------- | ------------ |
| ERROROUT | 向外部报告内部错误管理逻辑检测到的故障  | PMIC、外部监控电路  |
| VMONOUT  | 向外部报告电压监测相关状态        | PMIC、复位或安全电路 |
| PWRCTL   | 控制特定低功耗流程中的外部 VDD 供电 | 电源使能或时序电路    |
## 4、VCC 根据需求划分模拟、数字电源；APxx 属于模拟供电域
## 5、休眠时要保持模拟VCC和数字VCC常电，即3.3V或5V常电，防止电流倒灌，VDD可断电
## 6、EFTNB1RFFCLK，如果主晶振使用20M可以使用这个引脚，如果不是20M就用外部25M晶振提供，且外部晶振更容易过EMC;
## 7、U2A 硬件设计指导拆分为多个文档，旧版存在错误
## 8、如果MCU使用的是U2Cx，且有使用PMIC，U2Cx的ERROROUT和VMONOUT是同一管脚，接到PMIC上的ERROROUT管脚就行，PMIC上的VMONOUT保持悬空即可
## 9、EnVCC供3.3V AnVCC供5V时，低精度ADC使用受限，因为参考电压不一样
## 10、在审核原理图的时候，一定要注意中断的对应的管脚，是否能独立出来，有关中断是和其他功能绑定在一块的，比如P2_6的CANORX与INTPO绑定在一块，那么P2_6分配为INTP功能时，就会自动配置CANORX，其他管脚也就无法再配置CANORX了
## 11、当使用U2Ax+RAA27084的搭配方式时，请勿使用硬件指导手册内的case3，因为PWRCTL与ERROROUT电平不一致，会有烧坏PMIC的可能，需要规避
## 12、邻脚耦合噪声:切忌在一个正在采样的通道旁脚施加数字脉冲，否则可能通过硅衬底或封装耦合到采样电路，产生抖动
## 13、PWM 输入、输出放在两个定时器单元上
避免生成 PWM 与测量 PWM 争用同一组计数资源
## 14、AnVCC和AnREFH的供电电源不一致时，确保AnVCC时刻大于等于AnREFH，尤其是掉电时要避免AnVCC比AnREFH掉电快的情况，否则在掉电时电源会出现波形回钩
## 15、管脚分配初期，如果后期需要上功能安全，确定Resetout管脚是否使用，如果无法确定，建议预留
MCU发生内部复位，会拉低RESETOUT和ERROROUT管脚，如果PMIC通过ERROROUT管脚判断
MCU状态，在MCU内部复位后，可能会导致PMIC再拉低一次RESET，导致两次复位，在复位源上能
看见两个复位源，内部复位源和外部复位源，当然可能也有上电复位;
## 16、AWOVCL 的 T15 留空，是允许的电容连接方案
硬件手册 Section 55.2.1.5 Regulator Characteristics, Table 55.7 前的 Note 1 明确规,
AWOVCL电容可以采用以下两种方式:
1.将器件所需的总电容量均匀分配到所有AWOVCL管脚;
2.将器件所需的总电容量集中连接到一个AWOVCL管脚。
如果采用第二种“集中式”接法，电容必须连接到A22;其余没有连接电容的AWOVCL管脚必须保
持开路。手册原文中明确列出了:
### 其他补充





## 流程
### 一、选择型号
![](picture/Pasted%20image%2020261012120824.png)
## 二、调整时钟树
![](picture/Pasted%20image%2020261012121426.png)

将主时钟改为16Mhz，并且把CPU路径选择为主时钟
## 1.普通 GPIO 输入
![](picture/Pasted%20image%2020261012135505.png)
![](picture/Pasted%20image%2020261012135534.png)
![](picture/Pasted%20image%2020261012135554.png)
![](picture/Pasted%20image%2020261012135616.png)
## 2.电源状态 PG 输入
| 物理脚号 | 端口         | 信号             |
| ---- | ---------- | -------------- |
| 50   | **P1_11**  | VCCST_PWRGD    |
| 116  | **P20_2**  | V1P8A_PG       |
| 117  | **P20_1**  | V5A_PG         |
| 118  | **P20_0**  | VDD5V_PG       |
| 119  | **P20_5**  | VCCIO_PG       |
| 144  | **P18_1**  | V3P3A_PG       |
| 146  | **P18_3**  | V1P5_RTC_PG    |
| 148  | **P18_5**  | VCCPRIM_3P3_PG |
| 150  | **P18_7**  | SSD_PWR3_PG    |
| 153  | **P10_7**  | SSD_PWR1_PG    |
| 156  | **P10_10** | SSD_PWR2_PG    |
| 159  | **P10_13** | ETH_0.9V_PG    |
| 166  | **P11_6**  | VNNAON_PG      |
| 170  | **P12_1**  | FAN_5V_PG      |
## 3.第三组普通状态输入
| 物理脚号 | 端口        | 信号                             | 配置        |
| ---- | --------- | ------------------------------ | --------- |
| 49   | **P2_0**  | RSMRST_PWRGD_N                 | In        |
| 75   | **P1_4**  | MCU_SLP_S3_33                  | In        |
| 76   | **P2_4**  | MCU_SLP_S4_33                  | In        |
| 77   | **P2_5**  | MCU_SLP_S5_33                  | In        |
| 78   | **P1_14** | PLT_MCU_RST_33                 | In        |
| 39   | **P8_10** | CPU_C10_GATE_33                | In        |
| 40   | **P8_11** | PROCHOT_EC_33，原理图写作 PROCHOT_EC | In        |
| 41   | **P8_12** | PD_EC_SMBUS1_ALERT_0_N         | In，暂按普通输入 |
|      |           |                                |           |
## 4.调试 UART
![](picture/Pasted%20image%2020261012145629.png)
###  配置 MCU 与 SoC 通信的 UART

## 5.四个外部中断
| 物理脚号 | 引脚   | 中断资源      | 信号           |
| ---- | ---- | --------- | ------------ |
| 81   | P8_1 | **INTP5** | IGN_DET      |
| 82   | P8_3 | **INTP7** | MCU_CAN0_INH |
| 83   | P8_4 | **INTP8** | B_DET_MCU    |
| 84   | P8_5 | **INTP9** | ACC_DET      |
![](picture/Pasted%20image%2020261012152059.png)
## 6.**RIIC0**和RIIC1
![](picture/Pasted%20image%2020261012154751.png)
## 7.ADCA0模拟输入
![](picture/Pasted%20image%2020261012161151.png)
## 8.SoC与MCU之间的SPI（CSIH0）
![](picture/Pasted%20image%2020261012161645.png)
## 9.CSIH0和CSIH2
**SoC是主机、MCU是从机**。因此MCU的片选必须使用从机片选输入 _CSIH0SSI

![](file:///C:\Users\TPSZ-1~1\AppData\Local\Temp\ksohtml29620\wps1.jpg) 

![](file:///C:\Users\TPSZ-1~1\AppData\Local\Temp\ksohtml29620\wps2.jpg)
## 10.CAN通道
![](picture/Pasted%20image%2020261012163502.png)![](picture/Pasted%20image%2020261012163517.png)
## 11.定时器相关
![](picture/Pasted%20image%2020261012164940.png)![](picture/Pasted%20image%2020261012165140.png)
![](picture/Pasted%20image%2020261012170252.png)
## 12.使能引脚
| MCU引脚  | 网络             |
| ------ | -------------- |
| P12_3  | MCU_VCCST_EN   |
| P2_1   | EC_USTPWR_EN   |
| P0_8   | MCU_LPDDR5X_EN |
| P0_7   | MCU_BAT12V_EN1 |
| P1_5   | MCU_VBATA_EN   |
| P1_15  | MCU_IMVP_EN_33 |
| P9_2   | V5A_EN         |
| P9_3   | VDD5V_EN       |
| P20_3  | V1P8A_EN       |
| P20_4  | VCCIO_EN       |
| P18_0  | V3P3A_EN       |
| P18_2  | V1P5_RTC_EN    |
| P18_4  | VCCPRIM_3P3_EN |
| P18_6  | SSD_PWR3_EN    |
| P10_6  | SSD_PWR1_EN    |
| P10_9  | SSD_PWR2_EN    |
| P10_12 | ETH_0.9V_EN    |
| P10_14 | ETH_3.3V_EN    |
| P11_1  | ETH_1.8V_EN    |
| P11_5  | VNNAON_EN      |
| P11_7  | V1P8S_EN       |
| P12_0  | FAN_5V_EN      |
| P12_2  | USB_5V_EN      |
## 13.控制、复位和状态类GPIO输出
|MCU引脚|网络|分类|
|---|---|---|
|P10_15|GPP_V01_ACPRESENT|状态输出|
|P11_8|SYS_PWROK_EC_LS|电源状态输出|
|P11_9|EC_PM_PCH_PWROK|电源状态输出|
|P11_11|SMC_WAKE_SCI_N|低有效唤醒输出|
|P2_6|MCU_CAN1_STB|CAN收发器待机控制|
|P1_8|ANX7493_FLIP|接口方向控制|
|P2_3|RSMRST_R_N_LS|低有效复位输出|
|P2_2|ANX7493_EN|芯片使能输出|
|JP0_6|PDN_E3_N|低有效控制输出|
|P8_6|WD_SET|外部看门狗控制|
|P10_0|GPP_B04_STRAP_EC|SoC启动配置输出|
|P10_1|EC_GPPC_B14|SoC启动配置输出|
|P10_2|PM_PWRBTN_N|低有效电源按键输出|

76页
## 电容部分

