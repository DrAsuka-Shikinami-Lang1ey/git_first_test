# U2X 环境安装与工程验证


## 一、环境与目录约定

- GHS Compiler：`C:\ghs\comp_202214`
- GHS MULTI：`C:\ghs\multi_716d`
- MCAL 根目录：`D:\MCAL`
- MCAL 工程目录：`D:\MCAL\Project`
- U2A8 示例工程：`D:\MCAL\Project\U2A8_Demo`
- U2A8 手动练习工程：`D:\MCAL\Project\U2A8_Manual_Practice`
- DaVinci/BSWMD：每种芯片或封装单独解压到 `D:\Vector` 下，避免相互覆盖。



## 二、安装 GHS Compiler 和 MULTI

### 1. 启动安装程序

进入 GHS 安装包根目录，双击 `ginstall.exe`。

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17851622269793.png)

### 2. 执行默认安装

选择 **Easy Install**，安装程序会同时安装 Compiler、Probe 支持和 MULTI IDE。

![](picture/Pasted%20image%2020260727222651.png)

保持默认安装路径：

```text
C:\ghs\comp_202214
C:\ghs\multi_716d
```

### 3. 结束基础安装

出现 `Installation complete` 后，点击 **Exit**。发布说明可以按需查看，不需要在此时启动 MULTI。

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17851651786714.png)

### 4. 安装 MULTI 7.1.6 补丁

进入安装介质中的 `multi-v7.1.6-patch` 目录，运行 `install.bat`。

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17851667709709.png)

根据提示输入 MULTI 安装目录：

```text
C:\ghs\multi_716d
```

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17851668511820.png)

脚本正常结束且未显示错误，即表示补丁安装完成。

## 三、部署 AUTOSAR/MCAL 环境

### 1. 运行资料包根目录中的配置脚本

回到 `RH850-U2A Demo板资料包` 根目录，运行 `TeampoU2AScript.exe`。该脚本位于资料包根目录，不在 GHS 安装目录中。

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17851673755863.png)

### 2. 选择“安装环境”

在命令行菜单中输入 `2`，开始安装 AUTOSAR/MCAL 环境。

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17851674911393.png)

安装目录使用：

```text
D:\MCAL
```

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17851677658467.png)

当所有解压进度均达到 `100%`，并显示“环境已安装到 `D:\MCAL`”时，按任意键退出。

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17851678656377.png)

安装完成后，`D:\MCAL` 下应包含 MCAL、Project、Vector、Demo 板资料包及配置脚本。

## 四、使用脚本创建 U2A8 Demo 工程

### 1. 启动 MCAL 根目录脚本

运行：

```text
D:\MCAL\TeampoU2AScript.exe
```

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17851680344021.png)

### 2. 新建工程

选择 `2. 新建工程`。

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_1785168105436.png)

输入工程名，例如：

```text
U2A8_Demo
```

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17851682091440.png)

依次选择芯片类型和封装。U2A8 292-pin/FABG 的选择为：

```text
芯片类型：2（U2A8）
芯片型号：5（EABG/FABG，292-pin）
```

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17851683334918.png)

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17851683744166.png)

脚本提示工程创建完成后退出。

![](picture/Pasted%20image%2020260728000710.png)

## 五、打开 DaVinci 工程并生成 MCAL 代码

### 1. 从工程目录启动脚本

运行：

```text
D:\MCAL\Project\U2A8_Demo\TeampoU2AScript.exe
```

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17851684822334.png)

选择 `2. 选中工程`，脚本会启动与该工程匹配的 DaVinci Configurator。

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17851686394515.png)

首次启动时会显示安全提示，阅读后选择 **I accept**。

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_1785168751104.png)

### 2. 打开 `u2a.dpa`

在 DaVinci 中选择 **File → Open**，打开：

```text
D:\MCAL\Project\U2A8_Demo\tools\Mcal\u2a.dpa
```

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_1785169048497.png)

如果提示 `Some paths in the DPA file are not valid`：

- 路径仅是首次运行尚未创建时，可选择 **Create Paths**；
- 已确认目录存在时，可点击 **OK** 继续；
- 若提示路径仍指向其他工程，应先关闭 DaVinci，修正 `u2a.dpa` 后重新打开。

![](picture/Pasted%20image%2020260728001826.png)

等待底部 `Initializing meta model...` 完成。首次打开时出现 Validation 或 Unresolved References 提示并不等于程序安装失败，应以实际代码生成结果为准。

