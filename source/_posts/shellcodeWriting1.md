title: shellcode编写笔记1
date: 2015-12-20 11:45:26
tags: exploit编写笔记
---
这篇文章将只是涉及已经有的思想，引导你理解如何写并用通用的shellcode。没有包括
一些新的技术和新类型的shellcode。
<!-- more -->
# 实验环境
- C/C++编译器：dev c++
- 调试器：Immunity Debugger
- Windows SP3（预装Perl环境）
- 测试程序如下
```
char code[] = "paste your shellcode here";

int main(int argc, char **argv){	
	int(*func)();
	func = (int (*)()) code;
	(int)(*func)();
	
	return 0;
}
```
<font color="red">注意：</font>我是在XP SP3 平台下写这个教程的，所以如果你用不同版本的操作系统时一些地址会不同。
# 从 C 到shellcode
如果我们想写个显示一个MessageBox 的shellcode，MessageBox 上的文本是“You have been pwned by star”。这个exploit在实际生活中不怎么又用，但在你想写或者修改出更复杂的shellcode之前，这会教你一些基本技术。

我使用的是dev c++编译器，如果读者用的其他编译器，其中一些细节可能不一样，但概念和最后结果应该差不多。

### 1、源代码(a.cpp)：
```
#include <windows.h>
int main(int argc, char** argv){
	LoadLibrary("user32.dll");
	
	MessageBox(NULL,
		"You have been pwned by star",
		"star",
		MB_OK);
		
	return 0;
}
```
### 2、编译运行这个程序

