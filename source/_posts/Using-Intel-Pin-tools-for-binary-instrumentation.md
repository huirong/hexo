title: Linux Pin使用之设置Yama安全模块
date: 2015-12-30 19:51:34
tags: 插装
---
最近论文开始做试验了，需要用到二进制插装，学习过程与大家分享。
<!-- more-->

# Yama 安全模块
本文将介绍基本Intel Pin工具代码编写，但是在此之前，要注意的是Linux默认的 Security Module会禁止二进制插装。
 Yama is a Linux Security Module that collects a number of system-wide DAC security protections that are not handled by the core kernel itself.

 Yama使用sysctl进行控制，/proc/sys/kernel/yama/ptrace_scope 可能的值及其相应解释如下：
 ```
0 => Classic ptrace: a process can PTRACE_ATTACH to any other process running under the same uid, as long as it is dumpable.
1 => Restricted ptrace: a process can only PTRACE_ATTACH only its descendants although an inferior can call prctl(PR_SET_PTRACER, debugger, …) to allow the debugger to call PTRACE_ATTACH.
2 => Admin-only attach: only processes with CAP_SYS_PTRACE may use ptrace with PTRACE_ATTACH
3 => No attach: no processes may use ptrace with PTRACE_ATTACH nor via PTRACE_TRACEME.
 ```

想了解更多详细信息，请阅读[关于Yama的Linux内核文档](https://www.kernel.org/doc/Documentation/security/Yama.txt)。

回到Pin，如果 /proc/sys/kernel/yama/ptrace_scope 的值设置为1sysctl kernel.yama.ptrace_scope=0，会有如下信息：

![](https://ww2.sinaimg.cn/large/005CA6ZCgw1ezhya70r2tj30zm04j3zc.jpg)

为了允许调试程序，需要使用 sysctl 设置 /proc/sys/kernel/yama/ptrace_scope 的值
```
sysctl kernel.yama.ptrace_scope=0
```

![](https://ww2.sinaimg.cn/large/005CA6ZCgw1ezhyeydkzej30zo07n76x.jpg)

OK！！！！可以成功运行Pin工具了。。。。

# 参考文献
[Using Intel Pin tools for binary instrumentation](https://labs.portcullis.co.uk/blog/using-intel-pin-tools-for-binary-instrumentation/)
