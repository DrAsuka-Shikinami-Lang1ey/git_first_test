# Step 1 下载RASC安装程序
![](photo/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823761273597.png)
# Step 2 进入安装配置界面，点击next
![](photo/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823578862522.png)
# Step 3 选择RA和R-Car型号
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823581912579.png)
# Step 4 在 “Customise Features”项目，勾选 R-Car MCU Smart Configurator。
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823582511558.png)
# Step 5 安装附加软件
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823583419428.png)
# Step 6 同意协议
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823584031028.png)
# Step 7 下载结束后点击“OK”结束
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823584704247.png)
# Step 8 在这个目录下解压RCARMCU_FSP_Packs，并复制文件夹internal
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823774422929.png)
# Step 9 替换此目录底下的文件夹internal
![](%E9%99%84%E4%BB%B6/Pasted%20image%2020260625165411.png)
# Step 10 在VS Code中安装必备组件
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823662176433.png)
# Step 11 点击运行调试生成renesaselectronicscorporation.renesas-debug文件
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823678691748.png)
# Step 12 复制此目录底下的DebugComp文件
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823679142768.png)
# Step 13 复制到renesaselectronicscorporation.renesas-debug目录下
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823680906268.png)
# Step 14 在此目录下运行rasc.exe
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823682822991.png)
# Step 15 选择Renesas R-Car MCU U5x
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823702521566.png)
# Step 16 其他步骤全选默认，然后点击打开目录
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823708726444.png)
# Step 17 用VS Code打开该文件
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823708945888.png)
# Step 18 点击左上角“···”，新建终端并点击下方打开输出
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823709767882.png)
# Step 19 用VS Code打开此目录底下的launch，并复制
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823799407374.png)
# Step 20 替换掉工作区.vscode中的launch文件
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_1782380220802.png)
# Step 21 下载CMake，安装时确认勾选 Add CMake to the system PATH
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823803907408%201.png)
# Step 22 下载ninja，并将下载目录添加到 Windows 用户环境变量 Path
![697](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823805978488.png)
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823810944230.png)
# Step 23 安装 ARM GCC 编译器
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17823813918635.png)
# Step 24 新建用户变量并重启VS Code
![](%E9%99%84%E4%BB%B6/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17824364941392.png)
# Step 25 点击底部Kit，选择上方ARM GCC - Ninja
![](%E9%99%84%E4%BB%B6/Pasted%20image%2020260626092418.png)
# Step 26 进入左侧栏 CMake，点击如图所示图标对项目进行调试配置
![](%E9%99%84%E4%BB%B6/Pasted%20image%2020260626092757.png)
# Step 27 在命令面板中重新加载Window VS Code，然后等待CMake重新配置完成
![](%E9%99%84%E4%BB%B6/Pasted%20image%2020260626093245.png)
# Step 28 在src路径下新建文件或宏
![](%E9%99%84%E4%BB%B6/Pasted%20image%2020260626093650.png)
# Step 29 在左侧栏运行和调试中选择Jlink或E2
![](%E9%99%84%E4%BB%B6/Pasted%20image%2020260626094410.png)
