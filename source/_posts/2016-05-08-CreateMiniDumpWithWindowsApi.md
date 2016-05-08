---
title: 用MiniDumpWriteDump来创建MiniDump
date: 2016-05-08 18:59:44
categories:
- Error Handling
tags:
- Windows
- Exception
- MiniDump
- SEH
---
上一篇[文章](http://blog.wbscan.com/2016/05/08/StructuredExceptionHandling/)讨论了SEH，这里就基于它做点事儿。
<!-- more -->
***
任务很简单，就是在发现一个未处理异常时，使用[MiniDumpWriteDump](https://msdn.microsoft.com/en-us/library/windows/desktop/ms680360%28v=vs.85%29.aspx)创建一个Full Dump文件，从而允许之后的异常调试：

{% codeblock lang:c %}
#include <windows.h>
#include <dbghelp.h>
#pragma comment (lib, "dbghelp.lib")

LONG WINAPI MyUnhandledExceptionFilter(EXCEPTION_POINTERS *exceptionPointers)
{
    HANDLE file = CreateFile(
        L"FullDump.dmp", 
        GENERIC_READ | GENERIC_WRITE,
        0, 
        nullptr, 
        CREATE_ALWAYS, 
        FILE_ATTRIBUTE_NORMAL, 
        nullptr);

    MINIDUMP_EXCEPTION_INFORMATION miniDumpExceptionInfo = 
    { 
        GetCurrentThreadId(), 
        exceptionPointers, 
        FALSE
    };

    MINIDUMP_TYPE mdt = static_cast<MINIDUMP_TYPE>(
        MiniDumpWithFullMemory |
        MiniDumpWithFullMemoryInfo |
        MiniDumpWithHandleData |
        MiniDumpWithThreadInfo |
        MiniDumpWithUnloadedModules);

    MiniDumpWriteDump(
        GetCurrentProcess(),
        GetCurrentProcessId(),
        file, mdt,
        (exceptionPointers != nullptr) ? &miniDumpExceptionInfo : 0,
        nullptr,
        nullptr);

    CloseHandle(file);

    return EXCEPTION_EXECUTE_HANDLER;
}

int main()
{
    SetUnhandledExceptionFilter(MyUnhandledExceptionFilter);

    *static_cast<int*>(nullptr) = 1;

    return 0;
}
{% endcodeblock %}

从事故现场拿到这个"FullDump.dmp"文件后，挂到Visual Studio或者WinDbg中，就可以调试现场了，包括调用栈和全局变量、堆数据都可以访问。

上面的代码是在发现未处理异常的时候写MiniDump文件的，此时的选择只剩下结束进程。其实也可以选择在任何一个__except的filter表达式中创建MiniDump，然后在其对应的handler中访问这个文件，并且选择不退出、继续执行。

如果我们创建的MiniDump，是用来诊断异常现场的，那么，MiniDump所需要的MINIDUMP_EXCEPTION_INFORMATION，只能由SEH filter提供，即在__except的filter expression中调用GetExceptionInformation或者通过SetUnhandledExceptionFilter注册成filter之后接受传入的Callback参数。

如果你不需要异常现场，只是想诊断为什么进程的内存占用很大，或者检查线程的非正常状态，那么，也可以将这个异常信息指针置为空，来简单的创建进程状态的Snapshot：

{% blockquote MSDN https://msdn.microsoft.com/en-us/library/windows/desktop/ms680360(v=vs.85).aspx MiniDumpWriteDump function %}
ExceptionParam [in]
A pointer to a MINIDUMP_EXCEPTION_INFORMATION structure describing the client exception that caused the minidump to be generated. If the value of this parameter is NULL, no exception information is included in the minidump file.
{% endblockquote %}


上面的代码创建的是一个Full Dump，如果目标进程本身的[Working Set](https://msdn.microsoft.com/en-us/library/windows/desktop/cc441804%28v=vs.85%29.aspx)很大，那么Dump的时间和输出文件都会很大，影响服务器响应或者不容易分发。因此，你可以针对不同的目的，裁剪一些进程信息，创建更小的Dump文件。比如，一个只包括调用栈的Mini Dump可能已经足够你解决问题了。

这里([1](http://www.debuginfo.com/articles/effminidumps.html), [2](http://www.debuginfo.com/articles/effminidumps2.html))是一些不错的讲这个主题的文章，然后[这里](http://www.debuginfo.com/examples/effmdmpexamples.html)也给出了创建能用于不同场合的各种大小的Dump文件的例子：

+ [TinyDump](http://www.debuginfo.com/examples/src/effminidumps/TinyDump.cpp): shows how to create the smallest possible minidump (more information
+ [MiniDump](http://www.debuginfo.com/examples/src/effminidumps/MiniDump.cpp): shows how to create a small but useful minidump (more information
+ [MidiDump](http://www.debuginfo.com/examples/src/effminidumps/MidiDump.cpp): shows how to create a relatively large but very informative minidump
+ [MaxiDump](http://www.debuginfo.com/examples/src/effminidumps/MaxiDump.cpp): shows how to create a minidump that contains all possible kinds of information


最后，MiniDumpWriteDump 函数其实也可以用来为其他进程创建Mini Dump，只不过要求一些内存、线程访问权限：

{% blockquote MSDN https://msdn.microsoft.com/en-us/library/windows/desktop/ms680360(v=vs.85).aspx MiniDumpWriteDump function %}
hProcess [in]
A handle to the process for which the information is to be generated.
This handle must have PROCESS_QUERY_INFORMATION and PROCESS_VM_READ access to the process. If handle information is to be collected then PROCESS_DUP_HANDLE access is also required. For more information, see Process Security and Access Rights. The caller must also be able to get THREAD_ALL_ACCESS access to the threads in the process. For more information, see Thread Security and Access Rights.
{% endblockquote %}
***
这个话题就讨论到这里，我们可以在我们的C++/.Net程序crash之前，搜集到足够的诊断信息，方便我们的调试了。