![](https://ww1.sinaimg.cn/large/005CA6ZCgw1ez44ngew29j305a02uglm.jpg)

<font color="red">Tips：</font>MessageBox 函数在user32.dll中，Dev C++需要手动加载此库

### 3、查看可执行文件
使用 Immunity Debugger 打开刚生成的可执行文件a.exe，main函数如下：

![](https://ww1.sinaimg.cn/large/005CA6ZCgw1ez44sumlukj30kh08s41o.jpg)

### 4、分析
- push ebp 和mov ebp，esp 指令是用来设置堆栈的一部分指令。我们在自己的shellcode里不需要他们，因为我们会在一个已经运行的程序里运行我们的shellcode，我们假设堆栈已经被正确的设置好了。（这可能是不对的，在实际生活中，你需要调节寄存器/堆栈来使你的shellcode 工作，但这个暂时超出讨论的范围）。
- 我们把会用到的参数放到栈顶，按反序入栈。标题（a.0040400B）和MessageBox 文本（a.00404010）是从可执行文件的.data 节中取出的；按钮的样式（MB_OK）和句柄hOwner 都是0。
- 我们调用MessageBoxA 这个windows API（包含在user32.dll 中），这个API 有四个参数。

实际上，我们离把这个转化成可用的shellcode 不远。如果我们从上面的输出得到机器码字节，我们有了自己的基本shellcode。我们只需要改变几处地方就能使它工作：
1. 改变字符串（“Corelan”作为标题，“You have been pwned by Corelan”作为文本）放在栈中的方式。在我们的例子里，这些字符串是从C 程序的.data 节中取出来的。但是当我们exploit 另一个程序时，我们不能用特定程序的.data 节（因为它会包含一些其他的东西）。因此我们需要自己把字符串放到栈中，然后把指向字符串的指针传递给MessageBoxA 函数。
2. 找到MessageBoxA 的地址然后直接调用。Immunity Debugger打开a.exe，右键 search for -> Name in all Modules

![](https://ww4.sinaimg.cn/large/005CA6ZCgw1ez454c096oj30i60dtae7.jpg)

查找MessageBox的地址

![](https://ww1.sinaimg.cn/large/005CA6ZCgw1ez455hw52xj30ch02haai.jpg)

MessageBox的地址为：0x77D507EA
<font color="red">Tips：</font>这个地址将会和其他版本的操作系统上的地址不一样

# 把asm转化为 shellcode：将字符串入栈并且返回指向字符串的指针
- 把字符串转化为十六进制值
- 把十六进制值入栈（按反序）。不要忘了字符串末尾的null 字节，确保一切都是4 字节对齐（需要时加上一些空格）

接下来的小脚本将会产生把字符串入栈的机器码（a.pl)：
```
#!/usr/bin/perl
# Perl script written by Peter Van Eeckhoutte
# http://www.corelan.be:8800
# This script takes a string as argument
# and will produce the opcodes
# to push this string onto the stack
if ($#ARGV ne 0) {
	print " usage: $0".chr(34)."String to put on stack".chr(34)."\n";
	exit(0); 
}

#convert string to bytes
my $strToPush=$ARGV[0];
my $strThisChar="";
my $strThisHex="";
my $cnt=0;
my $bytecnt=0;
my $strHex="";
my $strOpcodes="";
my $strPush="";
print "String length :" . length($strToPush)."\n";
print "Opcodes to push this string onto the stack :\n\n";
while ($cnt < length($strToPush))
{
	$strThisChar=substr($strToPush,$cnt,1);
	$strThisHex="\\x".ascii_to_hex($strThisChar);
	if ($bytecnt < 3)
	{
		$strHex=$strHex.$strThisHex;
		$bytecnt=$bytecnt+1;
	}
	else
	{
		$strPush = $strHex.$strThisHex;
		$strPush =~ tr/\\x//d;
		$strHex=chr(34)."\\x68".$strHex.$strThisHex.chr(34).
		" //PUSH 0x".substr($strPush,6,2).substr($strPush,4,2).
		substr($strPush,2,2).substr($strPush,0,2);
		$strOpcodes=$strHex."\n".$strOpcodes;
		$strHex="";
		$bytecnt=0;
	}
	$cnt=$cnt+1;
}

#last line
if (length($strHex) > 0)
{
	while(length($strHex) < 12)
	{
		$strHex=$strHex."\\x20";
	}

	$strPush = $strHex;
	$strPush =~ tr/\\x//d;
	$strHex=chr(34)."\\x68".$strHex."\\x00".chr(34)." //PUSH 0x00".
	substr($strPush,4,2).substr($strPush,2,2).substr($strPush,0,2);
	$strOpcodes=$strHex."\n".$strOpcodes;
}
else
{
	#add line with spaces + null byte (string terminator)
	$strOpcodes=chr(34)."\\x68\\x20\\x20\\x20\\x00".chr(34).
	" //PUSH 0x00202020"."\n".$strOpcodes;
}

print $strOpcodes;

sub ascii_to_hex ($)
{
	(my $str = shift) =~ s/(.|\n)/sprintf("%02lx", ord $1)/eg;
	return $str;
}
```

例子：

![](https://ww1.sinaimg.cn/large/005CA6ZCgw1ez45cz5ovdj30i80c3gnz.jpg)

只把文本入栈是不够的。MessageBoxA 函数（就像其他的windows API 函数）希望得到指向文本的指针，而不是文本自身。因此我们必须把这个考虑进去。其他的两个参数（hWnd和ButtonType）不要是指针，只要设为0 就行了。因此我们需要对这两个参数用不同的方法。
```
int MessageBox(
	HWND hWnd,
	LPCTSTR lpText,
	LPCTSTR lpCaption,
	UINT uType
);
```
=>hWnd 和uType 的值从堆栈中取出，lpText 和lpCaption 是指向字符串的指针。

# 从asm到 shellcode：把 MessageBox的参数入栈
这就是我们要做的：
- 将字符串入栈然后把指向文本字符串的指针保存在寄存器中。因此在把字符串入栈后，必须把当前的堆栈位置保存在一个寄存器中。我们将用ebx 来存指向标题文本的指针，ecx来保存messagebox 的文本字符串的指针。当前栈顶位置=ESP。所以一个简单的movebx,esp 或者mov ecx,esp 就行了。
- 将其中的一个寄存器置为0，所以我们在需要时将它入栈（用作hWnd 和Button 的参数）。把一个寄存器置为0 可以简单的采取对自身XOR（xor eax,eax）。把0 和指针按正确的顺序，正确的位置入栈（指向字符串的指针）
调用MessageBox 函数（将会从堆栈的前四个地址并且把寄存器的内容作为MessageBox函数的参数）

除了这个之外，当我们在user32.dll 中看MessageBox 函数时，我们可以看到：

![](https://ww3.sinaimg.cn/large/005CA6ZCgw1ez45h270fnj30k408w77f.jpg)

明显参数是从一个位置指向EBP 的偏移处（从EBP+8 到EBP+14）。然后EBP 是从堆栈中弹出的ESP 的值0x77D507EA。因此意味着我们必须确认我们的四个参数被精确定位。这意味着，基于我们将字符串入栈的方式，我们要在跳转到MessageBox 这个函数之前再将4 个字节入栈。（只要在调试器里调试一下，你就会知道要做什么）。

# 从asm到 shellcode：将东西组装在一起
OK，生成最终的 code 数组,并粘贴到“shellcodetest.c”程序中，编译。
```
#include<stdio.h>
#include <windows.h>

char code[] =
//first put our strings on the stack
"\x68\x20\x20\x20\x00" // Push "star"
"\x68\x73\x74\x61\x72" // = star
"\x8b\xdc" // mov ebx,esp  字符串"star"的地址存入ebx
"\x68\x74\x61\x72\x00" // Push
"\x68\x62\x79\x20\x73" // "You have been pwned by Corelan"
"\x68\x6e\x65\x64\x20" // = Text
"\x68\x6e\x20\x70\x77" //
"\x68\x20\x62\x65\x65" //
"\x68\x68\x61\x62\x61" //
"\x68\x59\x6f\x75\x20" //
//"\x68\x59\x6f\x75\x20" //
"\x8b\xcc" // mov ecx,esp  文本 Text的地址存入 ecx
//现在将 参数、指针压入栈中
//最后一个参数 hwnd = 0.
//eax清零，入栈
"\x33\xc0" //xor eax,eax => eax is now 00000000
"\x50" //push eax
//第二个参数是 caption. 地址在ebx中, so push ebx
"\x53"
//下一个参数 text.地址在ecx中, so do push ecx
"\x51"
//最后一个参数 button (OK=0). eax的值依然为0
//so push eax
"\x50"
//四个参数已经依次压入栈中了
//但是MessageBox是从EBP+8开始读取参数的，需向栈中再次压入8bytes，调整栈针
//we will just add anoter push eax instructions to align
"\x50"
// 调用函数
"\xc7\xc6\xea\x07\xd5\x77" // mov esi,0x77d507EA
"\xff\xe6"; //jmp esi = launch MessageBox

int main(int argc, char **argv){
	LoadLibrary("user32.dll");
	int (*func)();
	func = (int (*)()) code;
	(int)(*func)();

	return 0;
}
```

<font color="red">Tips：</font>你可以通过简单的指令（Immunity Debugger：mona）来得到机器码

![](https://ww1.sinaimg.cn/large/005CA6ZCgw1ez45k9ksurj30c303yjrw.jpg)

最终弹出对话框

![](https://ww2.sinaimg.cn/large/005CA6ZCgw1ez45n4ake9j305c02wmx6.jpg)

# Immunity Debugger分析

# 参考文献
<https://www.corelan.be/index.php/2010/02/25/exploit-writing-tutorial-part-9-introduction-to-win32-shellcoding/>
[Exploit 编写系列教程第九篇Win32 Shellcode编写入门](http://bbs.pediy.com/showthread.php?t=120649)
