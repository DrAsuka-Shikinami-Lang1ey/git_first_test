Config_OSTM0_user.c 定时器让端口8上的第五位引脚取高电平并与自己初始状态取异或来实现翻转。
```c

void R_Config_OSTM0_Create_UserInit(void)
{
    /* Start user code for user init. Do not edit comment generated here */
    /* End user code. Do not edit comment generated here */
}


{
    /* Start user code for r_Config_OSTM0_interrupt. Do not edit comment generated here */
   PORT.P8 ^= (1U << 5);    
    PORT.P0 ^= (1U << 14);   
    /* End user code. Do not edit comment generated here */
}

/* Start user code for adding. Do not edit comment generated here */
/* End user code. Do not edit comment generated here */

```


r_cg_main.c文件初始化led灯，创建定时器、启动定时器
```c
#include "r_smc_entry.h"

void r_main_userinit(void);


void main(void)
{
    r_main_userinit();
    /* Start user code for main. Do not edit comment generated here */
    
    PORT.P8 = _PORT_Pn5_OUTPUT_HIGH;
    PORT.P0 = _PORT_Pn14_OUTPUT_HIGH;
    
    R_Config_OSTM0_Create();
    //R_Config_TAUB0_0_Create();
    
    EI();
    
    
    
    R_Config_OSTM0_Start();
    //R_Config_TAUB0_0_Start();
    
    while (1)
    {
        
    }
    
    
}


void r_main_userinit(void)
{
    DI();
    /* Start user code for r_main_userinit. Do not edit comment generated here */
    /* End user code. Do not edit comment generated here */
    R_Systeminit();
    EI();
}
```