![](picture/Pasted%20image%2020260728002027.png)

### 3. 验证并生成代码

点击工具栏中的验证和生成按钮，或选择 **Project → Generate**。

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_1785226592760.png)

在 Generate 窗口中保持所需 MCAL 模块勾选，点击 **Generate**。所有模块显示绿色对勾且没有 Error，即表示代码生成完成。

## 六、配置 GHS 网络许可证

### 1. 打开顶层工程

双击：

```text
D:\MCAL\Project\U2A8_Demo\default.gpj
```

`default.gpj` 是整个 GHS 工程的入口；`u2a_mcal.gpj` 是子工程，`debug.gpc` 和 `release.gpc` 是连接配置文件。

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17852265365107.png)

首次启动 MULTI 时会自动打开 Licensing Wizard。

### 2. 指定许可证服务器

选择 **License my MULTI products**，点击 **Next**。

![](picture/Pasted%20image%2020260728163856.png)

选择第四项 **Specify the network location of License Managers**，点击 **Next**。

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_1785228025136.png)


确认后，配置会写入 MULTI 和 Compiler 的 `license.cfg`。
许可证管理器列表出现服务器 `TPSZXGHSSVR`、端口 `2009`、模式 `Floating`，表示服务器已被发现。随后可关闭许可证管理窗口。

![](picture/Pasted%20image%2020260728164905.png)

> [!NOTE]
> `All licenses ... are in use` 表示浮动席位已被占满；`No GHS LM found` 表示当前网络无法连接许可证服务器。前者需要等待或联系管理员释放席位，后者需要检查公司内网/VPN及防火墙。

## 七、编译并验证 Demo 工程

在 MULTI Project Manager 中选择 **Build → Rebuild default.gpj**。

![](picture/Pasted%20image%2020260728172410.png)

编译结束后检查 Build Details：

- 显示 `Build successful`；
- Errors 为 `0`；
- 成功生成 `u2a_mcal.elf`。

Warning 可以后续逐项分析，但不能存在 Error。

![](picture/Pasted%20image%2020260728172637.png)

## 八、手动配置 U2A8 292-pin/FABG 练习工程

这一部分用于理解脚本背后的 DaVinci/BSWMD 和 MCAL 路径配置。自动脚本创建的 `U2A8_Demo` 已能正常使用时，再进行本节操作。

### 1. 解压匹配的 DaVinci/BSWMD

运行交付包中的 `CBD2100679_D00.exe`。

![](picture/Pasted%20image%2020260729103653.png)

为避免覆盖原环境，将目标目录设置为：

```text
D:\Vector\CBD2100679_D00_U2A8_292_V430300_Practice
```

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17852930146700.png)

点击 **Extract**。如果弹出加密文件密码窗口，输入交付方提供的解压密码；解压完成后窗口会自动关闭。

![](picture/Pasted%20image%2020260729105029.png)

解压后可在以下目录找到 DaVinci 主程序：

```text
D:\Vector\CBD2100679_D00_U2A8_292_V430300_Practice\DaVinciConfigurator\Core\DaVinciCFG.exe
```

![](picture/Pasted%20image%2020260729110020.png)

### 2. 将 MCAL 定义复制到 BSWMD

进入：

```text
D:\MCAL\AUTOSAR_RH850_U2Ax_MCAL_Ver43.03.00\scripts
```

运行 `CopyCfgFile.bat`。

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17853033932557.png)

输入本套 DaVinci 的 BSWMD 目录：

```text
D:\Vector\CBD2100679_D00_U2A8_292_V430300_Practice\Components\MSR_Vector_SLP4\BSWMD
```

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17853036103711.png)

随后选择：

```text
芯片类型：2（U2A8）
芯片型号：5（EABG/FABG，292-pin）
```

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17853039148248.png)

每个模块均显示“已复制 1 个文件”，最后显示 `Copy completed!`，表示定义文件复制成功。

### 3. 建立练习工程副本

复制：

```text
D:\MCAL\Project\template
```

并将副本重命名为：

```text
U2A8_Manual_Practice
```

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17853042123934.png)

### 4. 修改 `u2a.dpa`

用记事本打开：

```text
D:\MCAL\Project\U2A8_Manual_Practice\tools\Mcal\u2a.dpa
```

![](picture/Pasted%20image%2020260729140121.png)

