title: Intel Pin 5 ：Pin 高级调试扩展命令
date: 2016-01-12 15:46:41
tags: Pin
---
<font color="blue">写在前面：Advanced Debugger Extensions，我翻译为 “高级调试扩展命令”，如果大家对此有什么异议，欢迎指正。</font>

Pin的高级调试扩展命令允许程序员调试应用程序，即使程序在Pin下运行。
<!-- more -->
Pin tool可以在不修改 GDB 或 Visual Studio的情况下，添加对 新调试命令 的支持，这允许程序员通过 live debugger seesion 控制 Pin tool。最后 Pin tools 可通过 instrumentation 增加强大的调试功能。

如，Pin tools 可使用 instrumentation 查找感兴趣的条件（如，内存缓冲区覆盖），在 live debugger session 中满足此条件时，停止调试。

Linux（使用 GDB ）和 Windows（使用 Visual Studio ）都支持此功能，并且两个平台的 API 相同，但每个调试器的 UI 不同，使用方法不相同。
原教程主要分两部分：
 - Linux 平台教程
 - Windows平台教程

两个教程都使用了同一个例子，读者只用阅读其中之一即可，本文只介绍了Linux平台下的使用方法。

<font color="red">注意：高级调试扩展命令和调试 Pintool 一点关系都没有，如果 Pintool 存在 bug，需要调试，请参见[Tips for Debugging a Pintool](https://software.intel.com/sites/landingpage/pintool/docs/67254/Pin/html/index.html#DEBUGGING)</font> 

# Advanced Debugging Extensions on Linux
## 1、基本使用方法
Linux 下几乎所有 GDB 版本都支持 Pin 调试扩展命令，但是 Pin 需使用 GDB 的 远程调试功能，因此你可以使用任何支持此功能的 GDB 版本。

本节使用 "source/tools/ManualExamples" 目录下的 "stack-debugger.cpp" 例子说明 Pin 调试扩展命令。

使用如下命令：
```
$ cd source/tools/ManualExamples
$ make DEBUG=1 stack-debugger.test
```

tool 及其相关测试程序  "fibonacci" 在 "obj-ia32"（x86）/ "obj-intel64"(x64) 目录中。

Pin 命令添加 <font color="blue">-appdebug</font> 选项，开启调试扩展。
执行命令后， Pin 会启动应用程序并停在第一条指令前，然后输出“开启GDB”提示信息，。

```
$ ../../../pin -appdebug -t obj-intel64/stack-debugger.so -- obj-intel64/fibonacci 1000
Application stopped until continued from debugger.
Start GDB, then issue this command at the (gdb) prompt:
  target remote :33030
```

在另一个终端中开启 GDB，输入 提示信息中的命令。
```
$ gdb fibonacci
(gdb) target remote :33030
```
此时，调试器已经附加了在 Pin 下运行的应用程序，你可以设置断点，继续执行，打印变量，反汇编代码等。
```
(gdb) break main
Breakpoint 1 at 0x401194: file fibonacci.cpp, line 12.
(gdb) cont
Continuing.

Breakpoint 1, main (argc=2, argv=0x7fbffff3c8) at fibonacci.cpp:12
12          if (argc > 2)
(gdb) print argc
$1 = 2
(gdb) x/4i $pc
0x401194 <main+27>:     cmpl   $0x2,0xfffffffffffffe5c(%rbp)
0x40119b <main+34>:     je     0x4011c8 <main+79>
0x40119d <main+36>:     mov    $0x402080,%esi
0x4011a2 <main+41>:     mov    $0x603300,%edi
```

当然，你在调试器中观察到的所有信息都是应用程序的原始状态，Pin 和 tool 的 instrumentation 细节都隐藏了。
如，上面显示的汇编代码都是应用程序的指令，没有 tool 插入的指令。但是当使用 "cont" 或 "step" 命令执行程序时，Pin 下的 tool instrumentation 会正常运行。

<font color="red">注意：使用 "target remote"  命令连接 GDB 后，不能使用 "run" 命令，因为程序已经运行并停在第一条指令前，只需使用 "cont" 命令继续执行程序。</font>

## 2、Adding New Debugger Commands
本节主要讲述在不修改 GDB 的情况下，为 Pintool 增加自定义调试命令，此命令允许程序员通过 live debugger session 控制 Pintool，比如，设置 Pintool输出收集到的信息，或只在程序的特定阶段开启 instrumentation。

详细参见  PIN_AddDebugInterpreter() ，该 API 的回调函数如下：
```
static BOOL DebugInterpreter(THREADID tid, CONTEXT *ctxt, const string &cmd, string *result, VOID *)
{
    TINFO_MAP::iterator it = ThreadInfos.find(tid);
    if (it == ThreadInfos.end())
        return FALSE;
    TINFO *tinfo = it->second;

    std::string line = TrimWhitespace(cmd);
    *result = "";

    // [...]

    if (line == "stats")
    {
        ADDRINT sp = PIN_GetContextReg(ctxt, REG_STACK_PTR);
        tinfo->_os.str("");
        if (sp <= tinfo->_stackBase)
            tinfo->_os << "Current stack usage: " << std::dec << (tinfo->_stackBase - sp) << " bytes.\n";
        else
            tinfo->_os << "Current stack usage: -" << std::dec << (sp - tinfo->_stackBase) << " bytes.\n";
        tinfo->_os << "Maximum stack usage: " << tinfo->_max << " bytes.\n";
        *result = tinfo->_os.str();
        return TRUE;
    }
    else if (line == "stacktrace on")
    {
        if (!EnableInstrumentation)
        {
            PIN_RemoveInstrumentation();
            EnableInstrumentation = true;
            *result = "Stack tracing enabled.\n";
        }
        return TRUE;
    }

    // [...]

    return FALSE;  // Unknown command

}
```

 PIN_AddDebugInterpreter() API 允许 Pintool 为 扩展 GDB 命令创建句柄（handler），如上面的代码段实现了新命令 "stats" 和 "stacktrace on"。
 在 GDB 中使用 "monitor" 命令执行上述命令:
 ```
 (gdb) monitor stats
Current stack usage: 688 bytes.
Maximum stack usage: 0 bytes.
 ```

当用户输入一个扩展调试命令时，Pintool可以做很多事情，如，"stats" 命令打印 tool 收集到的信息，任何 tool 写入“结果”参数中的文本会输出到 GDB 控制台。

程序员还可以使用 调试扩展命令 启用/禁止 Pintool instrumentation，比如，"stacktrace on" 命令。如果想要在应用的程序初始启动阶段快速启动 Pintool，可以禁用 Pintool instrumentation，直到触发一个断点，然后使用扩展命令，只在程序员感兴趣的应用程序执行阶段开启 instrumentation 。在上面的 stack-debugger 示例中，PIN_RemoveInstrumentation() 函数移除所有的 instrumentation，然后当调试器继续执行应用程序时，tool 重新 instrument 代码。稍后会介绍，tool 的全局变量 "EnableInstrumentation" 调整插入过的 instrumentation 。

## 3、Semantic Breakpoints 语义断点
高级调试扩展的最后一个功能为：调用 tool analysis 中的 API 使程序停在断点处，此功能虽然简单，但很强大。Pintool 通过 instrumentation 查找复杂的条件，当满足条件时，停在断点处。

"stack-debugger" tool 对所有分配栈空间的指令进行插装，一旦应用程序的栈空间达到某一阈值，程序停在断点处。传统调试器并不能实现此功能，因为传统调试器不可能找到所有分配栈空间的指令。

以下是 "stack-debugger" tool 中的部分代码，功能为：使用 Pin instrumentation 检索所有分配栈空间的指令。
```
static VOID Instruction(INS ins, VOID *)
{
    if (!EnableInstrumentation)
        return;

    if (INS_RegWContain(ins, REG_STACK_PTR))
    {
        IPOINT where = IPOINT_AFTER;
        if (!INS_HasFallThrough(ins))
            where = IPOINT_TAKEN_BRANCH;

        INS_InsertIfCall(ins, where, (AFUNPTR)OnStackChangeIf, IARG_REG_VALUE, REG_STACK_PTR,
            IARG_REG_VALUE, RegTinfo, IARG_END);
        INS_InsertThenCall(ins, where, (AFUNPTR)DoBreakpoint, IARG_CONST_CONTEXT, IARG_THREAD_ID, IARG_END);
    }
}
```

 INS_RegWContain() 判断指令是否修改栈指针，如果是，在此条指令后插入 analysis 函数，检查应用程序的栈空间是否超过某一阈值。

 <font color="red">注意：</font>所有 instrumentation 的开启/禁用由全局标识变量 "EnableInstrumentation" 设置。因此，在不感兴趣的应用程序执行阶段，禁用 instrumentation（使用 "stacktrace off" 设置），快速执行程序；然后在感兴趣阶段开启 instrumentation （使用 "stacktrace on" 设置）。

如果应用程序的栈空间超过某一阈值， OnStackChangeIf() 返回 TRUE，如果返回值为 TRUE，调用 DoBreakpoint() 停在断点处。

```
static ADDRINT OnStackChangeIf(ADDRINT sp, ADDRINT addrInfo)
{
    TINFO *tinfo = reinterpret_cast<TINFO *>(addrInfo);

    // The stack pointer may go above the base slightly.  (For example, the application's dynamic
    // loader does this briefly during start-up.)
    //
    if (sp > tinfo->_stackBase)
        return 0;

    // Keep track of the maximum stack usage.
    //
    size_t size = tinfo->_stackBase - sp;
    if (size > tinfo->_max)
        tinfo->_max = size;

    // See if we need to trigger a breakpoint.
    //
    if (BreakOnNewMax && size > tinfo->_maxReported)
        return 1;
    if (BreakOnSize && size >= BreakOnSize)
        return 1;
    return 0;
}

static VOID DoBreakpoint(const CONTEXT *ctxt, THREADID tid)
{
    TINFO *tinfo = reinterpret_cast<TINFO *>(PIN_GetContextReg(ctxt, RegTinfo));

    // Keep track of the maximum reported stack usage for "stackbreak newmax".
    //
    size_t size = tinfo->_stackBase - PIN_GetContextReg(ctxt, REG_STACK_PTR);
    if (size > tinfo->_maxReported)
        tinfo->_maxReported = size;

    ConnectDebugger();  // Ask the user to connect a debugger, if it is not already connected.

    // Construct a string that the debugger will print when it stops.  If a debugger is
    // not connected, no breakpoint is triggered and execution resumes immediately.
    //
    tinfo->_os.str("");
    tinfo->_os << "Thread " << std::dec << tid << " uses " << size << " bytes of stack.";
    PIN_ApplicationBreakpoint(ctxt, tid, FALSE, tinfo->_os.str());
}
```

OnStackChangeIf() 跟踪栈使用情况，并判断栈空间是否达到阈值。如果时，返回非0（即 TRUE ），Pin执行 DoBreakpoint()。

DoBreakpoint() 的最后调用  PIN_ApplicationBreakpoint() ，停止所有线程的执行，并触发调试器中断。当中断触发时，GDB 的控制台输出  PIN_ApplicationBreakpoint() 的字符串参数，此参数告诉用户为什么会触发中断，本例中，此字符串为 "Thread 10 uses 4000 bytes of stack" 。

```
(gdb) monitor stackbreak 4000
Will break when thread uses more than 4000 bytes of stack.
(gdb) c
Continuing.
Thread 0 uses 4000 bytes of stack.
Program received signal SIGTRAP, Trace/breakpoint trap.
0x0000000000400e27 in Fibonacci (num=0) at fibonacci.cpp:34
(gdb)
```

此时，你可以继续执行程序，或终止程序：
```
(gdb) quit
The program is running.  Exit anyway? (y or n) y
```

## 4、Connecting the Debugger Later
上例使用 " -appdebug" 停止程序并从第一条指令开始调试，本小节将介绍程序不停在第一条指令之后，而是其他位置。
下例说明了如何使用 stack-debugger tool 启动应用程序，并只在触发“栈上限”断点时，调试器才附加程序。
```
$ ../../../pin -appdebug_enable -appdebug_silent -t obj-intel64/stack-debugger.so -stackbreak 4000 -- obj-intel64/fibonacci 1000
```

 -  -appdebug_enable选项，开始调试程序的位置不为第一条指令
 -   -appdebug_silent，自定义 GDB 控制台提示信息，详见下例
 -   -stackbreak 4000，当栈空间超过 4000 时，触发断点。当触发断点时，输出如下消息：
```
Triggered stack-limit breakpoint.
Start GDB and enter this command:
  target remote :45462
```

连接 GDB 的方式和之前一样，只是 GDB 停在“栈上限”断点处。
```
gdb fibonacci
(gdb) target remote :45462
0x0000000000400e27 in Fibonacci (num=0) at fibonacci.cpp:37
(gdb)
```

以下是 连接调试器的代码：
```
static void ConnectDebugger()
{
    if (PIN_GetDebugStatus() != DEBUG_STATUS_UNCONNECTED)
        return;

    DEBUG_CONNECTION_INFO info;
    if (!PIN_GetDebugConnectionInfo(&info) || info._type != DEBUG_CONNECTION_TYPE_TCP_SERVER)
        return;

    *Output << "Triggered stack-limit breakpoint.\n";
    *Output << "Start GDB and enter this command:\n";
    *Output << "  target remote :" << std::dec << info._tcpServer._tcpPort << "\n";
    *Output << std::flush;

    if (PIN_WaitForDebuggerToConnect(1000*KnobTimeout.Value()))
        return;

    *Output << "No debugger attached after " << KnobTimeout.Value() << " seconds.\n";
    *Output << "Resuming application without stopping.\n";
    *Output << std::flush;
}
```

每当 tool 需停在断点处，会调用 ConnectDebugger() 函数，它首先调用 PIN_GetDebugStatus() 判断是否和调试器连接。如果没有，使用 PIN_GetDebugConnectionInfo() 获取 TCP 端口号，用于连接 GDB，本例中的端口号为 "45462"，在 "target remote" 命令中使用。
 PIN_WaitForDebuggerToConnect() 等待连接 GDB，如果一段时间后，用户没有开启GBD并附加进程，tool 输出一条提示信息，然后继续执行程序。


# 参考文献
[Pin 用户手册：The Pin Advanced Debugging Extensions](https://software.intel.com/sites/landingpage/pintool/docs/67254/Pin/html/index.html#APPDEBUG)
