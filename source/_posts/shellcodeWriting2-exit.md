title: shellcode 编写笔记2：退出shellcode
date: 2015-12-20 16:34:13
tags: exploit编写笔记
---

在 C 程序中，调用完MessageBox API 后，将会用2 个指令退出进程：LEAVE 和RET。在独立的应用程序中shellcode 能工作良好，但是我们的shellcode 要注入到另一个应用程序中。因此在调用MessageBox 后调用leave/ret 很有可能破坏原来的程序，使程序崩溃。
<!-- more -->

# 退出shellcode方法
有两种方法来退出shellcode：尽可能悄无声息的结束一切，但是我们也可以试着保持父进程继续运行...可能下次还能再次被exploit。
在此将会讨论第三种能够用来退出shellcode 的方法：
- process:用ExitProcess()
- SEH：强制产生一个异常调用。记住这种方法可能会使exploit 代码不停的运行。（如果那个原始的bug 是专门为这个例子设的SEH）
- thread：用ExitThread()

明显的，上面的方法没有一个能确保父进程不会崩溃或者当它被exploit 时会继续保持可利用性。我只是讨论三种技术（顺便提一下，这些在Metasploit 上都是有的）：））

# ExitProcess()
这个技术是基于一个windows API“ExitProcess”，可以在kernel32.dll 中找到。只有一个参数：ExitProcess 的退出码。它的值在调用这个API 前（0 意味着一切OK）必须入栈。

在XP SP3 上，ExitProcess（）这个API 能在0x7c81cafa 地址处找到:

![](https://ww4.sinaimg.cn/large/005CA6ZCjw1ez681dbpktj30ct07yq4q.jpg)

因此为了使shellcode 正常退出，我们必须把下面的指令加到shellcode 的底部，在MessageBox 函数被调用之后正常退出
```
xor eax, eax ; zero out eax (NULL)
push eax ; put zero to stack (exitcode parameter)
mov eax, 0x7c81cafa ; ExitProcess(exitcode)
call eax ; exit cleanly
```

或者用字节/机器码（使用mona插件转化，mona插件的使用，我之前的[博客](http://huirong.github.io/2015/12/18/mona/)有介绍）：
```
"\x33\xc0" //xor eax,eax => eax is now 00000000
"\x50" //push eax
"\xc7\xc0\xfa\xca\x81\x7c" // mov eax,0x7c81cafa
"\xff\xe0" //jmp eax = launch ExitProcess(0)
```

# SEH
退出shellcode 的第二种方法（同时使父进程继续运行）是触发一个异常（比如call 0x00）--就像这样：
```
xor eax,eax
call eax
```

因为这个代码明显比其他的短，它可能导致不可预期的结果。如果一个异常处理函数已经设置好，就可以在你的exploit 中利用异常处理函数（基于SEH 的exploit），然后这个shellcode就会循环。这在特定的情况下会OK（比如说，举个例子，你试着使机器可利用而不是只exploit一次）。

# ExitThread()
这个函数需要一个参数：退出码（跟ExitProcess())函数很像）。