将文件中原工程目录替换为 `U2A8_Manual_Practice`，并确认 DaVinci/BSWMD 路径指向本节解压的 `...U2A8_292_V430300_Practice` 目录。替换完成后保存。

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17853050048325.png)

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17853051034586.png)

### 5. 修改实际代码生成脚本

先复制 `CodeGenerator_New.bat`，将副本命名为：

```text
CodeGenerator_New_U2A8_Demo_backup.bat
```

该文件仅用于备份和恢复，**DaVinci 实际调用的仍然是 `CodeGenerator_New.bat`**。

![](picture/Pasted%20image%2020260729141456.png)

用记事本打开原始 `CodeGenerator_New.bat`，确认以下两项指向练习工程：

```bat
set GEN_SRC_ARXML=D:\MCAL\Project\U2A8_Manual_Practice\tools\Mcal\Config\ECUC\u2a.ecuc.arxml
set GEN_OUT_DIR=D:\MCAL\Project\U2A8_Manual_Practice\u2a_mcal\bsp\mcal\modules
```

保存文件。

![](picture/Pasted%20image%2020260729142155.png)

> [!WARNING]
> `CodeGenerator_New.bat` 是 MCAL 安装目录下的共享脚本，同一时间只能指向一个练习工程。切换工程前必须检查这两处路径，避免把生成代码写入错误目录。

### 6. 打开工程并生成代码

启动与 292-pin/FABG 工程匹配的 DaVinci：

```text
D:\Vector\CBD2100679_D00_U2A8_292_V430300_Practice\DaVinciConfigurator\Core\DaVinciCFG.exe
```

![](picture/Pasted%20image%2020260729142854.png)

在 DaVinci 中选择 **File → Open**，打开：

```text
D:\MCAL\Project\U2A8_Manual_Practice\tools\Mcal\u2a.dpa
```

![](picture/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17853068648319.png)

然后选择 **Project → Generate**。所有模块生成成功后，再打开练习工程根目录中的 `default.gpj`，执行 **Rebuild default.gpj**，确认 `0 Errors` 并生成 `.elf` 文件。

## 九、安装其他芯片/封装的 DaVinci/BSWMD

不同芯片或封装应分别解压一套 DaVinci/BSWMD，并使用独立目录，不能覆盖 U2A8 292-pin 环境。

### 1. U2A8 373-pin

将 `U2A8_373_V430300.exe` 解压到独立目录，例如：

```text
D:\Vector\CBD2100679_D00_U2A8_373_V430300
```

![](picture/Pasted%20image%2020260730094240.png)

运行 `CopyCfgFile.bat` 时选择：

```text
芯片类型：2（U2A8）
芯片型号：4（EBBA，373-pin）
```

![](picture/Pasted%20image%2020260730094923.png)

其余步骤与 U2A8 292-pin 相同，但工程中的 `u2a.dpa`、代码生成路径和芯片配置必须指向 373-pin 对应目录。

### 2. U2A16 516-pin

将 `U2A16_516_V430300.exe` 解压到独立目录，例如：

```text
D:\Vector\CBD2100679_D00_U2A16_516_V430300
```

运行 `CopyCfgFile.bat` 时选择：

```text
芯片类型：3（U2A16）
芯片型号：3（516-pin）
```

当前脚本默认使用 `EBBG` 搜索定义文件，但这套 MCAL 43.03.00 中对应的实际文件标识为 `EBBC`。不要直接破坏公共脚本，建议先复制为：

```text
CopyCfgFile_U2A16_516_fix.bat
```

再在副本中将：

```bat
set "ChipID=EBBG"
```

改为：

```bat
set "ChipID=EBBC"
```

![](picture/Pasted%20image%2020260730102338.png)

重新运行修正后的脚本；每个模块显示“已复制 1 个文件”并最终出现 `Copy completed!`，才表示复制成功。若显示“已复制 0 个文件”，说明芯片标识或目录仍不匹配。

## 十、简要总结

U2X 环境搭建的核心顺序是：**安装并打补丁 GHS → 部署 MCAL → 安装匹配的 DaVinci/BSWMD → 创建并配置工程 → 生成 MCAL 代码 → 使用 MULTI 编译验证**。切换芯片或工程时，重点检查 `u2a.dpa`、`CodeGenerator_New.bat`、BSWMD 路径和芯片封装标识，最终以代码生成无 Error、MULTI 编译得到 `.elf` 为完成标准。
