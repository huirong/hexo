title: 跳转到 shellcode 方法
date: 2015-09-23 21:47:52
tags: 
- 缓冲区溢出
- exploit 编写
categories: exploit 编写
---

假设：假设shellcode已经在栈中，并且没有开启DEP
本文将就此讲述关于执行/跳转至shellcode的其它方法，最后你要面临的问题就是缓冲区是否足够大。
<!--more-->

# I、jump/call 到一个指向shellcode 的寄存器。
利用这种技术，你可以利用这个包含shellcode地址的寄存器，将该地址置入EIP中来实现。

你可以利用程序运行时加载的DLL，去搜索”jump/call register”等操作指令所在的内存地址。当构造payload时，我们就可以利用该内存地址去覆盖EIP。当然，如果有一个直接指向shellcode 的寄存器可以利用，那也不是不可以的。

# II、pop return
如果栈顶并没有指向攻击者指定的缓冲区，但此缓冲区又起始于栈顶下方的数字节处，那么你就可以使程序执行一系列的POP指令和一个RET指令，以此将这些字节弹出栈（每POP 一次，ESP 指针就更接近shellcode 入口一步），直至正确的缓冲区入口处。执行RET指令后，ESP 中的当前栈值将放入EIP 中。

因此当ESP+x 包含我们的shellcode所在的缓冲区地址时（当在调试器中执行命令“d esp”时，你就可以看到在ESP+offset 中的缓冲区地址，但由于Intel x86 是属于小端法机器，因此数据可能是反序的），POP RET 方法还是可行的。

# III、push return 
 这种方法明显不同于“call register”技术。如果你找不到<jump register>或者<call register>的机器码，那么你可以简单将一个地址压栈，然后执行ret，因此你只需搜索ret 之后的push <register>指令即可。先查找这一串操作指令，再查找执行这串指令的地址，最后利用该地址覆盖EIP。

# Ⅳ、jmp [reg + offset]
 如果寄存器指向包含shellcode 地址的缓冲区，但其并没有指向shellcode 入口，那么你可以通过搜索操作系统或者应用程序加载的DLL 中的指令，并向该指令中的寄存器添加上所需的字节偏移量，然后跳转至寄存器所指向的地址。笔者将此种方法称为jmp [reg+ offset]。

# V、blind return
假设ESP 指向当前栈基址。RET 指令将从栈中‘pop’新值（4 字节），然后将那地址放入ESP 中。因此如果用RET 指令所在地址去覆盖EIP，那么你将会把ESP 中的值置入ESI。

如果缓冲区中的可用空间被限制了（EIP 被覆盖之后），但是在覆写EIP 之前还有不少空间可利用，那么你可以先在小空间的缓冲区中执行jump code，以跳转至缓冲区首部的关键shellcode。

# Ⅵ、SHE
每一程序中均有默认的异常处理程序，这是由操作系统提供的。因此即使程序原本就没有使用异常处理，但你也可以用自己的地址去覆盖SHE handler，以使其跳转至shellcode。

利用SHE 可以使exploit 在windows 平台下运行得更为稳定，但在利用SHE 编写exploit 之前，你需要先掌握一些知识。如果你编写的exploit 无法在被给的操作系统中运行，那么payload 可能会导致程序崩溃（触发异常）。因此你可以将“regular”exploit 配合覆盖SHE 的方式来编写exploit，以此编写出更为可靠的exploit。

在一个被覆写EIP 的典型栈溢出中，也可以利用SHE 技术来编写exploit，以使其运行得更为稳定，同时获取更大可用空间的缓冲区（覆盖EIP 以触发SHE„„真可谓一箭双雕）。
