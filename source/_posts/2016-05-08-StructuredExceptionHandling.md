---
title: Windows中的结构化异常处理
date: 2016-05-08 05:33:42
categories:
- Error Handling
tags:
- Windows
- Exception
- MiniDump
- SEH
---
### 背景
上一次在 Windows 上用 C/C++ 做正经项目、需要做严格异常处理和调试的场合，已经是 5 年前了，那时我还在维护 PC 端游戏引擎。
<!-- more -->
在那以后，经历了移动开发到现在的 C# 开发，都不需要再跟 Windows 的结构化异常处理 ([SEH](https://zh.wikipedia.org/zh/%E7%BB%93%E6%9E%84%E5%8C%96%E5%BC%82%E5%B8%B8%E5%A4%84%E7%90%86)) 打交道。
虽说用不上，但看技术文档总有可能碰上，正好最近就有这样的场合，显然我已经把 [《Windows核心编程》](https://book.douban.com/subject/3235659/) 里的细节给忘光光了，只能重新学习下。


### SEH的概念
把 MSDN 上的[文档](https://msdn.microsoft.com/en-us/library/windows/desktop/ms680657%28v=vs.85%29.aspx) 读了过后，发现最有价值的，还是[这里](https://msdn.microsoft.com/en-us/library/windows/desktop/ms679270%28v=vs.85%29.aspx)。

首先，Windows 的 SEH 整个脉络是这样的：

+ Exception Triggering
    - Guarded Body
    - Raising Exception
+ Exception Handling
    - [Frame-based Exception Handling](https://msdn.microsoft.com/en-us/library/windows/desktop/ms679353%28v=vs.85%29.aspx)
    -  [Vectored Exception Handling](https://msdn.microsoft.com/en-us/library/windows/desktop/ms679270%28v=vs.85%29.aspx)
    -  [Termination Handling](https://msdn.microsoft.com/en-us/library/windows/desktop/ms681394%28v=vs.85%29.aspx)

分为异常触发和异常处理，而在编程层面讲，上面的功能需要藉由以下关键字或者函数来实施：(和上面一一对应)

+ Exception Triggering
    - [__try](https://msdn.microsoft.com/en-us/library/windows/desktop/ms679270%28v=vs.85%29.aspx) block
    - [RaiseException](https://msdn.microsoft.com/en-us/library/windows/desktop/ms680552%28v=vs.85%29.aspx) function, 以及硬件异常(CPU 触发)
+ Exception Handling
    - [__except](https://msdn.microsoft.com/en-us/library/windows/desktop/ms679270%28v=vs.85%29.aspx) block, [GetExceptionCode](https://msdn.microsoft.com/en-us/library/windows/desktop/ms679356%28v=vs.85%29.aspx),  [GetExceptionInformation](https://msdn.microsoft.com/en-us/library/windows/desktop/ms679357%28v=vs.85%29.aspx)
    - [AddVectoredContinueHandler](https://msdn.microsoft.com/en-us/library/windows/desktop/ms679273%28v=vs.85%29.aspx), [RemoveVectoredContinueHandler](https://msdn.microsoft.com/en-us/library/windows/desktop/ms680567%28v=vs.85%29.aspx), [AddVectoredExceptionHandler](https://msdn.microsoft.com/en-us/library/windows/desktop/ms679274%28v=vs.85%29.aspx), [RemoveVectoredExceptionHandler](https://msdn.microsoft.com/en-us/library/windows/desktop/ms680571%28v=vs.85%29.aspx) 
    - [__finally](https://msdn.microsoft.com/en-us/library/windows/desktop/ms679270%28v=vs.85%29.aspx) block, [AbnormalTermination](https://msdn.microsoft.com/en-us/library/windows/desktop/ms679265%28v=vs.85%29.aspx)


### SEH原理
假设给定一个 Call stack，每一个 Callsite ，都位于 Caller 的 __try block  中。那么，当最内层的一个函数调用 RaiseException 来抛出一个软件异常、或者在除零/非法内存访问抛出硬件异常的情形下，会发生这些事：

1. 当执行流进入一个函数的 \_\_try block 时，会把 \_\_except 的filtering expression (也就是 \_\_except 的条件，在支持异常处理的高级语言中，对应 catch(e) 中的e )、handling block、以及 \_\_finally block 这三个程序块儿的地址 push 到一个 thread local 的链表头上，从而将所有栈上 frame 的 exception handling 逻辑块串成一个链表
2.  当 Call stack 的栈顶上触发一次异常时，无论是软件异常还是硬件异常，OS 先准备异常码(GetExceptionCode 的返回值)和调用上下文 (包括当前线程的各种寄存器，即 GetExceptionInformation 的返回值)，然后以他们为参数 (除非用户在filter中完全没有访问这些信息)，逐个调用 thread local 的异常处理链中的 filter，检测每个 filter 的返回值，决定下一步行为。此时，filter 可以返回三种结果，来影响异常处理流程：
    + [EXCEPTION_CONTINUE_EXECUTION](https://msdn.microsoft.com/en-us/library/s58ftw19.aspx): 线程继续从异常触发点往下执行，即忽略异常。一般在返回该值之前 filter 需要调整 Eip/Rip 值，否则同样的异常会再次触发。如果一个 \_\_except 的 filter 返回该值表示要继续执行，那么，通过 AddVectoredContinueHandler 注册的 handler 会被调用。
    +  EXCEPTION\_CONTINUE\_SEARCH: 沿异常处理链继续尝试下个 filter，即尝试调用栈上更外层函数的 \_\_except 是否匹配。如果所有的用户定义的 \_\_except 的 filter 块都匹配失败，则调用用户通过 [SetUnhandledExceptionFilter](https://msdn.microsoft.com/en-us/library/windows/desktop/ms680634%28v=vs.85%29.aspx) 注册的顶层filter，一般用户可以在这个 filter 里面写 minidump 文件了。
    +  EXCEPTION\_EXECUTE\_HANDLER: 执行该 filter 对应的 handling block，即认为该 filter 匹配上了。由于匹配的 filter 所对应的 handler 很可能不在 Call stack 的栈顶，那么需要进行 Stack unwinding。在Stack unwinding 之前，如果用户通过 AddVectoredExceptionHandler 注册了 handler ，此时也会被调用；之所以要强调在栈开解之前调用 handler，是因为 handler 需要访问异常触发现场完整的栈。另外，在栈开解的同时，还要执行被弹出的 Stack frame 的相应 \_\_finally 块的 handler。最后，执行完对应 \_\_except 的 handler 块儿过后，线程继续从 \_\_except 之后往下执行。
    
恩，上面的过程很复杂，我们再简单的描述下：

1. \_\_try 使得 \_\_except, \_\_finally 的过滤条件和处理逻辑被 push 到当前线程的异常处理链上
2. 触发异常时，以异常信息为参数，沿着异常处理链逐个调用 filter 决定处理流程。filter 可以说“忽略该异常”，那么，线程在调用 vectored continue handler 过后继续从异常触发的指令往下执行；如果 filter 说“继续搜索”，那么，我们沿异常处理链访问下个 filter，如果最终都没有匹配的 filter，会调用通过 SetUnhandledExceptionFilter 注册的顶层 handler；如果 filter 说 “匹配”，则准备执行该 filter 对应的 handler (即 \_\_except 的 block)，于是，先调用 vectored exception handler，再进行 Stack unwinding，弹出中间栈帧的时候执行相应的 \_\_finally 块儿，最后，执行匹配位置的 handler。

### 示例

1. 第一个例子，filter 说“忽略”，所以，vectored continue handler 被调用，整个程序会输出 123 。注意返回 EXCEPTION_CONTINUE_EXECUTION 的 filter 需要调整 Eip。

    {% codeblock lang:c %}
    #include <windows.h>

    static LONG NTAPI MyFramedFilter(_EXCEPTION_POINTERS *ExceptionInfo) 
    {
        // skip the illegal op:  *reinterpret_cast<int*>(nullptr) = 0;
        reinterpret_cast<PCONTEXT>(ExceptionInfo->ContextRecord)->Eip += 9;

        puts("1");
        return EXCEPTION_CONTINUE_EXECUTION;
    }

    static LONG NTAPI MyVectoredContinueHandler(_EXCEPTION_POINTERS *ExceptionInfo) 
    {
        puts("2");
        return EXCEPTION_CONTINUE_EXECUTION;
    }

    static void Func2() 
    {
        *reinterpret_cast<int*>(nullptr) = 0;
    }

    static void Func1() 
    {
        __try 
        {
            Func2();
        } 
        __except (MyFramedFilter(GetExceptionInformation())) 
        {
            puts("never");
        }
        puts("3");
    }

    int main() 
    {
        AddVectoredContinueHandler(1, MyVectoredContinueHandler);
        Func1();
    }
    {% endcodeblock %}

2. 第二个例子，filter 说“匹配”，所以先执行 filter，在栈开解的时候执行 \_\_finally block ，最后是 \_\_except 的 handler 和之后的代码。程序输出 1234 。

    {% codeblock lang:c %}
    static LONG NTAPI MyFramedFilter(_EXCEPTION_POINTERS *ExceptionInfo) 
    {
        puts("1");
        return EXCEPTION_EXECUTE_HANDLER;
    }

    static void Func2() 
    {
        __try 
        {
            *reinterpret_cast<int*>(nullptr) = 0;
        }
        __finally 
        {
            puts("2");
        }
    }

    static void Func1() 
    {
        __try 
        {
            Func2();
        } 
        __except (MyFramedFilter(GetExceptionInformation())) 
        {
            puts("3");
        }
        puts("4");
    }

    int main() 
    {
        Func1();
    }
    {% endcodeblock %}

3. 第三个例子，filter 说“继续搜索”，于是顶层的 filter 被调用，这里顶层 filter 返回匹配，所以栈开解，调用内层的 __finally handler。程序输出 123 。

    {% codeblock lang:c %}
    static LONG NTAPI MyFramedFilter(_EXCEPTION_POINTERS *ExceptionInfo) 
    {
        puts("1");
        return EXCEPTION_CONTINUE_SEARCH;
    }

    static LONG NTAPI MyTopLevelFilter(_EXCEPTION_POINTERS *ExceptionInfo)
    {
        puts("2");
        return EXCEPTION_EXECUTE_HANDLER;
    }

    static void Func2() 
    {
        __try 
        {
            *reinterpret_cast<int*>(nullptr) = 0;
        }
        __finally 
        {
            puts("3");
        }
    }

    static void Func1() 
    {
        __try 
        {
            Func2();
        } 
        __except (MyFramedFilter(GetExceptionInformation())) 
        {
            puts("never");
        }
        puts("never");
    }

    int main() 
    {
        SetUnhandledExceptionFilter(MyTopLevelFilter);
        Func1();
    }
    {% endcodeblock %}


### 对比高级语言的异常
相比高级语言中的 try/catch/finally ， Windows 的 SEH 多了这些优点：

+ 能够捕获非法内存访问等硬件异常 (EXCEPTION_INT_DIVIDE_BY_ZERO, EXCEPTION_INT_OVERFLOW, EXCEPTION_ACCESS_VIOLATION)。
+ 可以从异常触发点继续执行 (EXCEPTION_CONTINUE_EXECUTION)。
+ 可以在判断 \_\_try block 有没有被完全执行 (AbnormalTermination)。
+ filter expression 比高级语言中的异常类型匹配更强大。
+ 捕获的异常信息可以用于创建 MiniDump 等 (_EXCEPTION_POINTERS)。

以下是缺点：

+ C++ 的异常处理可能不是通过 SEH 实现，于是结构化异常抛出时，类析构函数不会被调用。
+ 特定于 Windows 平台。


### 结语
这是一篇即使放到10年前都不算新的文章，写它，更算是把我学习的结果记下来，以及刻意的丰富下博客内容吧。

就这篇文章讨论的这个话题而言，还是再推荐下 [Windows核心编程](https://book.douban.com/subject/3235659/) 。尽管这次我没看这本书，但 Jeffrey 的书总是深入浅出、鞭辟入里。我认为进入一个新领域时，如果正好有他的书，那一定是从入门到进阶的好机会。比如，学习 .Net 就推荐他的 [CLR via C#](https://book.douban.com/subject/4112979/)。