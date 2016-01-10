title: Windows 溢出保护原理与绕过方法
date: 2015-09-17 10:44:40
tags: 缓冲区溢出
---
本文转载自[windows溢出保护原理与绕过方法概览](http://www.blogbus.com/riusksk-logs/80935313.html)
作者：riusksk（泉哥）
<!--more-->

本篇文章主要揭露Windows平台上的各种溢出保护机制原理及绕过方法
#①GS编译选项
##原理
通过VC++编译器在函数前后添加额外的处理代码，前部分用于由伪随机数生成的cookie并放入.data节段，当本地变量初始化，就会向栈中插入cookie，它位于局部变量和返回地址之间：

经GS编译后栈中的巨变变量空间分配情况：
```
sub esp 24h
mov eax,dword ptr [__security_cookie(408040h)]
xor eax,dword ptr [esp+24h]
mov dword ptr [esp+20h],eax
```
在函数尾部的额外代码用于函数返回时，调用security_check_cookie()函数，以判断cookie是否被修改过，当函数返回时的情况如下：
```
mov ecx,dword ptr [esp+20h]
xor ecx,dword ptr [esp+24h]
add esp,24h
jmp __security_check_cookie(4010B2h)
```

在缓冲区溢出利用时，如果将恶意代码从局部变量覆盖到返回地址，那么自然就会覆写cookie，当检测到与原始cookie不同时（也就是比较上面408040h 与 4010B2h 两处的cookie值），就会触发异常，最后终止进程。

##绕过方法
###1、猜测/计算cookie
Reducing the Effective Entropy of GS Cookie:<http://www.uninformed.org/?v=7&a=2&t=html>
自从覆盖SEH的方法出现后，这种方法已基本不用了，它没有后面的方法来得简便。

###2、覆盖SEH
由于当security_check_cookie()函数检测到cookie被更改后，会检查是否安装了安全处理例程，也就是SEH结点中保存的指针，如果没有，那么由系统的异常处理器接管，因为我们可以通过（pop pop ret）覆盖SEH来达到溢出的目的。
但对于SafeSEH保护的模块，就可能会导致exploit失效，关于它的绕过在后续部分在述。

辅助工具：OD插件safeSEH、pattern_create、pattern_offset、msfpescan、memdump

###2、覆盖虚表指针
堆栈布局：[局部变量][cookie][入栈寄存器][返回地址][参数][虚表指针]
当把虚表指针覆盖后，由于要执行虚函数得通过虚表指针来搜索，即可借此劫持eip。

#②SafeSEH
##原理
为了防止 SEH 结点被攻击者恶意利用，微软在 .net 编译器中加入 /sdeseh 编译选项引入 SafeSEH 技术。
编译器在编译时，将PE文件所有合法的异常处理例程的地址解析出来制成一张表，放在PE文件的数据块 (LQAJ-CON-FIG)中，并使用 shareuse 内存中的一个随机数加密，用于匹配检查。如果该 PE 文件不支持 SafeSEH ，则表中的地址为0。当 PE 文件被系统加载后，表中的内容被加密保存到 ntdll.dll 模块的某个数据区。在PE文件运行期间，如果发生异常需要调用异常处理例程，系统会逐个该例程在表中是否有记录：如果没有，则说明该例程非法，进而不执行该异常例程。

##绕过方法
利用SafeSEH保护模块之外的地址
对于目前的大部分Windows操作系统，其系统模块都受SafeSEH保护，可以选用未开启SafeSEH保护的模块来利用。
比如漏洞软件本身自带的dll文件，这个可以借助OD插件SafeSEH来查看进程中各模块是否开启SafeSEH保护。除此之外，也可以通过直接覆盖返回地址 ( jmp/call esp ) 来利用。另一种方法，如果 esp + 8 指向EXCEPTION_REGISTRATION结构，那么你仍然可以宣召一个 pop/pop/ret 指令组合（在加载模块的地址范围之外的空间），也可以正常工作。但如果你在程序的加载模块中找不到 pop/pop/ret 指令，你可以观察下 esp/ebp，查看下这些寄存器距离 nesp 的编译，接下来就是查找这样的指令：
```
call dword ptr[esp+nn] / jmp dword ptr[esp+nn] 
call dword ptr[ebp+nn] / jmp dword ptr[ebp+nn] 
call dword ptr[ebp-nn] / jmp dword ptr[ebp-nn]
```
（其中的 nn 就是寄存器的值到 nseh 的偏移，偏移 nn 可能是：esp+8，esp+14，esp+1c,esp+2c,esp+44,esp+50,ebp+0c,ebp+24,ebp+30,ebp-04,ebp-0c,ebp-18）.

如果遇到以上指令是以NULL字节结尾的，可将shellcode放置在SEH之前：
- 在nseh上放置后的跳转指令（跳转7字节：jmp 0xfffffff9）;
- 向后跳转足够长的地址以存放shellcode，并借此执行只shellcode；
- 把shellcode放在用于覆盖异常处理结构的指令地址之前

#③DEP
##原理
数据执行保护(DEP)是一套软硬件技术，能够在内存上执行额外检查以防止在不可运行的内存区域上执行代码。在Microsoft Windows XP Service Pack 2、Microsoft Windows 2003 Service Pack 1、Microsoft Windows XP Edition 2005、Microsoft Windows Vista 和 Windows 7中，由硬件和软件一起强制实施DEP。DEP有两种模式，吐过CPU支持内存页NX属性，就是硬件支持DEP。只有当处理器/系统支持NX/XD(禁止执行)时，Windows才能拥有硬件DEP，否则只能支持软件DEP，详单与只有SafeSEH保护。

##绕过方法
###1、ret2lib
思路：将返回地址指向lib库中的代码，而不是直接跳转到shellcode中去执行，进而实现恶意代码的运行。

可以在库中找到一段执行系统命令的大妈，比如system()函数，用它的地址覆盖返回地址，此时即使NX/XD禁止在堆栈上执行代码，但库中的代码依然是可以执行的。
- 函数system()可通过运行环境来执行其它程序，例如启动Shell等等。
- 另外，还可以通过VirtualProtect函数来修改恶意代码所在内存页面 的执行权限，然后再将控制转移到恶意代码，其堆栈布局如下所示：

##2、利用TEB突破DEP
在之前的《黑客防线》中有篇文章《SP2下利用TEB执行ShellCode》，有兴趣的读者可以翻看黑防出版的《缓冲区溢出攻击与防范专辑》，上面有这 篇文章。该作者在文中提到一种利用TEB（线程环境块）来突破DEP的方法，不过它受系统版本限制，只能在XP sp2及其以下版本的windows系统 上使用，因为更高版本的系统，其TEB地址是不固定的，每次都是动态生成的。
该方法的具体实现方法如下：
- 将返回地址覆盖成字符串复制函数的地址，比如lstrcpy，memcpy等等；
- 在返回地址之后用目标内存地址和shellcode地址覆盖，当执行复制操作时，就会将shellcode复制到目标内存地址，该目标内存地址位于TEB偏移0xC00的地方，它有520字节缓存用于ANSI-to-Unicode函数的转换；
- 复制操作结束后返回到shellcode地址并执行它。
此时其堆栈布局如下：
【shellcode】【save ebp】【lstrcpy】【TEB缓存地址，用于复制结束后返回到shellcode】【TEB缓存地址】【ShellCode地址】

##3、关闭DEP
关于此方法最原始的资料应该是黑客杂志《uninformed》上的文章《Bypassing Windows Hardware- enforced Data Execution Prevention》（http://www.uninformed.org/?v=2& a=4），另外也可以看下本人之前翻译的《突破win2003 sp2中基于硬件的DEP》（<http://bbs.pediy.com/showthread.php?t=99045>.），此方法的主要原理就是利用NtSetInformationProcess()函数来设置 KPROCESS 结构中的相关标志位，进而关闭DEP，KPROCESS结构中相关标志位情况如下：
```
0:000> dt nt!_KPROCESS -r
ntdll!_KPROCESS
. . .
+0x06b Flags       : _KEXECUTE_OPTIONS
  +0x000 ExecuteDisable     : Pos 0, 1 Bit
  +0x000 ExecuteEnable     : Pos 1, 1 Bit
  +0x000 DisableThunkEmulation   : Pos 2, 1 Bit
  +0x000 Permanent     : Pos 3, 1 Bit
  +0x000 ExecuteDispatchEnable   : Pos 4, 1 Bit
  +0x000 ImageDispatchEnable   : Pos 5, 1 Bit
  +0x000 Spare       : Pos 6, 2 Bits
```

当DEP 被启用时，ExecuteDisable 被置位，当DEP 被禁用，ExecuteEnable 被置位，当Permanent 标志置位时表示这些设置是最终设置，不可更改。代码实现：
```
ULONG ExecuteFlags = MEM_EXECUTE_OPTION_ENABLE;
NtSetInformationProcess(
  NtCurrentProcess(),   // ProcessHandle = -1
  ProcessExecuteFlags,   // ProcessInformationClass = 0x22（ProcessExecuteFlags）
  &ExecuteFlags,     // ProcessInformation = 0x2（MEM_EXECUTE_OPTION_ENABLE）
  sizeof(ExecuteFlags));   // ProcessInformationLength = 0x4
```

具体实现思路（以我电脑上VirtualBox虚拟机下的xp sp3为例）：
###1、将al设置为1，比如指令mov al,1 / ret，然后用该指令地址覆盖返回地址：
```
0:000> lmm ntdll
start    end        module name
7c920000 7c9b3000   ntdll      (pdb symbols)          c:\symbollocal\ntdll.pdb\1751003260CA42598C0FB326585000ED2\ntdll.pdb
0:000> s 7c920000 l 93000 b0 01 c2 04
7c9718ea  b0 01 c2 04 00 90 90 90-90 90 8b ff 55 8b ec 56  ............U..V
0:000> u 7c9718ea
ntdll!NtdllOkayToLockRoutine:
7c9718ea b001            mov     al,1
7c9718ec c20400          ret     4

```

由于上面的ret 4，因此要再向栈中填充4字节（比如0xffffffff）以抵消多弹出的4字节，如果选择的指令刚好是ret则无须再多填充4字节。

###2、跳转到ntdll!LdrpCheckNXCompatibility中的部分代码（从cmp al,1 开始，可通过windbg下的命令 uf ntdll!LdrpCheckNXCompatibility来查看其反汇编代码），比如以下地址就需要用0x7c93cd24来覆写堆栈上的第 二个地址：
```
ntdll!LdrpCheckNXCompatibility+0x13:
7c93cd24 3c01            cmp     al,1
7c93cd26 6a02            push    2
7c93cd28 5e              pop     esi
7c93cd29 0f84df290200    je      ntdll!LdrpCheckNXCompatibility+0x1a (7c95f70e)  ; 之前已将al置1，故此处实现跳转
```

###3、上面跳转后来到这里：
```
0:000> u 7c95f70e
ntdll!LdrpCheckNXCompatibility+0x1a:
7c95f70e 8975fc          mov     dword ptr [ebp-4],esi  ; [ebp-0x4]= esi = 2
7c95f711 e919d6fdff      jmp     ntdll!LdrpCheckNXCompatibility+0x1d (7c93cd2f)
```

###4、上面跳转后来到：
```
0:000> u 7c93cd2f
ntdll!LdrpCheckNXCompatibility+0x1d:
7c93cd2f 837dfc00        cmp     dword ptr [ebp-4],0
7c93cd33 0f85f89a0100    jne     ntdll!LdrpCheckNXCompatibility+0x4d (7c956831) ; 不相等再次实现跳转
```

###5、上面跳转后来到：
```
0:000> u 7c956831
ntdll!LdrpCheckNXCompatibility+0x4d:
7c956831 6a04            push    4     ;ProcessInformationLength = 4
7c956833 8d45fc          lea     eax,[ebp-4]
7c956836 50              push    eax      ;ProcessInformation = 2（MEM_EXECUTE_OPTION_ENABLE）
7c956837 6a22            push    22h       ;ProcessInformationClass = 0x22（ProcessExecuteFlags）
7c956839 6aff            push    0FFFFFFFFh
7c95683b e84074fdff      call    ntdll!ZwSetInformationProcess (7c92dc80)
7c956840 e92865feff      jmp     ntdll!LdrpCheckNXCompatibility+0x5c (7c93cd6d)
7c956845 90              nop
```

在这里调用函数ZwSetInformationProcess（），而其参数也刚好达到我们关闭DEP的各项要求.

###6、最后跳转到函数结尾：
```
0:000> u 7c93cd6d
ntdll!LdrpCheckNXCompatibility+0x5c:
7c93cd6d 5e              pop     esi
7c93cd6e c9              leave
7c93cd6f c20400          ret     4
```
最后的堆栈布局应为：
```
【AAA……】【al=1地址】【0xffffffff】【LdrpCheckNXCompatibility指令地址】【0xffffffff】【"A" x 54】【call/jmp esp】【shellcode】  
  ▲ 　　　　  ▲            ▲                       ▲                      ▲            ▲
填充数据　 返回地址 　 抵消ret 4的4字节     指令cmp al,0x1 的起始地址      平衡堆栈   调整NX禁用后的堆栈 
```
 
如果在禁用NX后，又需要读取esi或ebp，但此时它们又被我们填充的数据覆盖掉了，那么我们可以使用诸如push esp/pop esi/ret或者push esp/pop ebp/ret这样的指令来调整esi和ebp，以使关闭DEP后还能够正常执行。
辅助工具：ImmDbg pycommand插件（!pvefindaddr depxpsp3  + !findantidep）

##4、利用WPN与ROP技术
ROP（Return Oriented Programming）:连续调用程序代码本身的内存地址，以逐步地创建一连串欲执行的指令序列。
WPM（Write Process Memory）：利用微软在kernel32.dll中定义的函数比如：WriteProcess Memory函数可将数据写入到指定进程的内存中。但整个内存区域必须是可访问的，否则将操作失败。
具体实现方法参见我之前翻译的文章《利用WPN与ROP技术绕过DEP》：http://bbs.pediy.com/showthread.php?t=119300

##5、利用SEH 绕过DEP
启用DEP后，就不能使用pop pop ret地址了，因而采用pop reg/pop reg/pop esp/ret 指令的地址，指令 pop esp 可以改变堆栈指针，ret将执行流转移到nseh 中的地址上（用关闭NX 例程的地址覆盖nseh，用指向pop/pop /pop esp/ret 指令的指针覆盖异常处理器）。
辅助工具：ImmDbg插件!pvefindaddr

#④ASLR
##原理
ASLR（地址空间布局随机化）技术的主要功能是通过对系统关键地址的随机化，防止攻击者在堆栈溢出后利用固定的地址定位到恶意代码并加以运行。它主要对以下四类地址进行随机化：
- 堆地址的随机化；
- 栈基址的随机化；
- PE文件映像基址的随机化；
- PEB(Process Environment Block，进程环境块)地址的随机化。

它在vista,windows 2008 server,windows7下是默认启用的（IE7除外），非系统镜像也可以通过链接选项 /DYNAMICBASE(Visual Studio 2005 SP1 以上的版本，VS2008 都支持)启用这种保护,也可手动更改已编译库的 dynamicbase 位，使其支持ASLR 技术(把PE 头中的DllCharacteristics 设置成0x40 -可以
使用工具PE EXPLORER 打开库，查看DllCharacteristics 是否包含0x40 就可以知道是否支持ASLR 技术)。另外，也 可以使用Process Explorer来查看是否开启ASLR。启用ASLR后，即使你原先已经成功构造出exploit，但在系统重启后，你在 exploit中使用的一些固定地址就会被改变，进而导致exploit失效。

##绕过方法

###1、覆盖部分返回地址
对比下windows7系统启动前后OD中loaddll.exe的各模块基址，启动前：
```
可执行模块
基址       大小         入口       名称       文件版本          路径
00400000   00060000   00410070   loaddll                      D:\riusksk\TOOL\Ollydbg\loaddll.exe
6DDE0000   0008C000   6DDE1FFF   AcLayers   6.1.7600.16385 (  C:\Windows\AppPatch\AcLayers.dll
710E0000   00012000   710E1200   mpr        6.1.7600.16385 (  C:\Windows\System32\mpr.dll
71C50000   00051000   71C79834   winspool   6.1.7600.16385 (  C:\Windows\System32\winspool.drv
747F0000   00017000   747F1C89   userenv    6.1.7600.16385 (  C:\Windows\System32\userenv.dll
750A0000   0001A000   750A2CCD   sspicli    6.1.7600.16385 (  C:\Windows\System32\sspicli.dll
750C0000   0004B000   750C2B6C   apphelp    6.1.7600.16385 (  C:\Windows\System32\apphelp.dll
75190000   0000B000   75191992   profapi    6.1.7600.16385 (  C:\Windows\System32\profapi.dll
75420000   0004A000   75427A9D   KERNELBA   6.1.7600.16385 (  C:\Windows\system32\KERNELBASE.dll
75B50000   0000A000   75B5136C   LPK        6.1.7600.16385 (  C:\Windows\system32\LPK.dll
75B60000   0004E000   75B6EC49   GDI32      6.1.7600.16385 (  C:\Windows\system32\GDI32.dll
……
```

启动后：
```
可执行模块
基址         大小       入口       名称       文件版本          路径
00400000   00060000   00410070   loaddll                      D:\riusksk\TOOL\Ollydbg\loaddll.exe
6F510000   0008C000   6F511FFF   AcLayers   6.1.7600.16385 (  C:\Windows\AppPatch\AcLayers.dll
715B0000   00012000   715B1200   mpr        6.1.7600.16385 (  C:\Windows\System32\mpr.dll
72170000   00051000   72199834   winspool   6.1.7600.16385 (  C:\Windows\System32\winspool.drv
74C70000   00017000   74C71C89   userenv    6.1.7600.16385 (  C:\Windows\System32\userenv.dll
75520000   0001A000   75522CCD   sspicli    6.1.7600.16385 (  C:\Windows\System32\sspicli.dll
75540000   0004B000   75542B6C   apphelp    6.1.7600.16385 (  C:\Windows\System32\apphelp.dll
75610000   0000B000   75611992   profapi    6.1.7600.16385 (  C:\Windows\System32\profapi.dll
75690000   0004A000   75697A9D   KERNELBA   6.1.7600.16385 (  C:\Windows\system32\KERNELBASE.dll
759B0000   000CC000   759B168B   msctf      6.1.7600.16385 (  C:\Windows\System32\msctf.dll
75E60000   000AC000   75E6A472   msvcrt     7.0.7600.16385 (  C:\Windows\system32\msvcrt.dll
75F10000   0004E000   75F1EC49   GDI32      6.1.7600.16385 (  C:\Windows\system32\GDI32.dll
……
```

由此可见，各模块基址的高位是随机变化的，而低位是固定不变的，这里loaddll.exe不受ADSL保护，所以其基址没有随机化，如果是 Notepad.exe就有启用ASLR，还有其它经链接选项/DYNAMICBASE编译的程序也会启用ASLR。因此我们可以让填充字符只覆盖到返回 地址的一半，由于小端法机器的缘故，其低位地址在前，因此覆盖到的一半地址刚好处于低位，而返回地址的高位我们让它保持不变，所以我们必须在返回地址之前 的地址范围内（相当于漏洞函数所在的255字节空间地址）查找出一个可跳转到shellcode的指令，比如jmp edx(关键看哪一寄存器指向 shellcode)。除此之外，我们还必须将shellcode放在返回地址之前，不然连返回地址的高位也覆盖掉了，这是不允许的。纵观此法，相当的有 局限性，如果漏洞函数过短，可能就没有我们需要的指令了，这时就得另寻他法了。

###2、利用未启用ASLR的模块地址
这与之前绕过SafeSEH的方法类似，直接在未受ASLR保护的模块中查找跳转指令的地址来覆盖返回地址或者SEH结构，可以通过 Process Explorer或者ImmDbg命令插件!ASLRdynamicbase或者(!pvefindaddr noaslr)：来查看哪 些进程模块启用ASLR保护。

#⑤SEHOP
##原理
微软在Microsoft Windows 2008 SP0、Microsoft Windows Vista SP1和 Microsoft Windows 7中加入了另一种新的保护机制 SEHOP（Structured Exception Handling Overwrite Protection），它可作为SEH的扩展，用于检 测SEH是否被覆写。SEHOP的核心特性是用于检测程序栈中的所有SEH结构链表的完整性，特别是对最后一个SHE结构的检测。在最后一个SEH结构中 拥有一个特殊的异常处理函数指针，指向一个位于ntdll中的函数ntdll!FinalExceptHandler（）。当我们用 jmp 06 pop pop ret 来覆盖SEH结构后，由于SEH结构链表的完整性遭到破坏，SEHOP就能检测到异常从而阻止shellcode 的运行

##绕过方法

伪造SEH链表
由于SEHOP会检测SEH链表的完整性，那么我们可以通过伪造SEH链表来替换原先的SEH链表，进而达到绕过的目的。具体实现方法：

（1）查看SEH链表结构，可借助OD实现，然后记住最后一个SEH结构地址，以方便后面的利用；
（2）用JE(0x74) + 最后一个SEH结构的地址（由于地址开头是00，故可省略掉，可由0x74替代，共同实现4字节对齐）去覆盖nexSEH；
（3）用xor pop pop ret指令地址去覆盖SEH handle，其中的xor指令是用于将ZF置位，使前面的JE = JMP指令，进而实现跳转；
（4）在这两个SEH结构之前写入一跳转指令（JMP+8），以避免数据段被执行；
（5）在这两个SEH结构之间全部用NOP填充，如果两者之间还有其它SEH结构的话；
（6）将shellcode放置在最后一个SEH结构之后，即ntdll!FinalExceptHandler（）函数之后。

此时的堆栈布局如下：
```
【NOP…】【JMP 08】【JE XXXXXX】【xor pop pop ret】【NOP…】【JMP 08】【0xFFFFFFFF】【ntdll!FinalExceptHandler】【shellcode】
                        ▲              ▲                                  ▲                   ▲ 
           next SEH(指向0xffffffff)  SEH Handle                          next SEH           SEH Handle
```
更多信息可参见我之前翻译的《绕过SEHOP安全机制》：http://bbs.pediy.com/showthread.php?t=104707

#结论
本文简单地叙述了windows平台上的各类溢出保护机制及其绕过方法，但若结合实例分析的话，没有几万字是不可能完成的，因此这里概览一番，读者若想获 得相关的实例运用的资料，可参考文中提及一些paper，特别是由看雪论坛上dge兄弟翻译的《Exploit编写系列教程6》以及黑客杂志 《Phrack》、《Uninformed》上的相关论文。微软与黑客之间的斗争是永无休止的，我们期待着下一项安全机制的出现……
