## 问题一：一直进入
![](picture/Pasted%20image%2020260708170701.png)

关闭MPU




**支持 ECC**

```
ECC support for error detection and correction
```

这和我们前面怀疑的点有关。读取未初始化、非法、空白或受保护区域时，有可能触发 ECC 相关异常。

对项目帮助：

```
有帮助。
```

后面必须查 ECC 章节，看读取 blank Code Flash / Data Flash 会不会触发错误。


![](picture/Pasted%20image%2020260709102716.png)



![](picture/Pasted%20image%2020260709103542.png)



![](picture/Pasted%20image%2020260709103836.png)

![](picture/Pasted%20image%2020260709104513.png)


![](picture/Pasted%20image%2020260709104921.png)


![](picture/Pasted%20image%2020260709104831.png)



![](picture/Pasted%20image%2020260709105531.png)


![](picture/Pasted%20image%2020260709105542.png)![](picture/Pasted%20image%2020260709110810.png)