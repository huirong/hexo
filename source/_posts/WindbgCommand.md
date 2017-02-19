title: Windbg命令
date: 2015-09-15 20:53:31
tags: 
- 调试器
categories: 调试器
---
本文并非原创，转载自[Exploit开发系列教程-Windbg](http://drops.wooyun.org/tips/6813)
<!--more-->

##符号
打开某一WinDbg实例，如果你正使用Windbg调试某一进程，那么关闭WinDbg并将它重新打开。在File→Symbol File Path 里

输入：<font color="red">`SRV\*C:\windbgsymbols\*http://msdl.microsoft.com/download/symbols` </font>

保存工作区 (File→Save Workspace).

如上的星号是定义符。如上指定目录为本地符号缓存目录。paths/urls位于第二个星号后（如果有更多的paths/urls，那么使用‘；’分割）。用星号具体指定符号的位置。

##调试时调价符号
要在调试时追加符号的搜索路径,使用命令：<font color="red">.sympath+ c:\symbolpath</font>

(使用的命令如没有’+’,其作用是替换默认的搜索路径)

重载符号表: <font color="red">.reload</font>

##检查符号
如果需要了解模块加载了哪些符号，使用命令：<font color="red">x *!</font>

X命令支持使用通配符并可在搜索一个或多个模块中的符号时使用.例如，我们可以搜索kernel32内带有virtual字样开头的所有符号：

```
0:000> x kernel32!virtual*
757d4b5f          kernel32!VirtualQueryExStub (<no parameter info>)
7576d950          kernel32!VirtualAllocExStub (<no parameter info>)
757f66f1          kernel32!VirtualAllocExNuma (<no parameter info>)
757d4b4f          kernel32!VirtualProtectExStub (<no parameter info>)
757542ff          kernel32!VirtualProtectStub (<no parameter info>)
7576d975          kernel32!VirtualFreeEx (<no parameter info>)
7575184b          kernel32!VirtualFree (<no parameter info>)
75751833          kernel32!VirtualAlloc (<no parameter info>)
757543ef          kernel32!VirtualQuery (<no parameter info>)
757510c8          kernel32!VirtualProtect (<no parameter info>)
757ff14d          kernel32!VirtualProtectEx (<no parameter info>)
7575183e          kernel32!VirtualFreeStub (<no parameter info>)
75751826          kernel32!VirtualAllocStub (<no parameter info>)
7576d968          kernel32!VirtualFreeExStub (<no parameter info>)
757543fa          kernel32!VirtualQueryStub (<no parameter info>)
7576eee1          kernel32!VirtualUnlock (<no parameter info>)
7576ebdb          kernel32!VirtualLock (<no parameter info>)
7576d95d          kernel32!VirtualAllocEx (<no parameter info>)
757d4b3f          kernel32!VirtualAllocExNumaStub (<no parameter info>)
757ff158          kernel32!VirtualQueryEx (<no parameter info>)
```


在模块部分使用通配符：
```
0:000> x *!messagebox*
7539fbd1          USER32!MessageBoxIndirectA (<no parameter info>)
7539fcfa          USER32!MessageBoxExW (<no parameter info>)
7539f7af          USER32!MessageBoxWorker (<no parameter info>)
7539fcd6          USER32!MessageBoxExA (<no parameter info>)
7539fc9d          USER32!MessageBoxIndirectW (<no parameter info>)
7539fd1e          USER32!MessageBoxA (<no parameter info>)
7539fd3f          USER32!MessageBoxW (<no parameter info>)
7539fb28          USER32!MessageBoxTimeoutA (<no parameter info>)
7539facd          USER32!MessageBoxTimeoutW (<no parameter info>)
```

如想临时改变策略,立刻将所有模块的符号加载到WinDbg调试器，可以使用：<font color="red">ld*</font>

这可能会花去一段时间.可通过 Debug→Break 来停止调试。

##帮助
仅需输入<font color="red">.hh</font>或按F1打开帮助窗口。用以下命令得到指定命令的帮助信息：
```
.hh <command>
```
<font color="red"><command></font>为你想得到帮助信息的某个指定命令,或按F1，选择Index(索引)来搜索命令，从而得到其帮助信息.

## 调试模式 ##

### 本地调试 ###
可以调试某一新进程或某一正在运行的进程:

通过<font color="red">File→Open Executable</font>运行新进程以进行调试

通过<font color="red">File→Attach to a Process</font>附加到某一正运行的进程

###远程调试

至少使用如下两个选项来远程调试程序

1 如果你已在机器A上本地调试某一程序,那么使用如下命令(选择你想要的端口):
```
.server tcp:port=1234
```
此时开启服务器（WinDbg内）.转到 <font color="red">File→Connect to Remote Sessions</font>并输入：
```
tcp:Port=1234,Server=<IP of Machine A>
```
来指定端口和IP.

2 在机器A,用如下命令运行dbgsrv：
```
dbgsrv.exe -t tcp:port=1234
```
即可以在机器A启动服务器.

在机器B运行Windbg，接着 <font color="red">File→Connect to Remote Stub</font>，输入
```
tcp:Port=1234,Server=<IP of Machine A>
```
这里需要设置适当的参数。

你将看到 <font color="red">File→Open Executable</font>已无法选择，但你可以通过 <font color="red">File→Attach to a process</font>附加到进程 .这时可在机器A上看到进程列表。

如果要在机器A停止服务器，可用Task Manager（任务管理器）接着kill dbgsrv.exe。

##模块
当你加载某一可执行程序或附加到某一进程时,WinDbg将列出已加载的模块.如果你要再次列出模块,那么可输入：lmf 列出指定模块（ntdll.dll），可用: <font color="red">lmf m ntdll</font> 得到模块（ntdll.dll）的镜像头部信息: <font color="red">!dh ntdll</font>

带有‘!’符号的命令为扩展命令，这里的作用是显示指定模块的详细信息,等等。从某一外部DLL中导出某一外部命令，并且WinDbg内部会调用该命令。用户可创建他们自己的扩展程序来扩展WinDbg的功能。

当然了，你也可以使用模块的起始地址:
```
0:000> lmf m ntdll
start    end        module name
77790000 77910000   ntdll    ntdll.dll
0:000> !dh 77790000
```

##表达式
WinDbg支持使用表达式,这意味着，当需要某一值时,你可直接输入该值或输入与该值等价的表达式。例如，如果 EIP = <font color="red">77c6cb70</font>,那么 <font color="red">bp 77c6cb71</font> 和 <font color="red">bp EIP+1</font> 等价。

你也可以使用符号：<font color="red">u ntdll!CsrSetPriorityClass+0x41</font> 和寄存器: <font color="red">dd ebp+4</font> 数字默认用 <font color="red">base 16</font> 表示，添加前缀来明确使用的base所表示的进制格式：
```
0x123: base 16 (hexadecimal)   0n123: base 10 (decimal)    0t123: base 8 (octal)   0y111: base 2 (binary)
```

用命令.format来展示某一值的多种格式
```
0:000> .formats 123
 Evaluate expression:
 Hex:     00000000`00000123
 Decimal: 291
 Octal:   0000000000000000000443
 Binary:  00000000 00000000 00000000 00000000 00000000 00000000 00000001 00100011
 Chars:   .......#
 Time:    Thu Jan 01 01:04:51 1970
 Float:   low 4.07778e-043 high 0
 Double:  1.43773e-321
```

用’?’来对某个表达式求值，例如: <font color="red">? eax+4</font>

##寄存器与伪寄存器
在WinDbg中可支持多种伪寄存器(含有某些值). 用前缀‘$‘来指明其是伪寄存器.在使用寄存器或伪寄存器时，可以添加前缀’@’，这样做可让WinDbg确定该前缀之后的内容为某一寄存器而不是某一符号。

这有一些伪寄存器的范例：

<font color="red">$teb</font> 或 <font color="red">@$teb</font> (TEB的地址)

<font color="red">$peb</font> 或 <font color="red">@$peb</font> (PEB的地址)

<font color="red">$thread</font> 或 <font color="red">@$thread</font> (当前线程)

##异常
用sxe命令可中断某一特定的异常.例如,中断某一已被加载的模块,可输入:
```
sxe ld <module name 1>,...,<module name N>
```

例如,
```
sxe ld user32
```

查看异常类型的列表： <font color="red">sx</font>  用 <font color="red">sxi</font> 命令忽略某一异常: <font color="red">sxi ld</font>  使用该命令可让第一次输入的命令失效。

执行到single-chance和second-chance的异常处将会使Windbg中断 。它们并非是不同的异常类型。执行到异常处时，WinDbg将停止执行 ，并提示该位置为single-chance异常。 Single-chance意味着异常事件还没被发送到被调试的程序。当我们恢复执行时,WinDbg将异常事件发送到被调试的程序。如果被调试程序不处理异常,WinDbg将再次停止执行并提示此处为second-chance异常。

在我们测试EMET5.2时,我们需要忽略single-chance的单步异常（single step exceptions）。用如下命令实现: sxd sse

##断点
**软件断点：**

在某指令上设置断点时,WinDbg将指令的第一字节保存于内存并用0xCC覆盖它（操作码为”int 3”）。

当“int 3”指令被执行时,断点即被触发,那么执行将会被停止，且WinDbg通过重置它的首字节来重置该指令。

输入如下命令在位于0x4110a0地址的指令上设置断点:
```
bp 4110a0
```
第三次运行时激活0x4110a0地址的断点：
```
bp 4110a0 3
```
恢复执行（并在第一次触发的断点上停止）输入如下：<font color="red">g</font>

这是 “go“ 的缩写.运行直到到达某地址 (含有代码 ), 输入: <font color="red">g  (code location)</font>

WinDbg内将会在指定的位置上设置软件断点(如 ‘bp’ )，但此处的断点被触发后将会被删除.主要原因是使用 ‘g’ 设的是一次性软件断点.

###**硬件断点：**

使用特定的CPU寄存器设置硬件断点，它比软件断点更通用.事实上,它可中断执行或内存访问.硬件断点不会修改任意代码，甚至带有self modifying code。不幸的是，最多只能下4个硬件断点。

最简单的形式如下，命令格式为：
```
ba <mode> <size> <address> <passes (default=1)>
<mode> 可以是
‘e‘ （用于执行）
‘r‘ （用于读取存储器）
‘w‘ （用于写存储器）
```

`<size>`  是监控访问(当 <mode>是‘e’时，它总为1)指明位置的大小,其以字节的形式表示。 
`<address>` 为设置断点的位置
`<passes>` 激活断点时(查看 ’bp’ 用法的范例)需要的传递数,其起到计数器的作用.

笔记:在运行某一进程前，该进程不可能使用硬件断点。因为通过修改CPU寄存器(dr0,dr1,等等…)可以设置硬件断点，在开启进程及它的线程被创建时,寄存器将会被重置。

###**处理断点：**

列出断点类型: <font color="red">bl</font>

‘bl’表示断点列表（breakpoint list）. 例如：
```
0:000> bl
0 e 77c6cb70     0002 (0002)  0:**** ntdll!CsrSetPriorityClass+0x40
```

区域的位置，从左到右表示如下：

- 0：断点ID
- e: 断点状态,可以设置（enabled）或关闭（disabled）.
- 77c6cb70: 内存地址
- 0002（0002）: 在激活前余下的传递数（起到计数器作用），利用所有传递数来等待激活（当断点被创建时，将会指定该值） 0:***|*: 相关联的进程和线程.用星号代表该断点不是thread-specific。
- ntdll!CsrSetPriorityClass+0x40: 设置断点的位置（模块, 函数和偏移）

1. 关闭（disable）某一断点 `bd <breakpoint id>`
2. 删除断点 `bc <breakpoint ID>`
3. 删除所有断点 `bc *`

###**断点命令：**

每次某个断点被触发后将自动执行某个命令，可以使用如下命令：
```
bp 40a410 ".echo \"Here are the registers:\n\"; r"
```
另一个范例：自定义命令如下：
```
bp jscript9+c2c47 ".printf \"new Array Data: addr = 0x%p\\n\",eax;g"
```

##逐步执行
逐步执行有至少三种类型：

步进/跟踪(命令:t) 该命令中断每条指令的执行.如果执行到call指令或int指令,那么该命令将各自在调用函数的第一条指令或int handler上中断。 步过 (命令: p) 该命令能让每条指令（没有calls或ints，等等）执行后中断，如果你刚好执行到call或int指令，那么会在call或int指令执行后中断 步出 (命令: gu) 该命令(go up) 能让WinDbg恢复程序的执行，并且能在下一条ret指令执行后中断。在exit函数中经常使用到该命令。

还有其它两个用于exit函数的命令：

tt (trace to next return):等价于重复使用’t’命令并且在执行过程中遭遇的第一条ret指令上停止执行。 pt (step to next return):等价于重复使用‘p’命令并且在执行过程中遭遇的第一条ret指令上停止执行。

记录：使用tt命令会执行到函数内，如果你想到达当前函数的ret指令，那么改为使用pt命令。 pt和gu命令的不同点在于，使用pt命令将会在ret指令上中断，使用gu命令将会在ret指令后的下一条指令上中断。

这里是包含‘p‘ 和‘t‘命令的不同形式:
```
pa/ta <address>: step/trace 到地址。
pc/tc: step/trace 到 下一条 call/int 指令。
pt/tt: step/trace 到下一条 ret (discussed above at point 3)指令。
pct/tct: step/trace 到下一 条call/int 或 ret指令。
ph/th: step/trace 到下一分支的指令。
```

##查看内存
可使用‘d’或它的变量中的其中一种类型来展示（display）内存中的内容，
```
db: display bytes
dw: display words (2 bytes)
dd: display dwords (4 bytes)
dq: display qwords (8 bytes)
dyb: display bits
da: display null-terminated ASCII strings
du: display null-terminated Unicode strings
```
输入 .hh d 来查看其它变量。 ‘d’命令用相同的格式展示数据，正如大多数的d*命令那样（或如果不是单一数据则使用db）。

这些命令的(简化)格式为：d* [range]

这里，使用星号来描绘我们已列出的如上所有的变化，并且方框内应指明所选的范围。如果没有选好范围，那么在使用d*命令展示一部分数据后，将展示内存部分的数据。

可以用许多种方式指定范围：
```
1、<start address> <end address> 范例，db 77cac000 77cac0ff
2、<start address> L<number of elements> 范例, dd 77cac000 L10 查看 10 dwords（始于 77cac000地址）.Note: 因为
   范围比256 MB还要大,我们必须使用L?而不是L来指定行数。
3、<start address>
4、在只是指定起始地址时，用WinDbg将可以查看到128字节的内容。
```


##编辑（edit）内存
要编辑（edit）内存，使用:
```
e[d|w|b] <address> [<new value 1> ... <new value N>]
```
[d|w|b]是相关选项，它指定编辑的元素类型(d = dword, w = word, b = byte)。 如果新值被省略了，那么你在WinDbg中可以交互式地输入它们。

这是范例：<font color="red">ed eip cc cc</font>

用值0xCC来覆盖地址（在eip内）上的头两个dwords。

##搜索内存

使用‘s’命令来搜索内存。它的格式为：
```
s [-d|-w|-b|-a|-u] <start address> L?<number of elements> <search values>
```
<font color="red">d，w，b，a，u</font> 分别代表<font color="red">dword, word, byte, ascii</font> 和 unicode.

`<search values>`是序列值（用于搜索）

例如：
```
s -d eip L?1000 cc cc
```
在内存区间内搜索两个连续的 <font color="red">dwords 0xcc 0xcc。[eip, eip + 1000*4 – 1]</font>。

##指针

使用如下命令解引用某个指针：
```
dd poi(ebp+4)
```
用该命令，poi(ebp+4)对地址ebp+4求值，其结果的类型为dword或qword(在64位模式下)。

##使用于多个方面的命令

查看寄存器信息，输入如下：r

查看特定寄存器信息，例如eax和adx，输入：r eax, edx

打印前三行EIP指向的指令，用命令如下：u EIP L3

‘u‘ 是unassemble的缩写并且‘L‘可让指定你想查看信息的行数.列出调用栈（call stack）可以使用k
