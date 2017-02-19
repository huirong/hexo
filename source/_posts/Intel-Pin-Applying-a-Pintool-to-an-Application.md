title: Intel Pin 6 ：Pin 应用到应用程序
date: 2016-01-12 21:07:14
tags: Pin
---
调用应用程序和 tool 的命令如下：
<!-- more -->
```
pin [pin-option]... -t [toolname] [tool-options]... -- [application] [application-option]..
```

以下是几条常见选项，完整列表参见 [ Command Line Switches](https://software.intel.com/sites/landingpage/pintool/docs/67254/Pin/html/group__KNOBS.html)
 - -t toolname，使用的Pintool
 - -pause_tool n，打印进程 ID，并暂停 n 秒，允许 GDB 附加调试。详见[Tips for Debugging a Pintool](http://huirong.github.io/2016/01/13/Intel-Pin-Tips-for-Debugging-a-Pintool/)
 
-- 之后的都是 应用程序命令行
例如，应用 itrace 运行 “ls” 程序。
```
../../../pin -t obj-intel64/itrace.so -- /bin/ls
```

使用如下命令获取可用的 Pin 命令行选项列表：
```
pin -help
```

使用如下命令获取 itrace 示例的可用命令行选项列表：
```
../../../pin -t obj-intel64/itrace.so -help -- /bin/ls
```

<font color="red">注意：</font>最后的 "/bin/ls" 是必选的，但不会执行。

# 限制
在受 "McAfee Host Intrusion Prevention"（McAfee主机入侵防御）杀毒软件保护的系统上运行 Pin 时，存在一个问题。我们并没有测试其他杀毒软件和 Pin 的兼容性。

Linux系统会通过  sysctl /proc/sys/kernel/yama/ptrace_scope 禁用 ptrace attach，此时 Pin 不能使用其默认的 injection mode。

解决方法：
 - 以 root 权限运行如下命令

    ```
    $ echo 0 > /proc/sys/kernel/yama/ptrace_scope
    ```
 - 使用 "-injection child" 选项，关于 child injection 的更多信息参见 [ Injection](https://software.intel.com/sites/landingpage/pintool/docs/67254/Pin/html/index.html#INJECTION)

# 参考文献
[Pin 用户手册：Applying a Pintool to an Application](https://software.intel.com/sites/landingpage/pintool/docs/67254/Pin/html/index.html#EX)
