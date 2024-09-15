


---


　　大家好，我是痞子衡，是正经搞技术的痞子。今天痞子衡给大家分享的是**JLink命令行以及JFlash对于下载算法的作用地址范围认定**。


　　最近痞子衡在给一个 RT1170 客户定制一个 Infineon MirrorBit 类型 64MB Flash 的 SEGGER 下载算法，做完之后在 JFlash 下测试小数据下载没有问题，但是大数据下载就报了地址范围不适用的错误，所以今天我们就来深挖一下自制下载算法时作用地址设定问题：



> * Note: 本文所测试 JLink 版本为 V7\.94f


### 一、地址范围设定


　　关于 SEGGER 下载算法制作，痞子衡之前写过两篇文章：[《串行NOR Flash下载算法(MDK工具篇)](https://github.com) 一文讲得是如何制作 FLM 算法文件（MDK 算法与 SEGGER 算法是通用的），[《串行NOR Flash下载算法(J\-Link工具篇)》](https://github.com) 一文讲得是配套 XML 文件写法。


　　XML 文件里的 BaseAddr 和 MaxSize 参数设定的地址范围主要用于选定适用的 FLM 算法文件（即 Loader），而生成 FLM 算法文件源工程里的 FlashDev.c 文件里的 FLASH\_BASE\_ADDRESS 和 FLASH\_BASE\_SIZE 参数则是算法在运行过程中用于判断的有效下载数据地址范围。



> * Note：关于 XML 添加方法详见痞子衡旧文 [《从JLink V7\.62开始优化了手动增加新MCU型号支持方法》](https://github.com)


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/JLink_Algo_AddrRegion_TwoFiles.PNG)


### 二、测试地址范围


　　有了以上理论基础，现在我们测试一下地址范围设定对下载的影响。我们基于恩智浦 MIMXRT1170\-EVKB 评估板，选用一颗 64MB NOR Flash 连在 FlexSPI1 外设上（AHB 映射起始地址为 0x3000\_0000，FLM 下载算法里 FLASH\_BASE\_ADDRESS 固定设为 0x3000\_0000）。


#### 2\.1 JLink命令行下测试


　　先在 JLink 命令行下用 LoadFile 命令做测试，该命令支持所有主流格式的程序文件。为了方便设定下载起始地址，我们就用 .bin 格式做测试。



```
命令格式 LoadFile , [ (.bin only)].
命令解释 Load data file into target memory. Supported ext.: *.bin, *.mot, *.hex, *.srec, *.elf, *.out, *.axf

```

　　如果 XML, FLM, LoadFile 地址范围都设定无误，命令执行时后台会弹出下载进度条窗口，表明 FLM 算法被成功调用且在正常擦写 Flash。


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/JLink_Algo_AddrRegion_CmdSuccess.PNG)


　　现在我们尝试设定不同地址范围（下表里设定的非 0x3000\_0000 \- 0x33FF\_FFFF 有效 64MB Flash 空间范围之外的测试地址需要是真正的无效存储空间地址，不能是 MCU 片内的 SRAM 映射地址），做更多测试，结果如下：




| XML范围设定 | FLM范围设定 | LoadFile地址 | 测试结果 |
| --- | --- | --- | --- |
| 0x3000\_0000 \- 0x33FF\_FFFF | | 设定范围内 | 正常下载 |
| 设定范围外 | Writing target memory failed. |
| 0x4000\_0000 \- 0x43FF\_FFFF | 0x3000\_0000 \- 0x33FF\_FFFF | XML范围内 | Writing target memory failed. |
| FLM范围内 | Writing target memory failed. |
| 0x3000\_0000 \- 0x37FF\_FFFF | 0x3000\_0000 \- 0x33FF\_FFFF | XML且FLM范围内 | 正常下载 |
| XML范围内但FLM范围外 | Writing target memory failed. |
| 0x3000\_0000 \- 0x33FF\_FFFF | 0x3000\_0000 \- 0x31FF\_FFFF | XML且FLM范围内 | 正常下载 |
| XML范围内但FLM范围外 | Writing target memory failed. |
| 0x3000\_0000 \- 0x31FF\_FFFF | 0x3000\_0000 \- 0x33FF\_FFFF | XML且FLM范围内 | 正常下载 |
| XML范围外但FLM范围内 | Writing target memory failed. |
| 0x3000\_0000 \- 0x37FF\_FFFF | | 0x3000\_0000 \- 0x33FF\_FFFF内 | 正常下载 |
| 0x3400\_0000 \- 0x37FF\_FFFF内 | 实际下载到Addr\-0x4000000处 |


　　上述测试结果表明，仅当程序下载地址在 XML 和 FLM 共同指向的范围内，且属于有效的 Flash 空间时，下载才正常进行。此外，表格最后一项测试表明，即使超出实际连接的 Flash 最大空间，下载也没有报错，这是因为 MCU 发送给 Flash 操作命令地址溢出了，地址溢出部分被 Flash 自动忽略了。



> * Note：要实现表格最后一项测试效果，在制作 FLM 下载算法时，配置 MCU 存储接口外设（对于 i.MXRT1170 来说是 FlexSPI）的 AHB 空间必须与 FlashDev.c 里设定一致，且这个空间不超过芯片系统分配给外设的最大 AHB 空间。


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/JLink_Algo_AddrRegion_loadFileTest.PNG)


#### 2\.2 JFlash下测试


　　再在 JFlash 界面下做测试，打开软件，创建工程时 Target Device 需要设定为 XML 文件 ChipInfo 中 Name，这样可指定使用自制 FLM 文件。这里也可以看到界面里 Flash banks 自动就识别到了 XML 所设定的地址范围。



> * Note1：JFlash 认定的起始地址一定是 XML 中 BaseAddr。
> * Note2：当 XML 中 BaseAddr 与 FLM 中 FLASH\_BASE\_ADDRESS 一致时，JFlash 认定的空间长度由 XML 中 MaxSize 和 FLM 中 FLASH\_BASE\_SIZE 共同决定，两者取其小。
> * Note3：当 XML 中 BaseAddr 与 FLM 中 FLASH\_BASE\_ADDRESS 不一致时，JFlash 认定的空间长度由 XML 中 MaxSize 决定。


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/JLink_Algo_AddrRegion_GuiSetting.PNG)


　　JFlash 下测试结果本质上其实和 JLink 命令下行为一致，我们可以理解为 JFlash 底层调用得就是 JLink 命令实现，只不过界面里做了更多检查与附加功能。且上述 Note 表明 JFlash 在加载算法时对地址空间长度做了预处理，所以当程序下载地址超出 JFlash 认定范围时，JFlash 会弹框提示：


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/JLink_Algo_AddrRegion_GuiNote.PNG)


　　至此，JLink命令行以及JFlash对于下载算法的作用地址范围认定痞子衡便介绍完毕了，掌声在哪里\~\~\~


### 欢迎订阅


文章会同时发布到我的 [博客园主页](https://github.com):[wgetCloud机场](https://tabijibiyori.org)、[CSDN主页](https://github.com)、[知乎主页](https://github.com)、[微信公众号](https://github.com) 平台上。


微信搜索"**痞子衡嵌入式**"或者扫描下面二维码，就可以在手机上第一时间看了哦。


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/github/pzhMcu_qrcode_258x258.jpg)


