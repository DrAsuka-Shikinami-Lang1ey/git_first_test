MCU Deepstop模式，采用外部中断唤醒，定时器唤醒，休眠期间要保留看门狗功能，需要测量工作电流，休眠和正常工作要从硬件上看到区别

# DeepStop：
# Stand-by Conteoller(STBC):低功耗待机状态，功耗为微安级。
# SRAM = Static Random Access Memory 静态随机存取存储器。与DRAM区别就是不需要刷新，只要通电数据就能一直保存，所以速度较快（纳秒级），待机时功耗很低，但同时缺点是价格昂贵。
# HALT状态：CPU暂停，系统时钟继续运行，外设正常工作



## 例子：
```c
void Start_ADC_And_Wait(void) {
    ADC_Start_Conversion();
    adc_done_flag = 0;

    // 等待 ADC 转换完成中断
    while(adc_done_flag == 0) {
        __asm("HALT"); // CPU 暂停，等 ADC 转换完触发中断唤醒
    }
    
    // 唤醒后继续执行
    Read_ADC_Result();
}
```
### 专门处理ADC中断函数（ISR）
```c
void ADC_Interrupt_Handler(void){
     Clear_ADC_Interrupt_Flag();
     adc_done_flag = 1;//修改标志位
}
```
### _ _ arm("HALT");是直接使用汇编语言对CPU硬件下达最底层的命令。上面函数执行ADC启动函数时只是在很快的时间内向对应寄存器写一，然后进行下一步，由于此时ADC转换未完成，标志位未改变，所以程序会执行HALT指令。
# Stop模式：
## 通常会关闭的时钟（高速时钟）
### MainOSC、PLL1、HS IntOSC
## 通常会保留的时钟（低速时钟）
### LS IntOSC
## 1.首先要进入STOP模式，需要设置特定的寄存器位
```c
STBC0STPT.STBC0STPTRG = 1;
__asm("HALT");//一般配合HALT或WAIT指令才能真正进入低功耗状态？
```
### 2.STOP模式的灵活配置功能——可以选择哪些时钟域在STOP模式下继续保持运行
```c
CKSC_xxx_STPM.xxxSTPMSK

CKSC_PERI_STPM.PERISTPMSK = 1;//外设时钟在STOP模式下继续运行
CKSC_CPUCLKS_STPM.CPUSTPMSK = 0;//CPU时钟在STOP模式下停止
```
### 3.完整的STOP模式进入流程
```c
void Enter_STOP_Mode(void) {
    // 1. 配置时钟域掩码（选择哪些时钟在 STOP 模式下保持）
    CKSC_CPUCLKS_STPM.CPUSTPMSK = 0;    // CPU 时钟停止
    CKSC_TAU_STPM.TAUSTPMSK = 1;        // TAU 保持（用于唤醒）
    CKSC_IWDT_STPM.IWDTSTPMSK = 1;      // IWDT 保持
    
    // 2. 配置唤醒源（如 TAU 中断）
    TAU_Config_Interval_Timer_LS_IntOSC();  // 使用 LS IntOSC 作为时钟源
    INTC_Enable_IRQ(TAU_IRQn);
    
    // 3. 清除中断标志
    Clear_Interrupt_Flag(TAU_IRQn);
    
    // 4. 喂看门狗
    IWDT_Feed();
    
    // 5. 设置 STOP 模式触发位
    STBC0STPT.STBC0STPTRG = 1;
    
    // 6. 执行 HALT 指令进入 STOP 模式
    __asm("HALT");
    // 1. 清除中断标志 (防止下次进 STOP 立刻被唤醒)

        Clear_TAU_Flag();

        // 2. 喂看门狗 (防止复位)

        IWDT_Feed();
    
}
```
### 在实际工程中在Interval Timer模式下使用TAU0模块在Smart Configuratior设置。（12C。2配置时钟）
### 4.要执行喂狗保持STOP模式
### 5.CLMA
### ？？？CLMA0如果监测MainOSC，进入STOP模式就会导致系统立刻触发复位或中断。
### 6.Low Speed Internal Oscillator (LS IntOSC)
#### 它产生的时钟信号叫RL






--------------------------------------------
## MOSCE——主振荡器使能寄存器
## 用于启动和停止MainOSC,如果要更新这个寄存器，必须通过 `PROTCMD0` 寄存器执行**正确的写入保护解除/写入序列**

## MOSCS——主振荡器状态寄存器
## 用来查看MainOSC现在的状态
## 为什么要有状态寄存器？因为主振荡器不是一用MOSCE启动就立刻有用，要等待一段时间或轮询MOSCS状态，确认MainOSC稳定，再切换系统时钟到MainOSC。
## MOSCC——主振荡器控制寄存器
## 用于指定MainOSC（主振荡器）的放大增益。通常是和外接晶体/陶瓷振荡器的启动能力、稳定性、功耗有关。
## MOSCST——主振荡器稳定时间寄存器
## 用于决定MainOSC的稳定等待时间


![](%E9%99%84%E4%BB%B6/Pasted%20image%2020260629102053.png)
## DeepSTOP模式是一种芯片级待机模式，相比STOP模式可以进一步降低功耗。除了停止时钟供应之外，到隔离区域ISO的电源供应也会被关闭。
## 当STBC0PSC.STBC0DISTRG置为1时，进入DeepSTOP模式。
## ISO就是Isolated area 意思是隔离区域/隔离电源区域，在RH850中芯片内部会被分成不同电源/时钟区域，进入DeepSTOP时，某些区域会被断电，这个被断电的区域就叫ISO。与之对应的就是AWO，常供电区域。
## 查阅唤醒表，确定哪些唤醒源在DeepSTOP模式下能够使用
## 外部中断唤醒：TNMI
INTP0
INTP1
INTP2
INTP3
INTP4
INTP5
INTP6
INTP7
INTP8
INTP10
INTP11
INTP12

## 定时器唤醒：
WUTRG0 / WUTRG1    LPS0
INTTAUJ0I0~INTTAUJ0I3    TAUJ0
INTRTCA0IS / INTRTCA0AL / INTRTCA0R    RTCA0
INTTAUJ2I0~INTTAUJ2I3    TAUJ2


## DeepSTOP 唤醒配置核心：用 `WUFMSKx` 开唤醒源，用 `WUFx` 查唤醒原因，用 `WUFCx` 清唤醒标志。对你的外部中断唤醒、定时器唤醒、看门狗相关唤醒都直接有用。
## 查阅唤醒源和寄存器位的对应关系表
![](%E9%99%84%E4%BB%B6/Pasted%20image%2020260629115439.png)



F1KM-S1中写保护目标寄存器
![](%E9%99%84%E4%BB%B6/Pasted%20image%2020260629135856.png)


第一步Components 页：添加外部中断组件
![](%E9%99%84%E4%BB%B6/Pasted%20image%2020260629140521.png)
![](%E9%99%84%E4%BB%B6/Pasted%20image%2020260629184300.png)