---
title: MiniDump的创建和使用
date: 2016-05-08 20:16:10
categories:
- Error Handling
tags:
- Windows
- Exception
- MiniDump
- SEH
---
前面两篇文章([1](http://blog.wbscan.com/2016/05/08/StructuredExceptionHandling/), [2](http://blog.wbscan.com/2016/05/08/CreateMiniDumpWithWindowsApi/))从编程角度讲了Windows异常处理机制和用API创建MinDump文件，这里再来讲一些其他创建MiniDump的方式，以及MiniDump文件的基本使用。
<!-- more -->
***
### 创建MiniDump文件
由于不同事故现场有自己的限制，可能用户程序根本没有主动生成MiniDump文件，或者程序在运行、但你想要去诊断它的状态。

这篇[文章](http://www.wintellect.com/devcenter/jrobbins/how-to-capture-a-minidump-let-me-count-the-ways)列举了各种MiniDump生成方法，我认为很有价值：

+ [WinDBG](http://www.windbg.org/)的[.dump](https://msdn.microsoft.com/en-us/library/windows/hardware/ff562428%28v=vs.85%29.aspx)命令
+ Visual Studio的`将转储另存为`菜单项：调试状态下，`调试 -> 将转储另存为`。可以选择创建Mini Dump或者Full Dump。
+ 任务管理器：`选中进程 -> 创建转储文件`，这里创建的是Full Dump。
+ [Process Explorer](https://technet.microsoft.com/en-us/sysinternals/bb896653.aspx): `选中进程 -> 创建Mini Dump/Full Dump`
+ [ProcDump](https://technet.microsoft.com/en-us/sysinternals/dd996900.aspx): ProcDump可以基于各种条件来创建Mini Dump或者Full Dump，比如：
    - procdump notepad: 为notepad创建Mini Dump
    - procdump -ma 4572: 为pid为4572的进程创建Full Dump
    - procdump -mp -e store.exe: 当store.exe遇到第一个未处理异常的时候，为它创建一个比Mini Dump信息稍多的Dump文件
    - procdump -c 20 -s 5 -n 3 consume: 监视consume进程，一旦它连续5秒的CPU使用率都超过20%，则创建MiniDump，总共最多Dump 3次。之后，你可以用这不同的Dump来做对比等。
+ 通过Windows API [MiniDumpWriteDump](https://msdn.microsoft.com/en-us/library/ms680360.aspx) 创建转储文件，能够在代码中精确的控制需要转储的进程信息。这是我们上篇文章做的。

### 使用MiniDump文件
+ 使用Visual Studio，`文件 -> 打开 -> 文件`选中刚才创建的MiniDump文件即可。这个MiniDump文件，如果是从Managed进程生成的，那么用法会更丰富。
    - Native程序生成的MiniDump: 一般能够查看调用栈，如果是Full Dump的话，往往还可以查看堆数据。注意，如果MiniDump不是基于异常生成的，而是直接创建的，那么可以通过`调试 -> 窗口 -> 线程`来查看所有线程的调用栈。
    - Manged程序生成的MiniDump: 线程状态的诊断和Native程序差不多，但打开Managed程序的Dump文件时，多了一个菜单项是`调试托管内存`，这个功能很强大：
        + 查找托管堆中指定类型的所有对象 
        + 查找特定对象所包含的引用
        + 查找从GC根集到特定对象的所有引用路径。这是非常强大的特性，在资源泄露的排查中非常有帮助。比如，你发现程序的内存一直上涨，并且特定类型对象特别多，那么，利用Visual Studio的这个功能，选中异常类型的一个实例，就能看到它为什么还被引用住，而没有释放了（比如，被添加到一个static的字典里面，由于逻辑错误，没有被正常移除）。
+ 使用WinDBG或者[cdb](https://msdn.microsoft.com/en-us/library/windows/hardware/hh406277%28v=vs.85%29.aspx)。这些工具尺寸小，往往可以直接用于在生产服上的本机调试，但缺乏对Managed程序的特殊调试支持。另外，相比Visual Studio，上手曲线更大。作为初步诊断，可以直接在目标机上，通过任务管理器创建MiniDump，然后运行`cdb -c "~*kv; q" -y SymbolPath -z DumpFile`命令，这将打印所有线程的Stack trace，能用于处理一些简单问题。
***
这里讲了各种MiniDump文件的创建方式和基本使用，对常见问题诊断很有帮助。
