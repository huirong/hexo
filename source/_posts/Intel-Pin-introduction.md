title: Intel Pin 1 ：如何使用Pin进行插桩
date: 2015-12-30 20:31:36
tags: Pin
---
Pin是Intel公司开发的动态二进制插桩框架，可以用于创建基于动态程序分析工具，支持IA-32和x86-64指令集架构，支持windows和linux。
<!-- more -->
简单说就是Pin可以监控程序的每一步执行，提供了丰富的API，可以在二进制程序程序运行过程中插入各种函数，比如说我们要统计一个程序执行了多少条指令，每条指令的地址等信息。
# Pin
认识Pin的最好方法是认为它是一个JIT编译器。这个编译器的输入不是字节码而是普通的可执行文件。Pin截获这个可执行文件的第一条指令，产生新的代码序列。接着将控制流程转移到这个产生的序列。产生的序列基本上跟原来的序列是一样的，但是Pin保证在一个分支结束后重新获得控制权。重新获得控制权之后，Pin为分支的目标产生代码并且执行。Pin通过将所有产生的代码放在内存中，以便于重新使用这些代码并且可以直接从一个分支跳到另一个分支，这提高了效率。

在JIT模式，执行的代码都是Pin生成的代码。原始代码仅仅是用来参考。当生成代码时，Pin给用户提供了插入自己代码的机会（插桩）。

Pin的桩代码都会被实际执行的，不论他们位于哪里。大体上，对于条件分支存在一些异常，比如，如果一个指令从不执行，它将不会被插入桩函数。

# Pintools
从概念上讲，插桩包括两个组件：
 - 决定在哪里插入什么代码的机制
 - 插入点执行的代码

这两个组件就是** 桩 **和 ** 分析 ** 代码。两个组件都在一个单独的可执行体中，即Pintool。Pintools可以认为是在Pin中的的插件，它能够修改生成代码的流程。

Pintool在Pin中注册一些 ** 桩 ** 回调函数，每当Pin生成新的代码时就调用回调函数。这些回调函数代表了 ** 桩 ** 组件。它可以检查将要生成的代码，得到它的静态属性，并且决定是否需要以及在哪里插入 ** 分析函数 **。

 ** 分析函数 ** 收集关于程序的数据。Pin保证整数和浮点指针寄存器的状态在必要时会被保存和回复，允许传递参数给（分析）函数。

Pintool也可以注册一些事件通知类回调函数，比如线程创建和fork，这些回调大体上用于数据收集或者初始化与清理。

# Observations
由于Pintool类似插件一样工作，所以它必须处于Pin与被插桩的可执行文件的地址空间。这样，Pintool就能够访问可执行文件的所有数据。它也跟可执行文件共享文件描述符与进程其他信息。

Pin和Pintool从第一条指令控制程序。对于与共享库一起编译的可执行文件，这意味着动态加载器和共享库将会对Pintool可见。

当编写tools时，最重要的是调整分析代码而不是桩代码。因为桩代码执行一次，而分析代码执行许多次。

# Instrumentation Granularity（插桩粒度）
## 1、trace instrumentation（踪迹插桩）
如上所述，Pin的插桩是实时的。插桩发生在一段代码序列执行之前。我们把这种模式叫做踪迹插桩（trace instrumentation）。

踪迹插桩让Pintool在可执行代码每一次执行时都能进行监视和插桩。trace通常开始于选中的分支目标并结束于一个非条件分支（ unconditional branch ），包括调用(call)和返回(return)。Pin能够保证trace只在最上层有一个入口，但是可以有很多出口。如果在一个trace中发生分支，Pin从分支目标开始构造一个新的trace。Pin根据基本块(BBL)分割trace。一个基本块是一个有唯一入口和出口的指令序列。基本块中的分支会开始一个新的trace也即一个新的基本块。通常为每个基本块而不是每条指令插入一个分析调用。减少分析调用的次数可以提高插桩的效率。trace插桩利用了TRACE_AddInstrumentFunction API call。

注意，虽然Pin从程序执行中动态发现执行流，Pin的BBL与编译原理中的BBL定义不同。如，考虑生成下面的switch statement：
```
switch(i)
{
    case 4: total++;
    case 3: total++;
    case 2: total++;
    case 1: total++;
    case 0:
    default: break;
}
```
它将会产生如下的指令（在IA-32架构上）
```
.L7:
        addl    $1, -4(%ebp)
.L6:
        addl    $1, -4(%ebp)
.L5:
        addl    $1, -4(%ebp)
.L4:
        addl    $1, -4(%ebp)
```
在经典的基本块中，每一个addl指令会成为一个单指令基本块。然而对于不同的 switch cases，Pin 会产生包含 4 条指令的 BBL（当遇到.L7 case），3 条指令的基本块（当遇到.L6 case），如此类推。这就是说Pin的BBL个数会跟用书上的定义的BBL不一样。例如，这里当代码分支到.L7时，只有1个BBL，但是有四个经典的基本块被执行。

Pin也会拆散其他指令为BBL，比如cpuid,popf,和rep前缀的指令。因为rep前缀指令被当做隐式的循环，如果一个rep前缀指令不止循环一次，在第一迭代之后将会产生一个单指令的BBL，所以这种情形会产生比你预期多的基本块。

## 2、instruction instrumentation（指令插桩）
为了方便编写Pintool，Pin还提供了指令插桩模式（instruction instrumentation mode），让工具可以监视和插桩每一条指令。本质上来说这两种模式是一样的，编写Pintool时不需要在为trace的每条指令反复处理。就像在trace插桩模式下一样，特定的基本块和指令可能会被生成很多次。指令插桩用到了 INS_AddInstrumentFunction API call。

## 3、Image instrumentation（镜像插桩）
有时，进行不同粒度的插桩比trace更有用。Pin对这种模式提供了两种模式：** 镜像 **和 ** 函数 ** 插桩。这些模式是通过缓存插桩要求，因此需要额外的空间，这些模式也称作提前插桩。

 ** 镜像 ** 插桩让Pintool在IMG第一次导入的时候对整个image进行监视和插桩。Pintool的处理范围可以是镜像中的每个块(section，SEC)，块中的每个函数(routine, RTN)，函数中的每个指令（instruction, INS），因此可以在函数执行前后或指令执行前后进行插桩。
镜像插桩用到了 IMG_AddInstrumentFunction API call。镜像插桩依靠符号信息判断函数的边界，因此需要在PIN_Init之前调用PIN_InitSymbols。

## 4、Routine instrumentation（函数插桩）
 ** 函数 ** 插桩让Pintool在它所在的镜像首次加载时监视和插桩整个函数。Pintool的处理范围可以是函数里的每条指令。这里没有足够的信息把指令归并成基本块。可以在函数或指令执行前后进行插桩。函数插桩时Pintool的编写者能够更方便的在镜像插桩过程中遍历各个sections。
函数插桩用到了RTN_AddInstrumentFunction API call。插桩在函数结束后不一定能可靠地工作，因为当最后出现调用时无法判断何时返回。

注意在镜像插桩和函数插桩中，不可能知道一个(分析）函数会被执行（因为这些插桩发生在镜像载入时）。在Trace和Instruction中只有被执行的代码才会被遍历。

# Managed platforms support
Pin支持所有可执行文件包括托管的二进制文件。从Pin的角度来看，托管文件是一种自修改程序。有一种方法可以使Pin区分即时编译代码(Jitted代码)和其他所有动态生成的代码,并且将Jitted代码与合适的管理函数联系在一起。为了支持这个功能，运行管理托管平台的JIT compiler必须支持Jit Profiling API。

必须支持下面的功能：
 - RTN_IsDynamic() API,用来识别动态生成的代码。一个函数必须被Jit Profiling API标记为动态生成的。
 - Pin tool,可以使用RTN_AddInstrumentFunction API 插桩 Jitted 函数

为了支持托管平台，以下条件必须满足：
 - 设置INTEL_JIT_PROFILER32和INTEL_JIT_PROFILER64环境变量，以便占用pinjitprofiling dynamic library
对Windows
    
    ```
    set INTEL_JIT_PROFILER32=<The Pin kit full path>\ia32\bin\pinjitprofiling.dll
    set INTEL_JIT_PROFILER64=<The Pin kit full path>\intel64\bin\pinjitprofiling.dll
    ```

    对Linux
   
    ```
    setenv INTEL_JIT_PROFILER32 <The Pin kit full path>/ia32/bin/libpinjitprofiling.so
    setenv INTEL_JIT_PROFILER64 <The Pin kit full path>/intel64/bin/libpinjitprofiling.so
    ```

 - 在Pin命令行为Pin tool加入knob support_jit_api选项

# Symbols
Pin可利用符号对象（SYM）访问函数名。符号对象仅仅提供了在程序中的关于函数的符号。其他类型的符号（如数据符号）需要通过tool单独获取。

在Windows上，可以通过dbghelp.dll实现这个功能。注意在 桩函数 中使用dbghelp.dll并不安全，可能会导致死锁。一个可能的解决方案是通过一个不同的未被插桩的进程得到符号。

在Linux上，libefl.so或者libdwarf.so可以用来获取符号信息。

为了通过名字访问函数必须先调用PIN_InitSymbols。

# Floating Point Support in Analysis Routines
Pin在执行各种分析函数时保持着程序的浮点指针状态。

IARG_REG_VALUE不能作为浮点指针寄存器参数传给分析函数。
# Instrumenting Multi-threaded Applications
给多线程程序插桩时，多个合作线程访问全局数据时必须保证tool是线程安全的。Pin试图为tool提供传统C++程序的环境，但是在一个Pintool是不可以使用标准库的。比如，Linux tool不能使用pthread，Windows不能使用Win32API管理线程。作为替代，应该使用Pin提供的锁和线程管理API。

Pintools在插入桩函数时，不需要添加显示锁，因为Pin是在得到VM lock内部锁之后执行这些函数的。然而，Pin并行执行分析代码和替代函数，所以Pintools如果访问这些函数，可能需要为全局数据加锁。

Linux上的Pintools需要注意在分析函数或替代函数中使用C/C++标准库函数，因为链接到Pintools的C/C++标准函数不是线程安全的。一些简单C/C++函数本身是线程安全的，在调用时不需要加锁。但是，Pin没有提供一个线程安全函数的列表。如果有怀疑，需要在调用库函数的时候加锁。特别的，errno变量不是多线程安全的，使用这个变量的tool需要提供自己的锁。注意这些限制仅存在Unix平台，这些库函数在Windows上是线程安全的。

Pin可以在线程开始和结束的时候插入回调函数。这有助于 Pintool 分配和操作线程局部数据，并存放于线程局部内存。

Pin也提供了一个分析函数的参数（ARG_THREAD_ID），用于传递Pin指定的线程ID给调用的线程。这个ID跟操作系统的线程ID不同，它是一个从0开始的小整数。可以作为线程数据或是用户锁的索引。

除了Pin线程ID，Pin API提供了有效的线程局部存储(TLS），提供了分配新的TLS key并将它关联到指定数据的清理函数的选项。进程中的每个线程都能够在自己的槽中存储和取得对应key的数据。所有线程中key对应的value都是NULL。

False共享发生在多个线程访问相同的缓cache line的不同部分,至少其中之一是写。为了保持内存一致性，计算机必须将一个CPU的缓存拷贝到另一个，即使其他数据没有共享。可以通过将关键数据对其到cache line上或者重新排列数据结构避免False共享。

# Avoiding Deadlocks in Multi-threaded Applications
因为Pin,the tool和程序可能都会要求或释放锁，Pin tool的开发者必须小心避免死锁。死锁经常发生在两个线程以不同的顺序要求同样的锁。例如，线程A要求lock L1，接着要求L2,线程B要求lock L2，接着要求L1。如果线程A得到了L1，等待L2，同时线程B得到了L2，等待L1，这就导致了死锁。为了避免死锁,Pin为必须获得的锁强加了一个层次结构。Pin在要求任何锁时会要求自己的内部锁。我们假设应用将会在这个层次结构的顶端获得锁。下面的图展示了这种结构：
```
Application locks -> Pin internal locks -> Tool locks
```
Pin tool开发者在设计他们自己的锁时不应该破坏这个锁层次结构。下面是基本的指导原则：
 - 如果tool在一个Pin回调中要求任何锁，它在从这个回调中返回时必须释放这些锁。从Pin内部锁看来，在Pin回调中占有一个锁违背了这个层次结构。
 - 如果tool在一个分析函数中请求任何锁，它从这个分析函数中返回时必须释放这些锁。从Pin内部锁和程序自身看来，在分析函数中占有一个锁违背了这个层次结构。
 - 如果tool在一个Pin回调或者分析函数中调用Pin API，它不应该在调用API的时候占有任何锁。一些Pin API使用了内部Pin锁，所以在调用这些API时占有一个tool锁违背了这个层次结构。
 - 如果tool在分析函数里面调用了Pin API，它可能需要在要求Pin客户锁时调用PIN_LockClient()。这取决于API，需要查看特定API的更多信息。注意tool在调用PIN_Lockclient（）时，不能占有任何上述所述其他锁。

虽然上述的指导在大多数情况下已经足够，但是它们活血在某些特定的情形下显得比较严格。下面的指导解释了上述基本指导的放松情形：
 - 在JIT模式下，tool可能在分析函数中要求锁而不释放它们，直到将要离开包含这个分析函数的trace。tool必须期望trace在程序还没有抛出异常的时尽早退出。任何被tool占有的锁L在程序抛出异常时，必须遵守以下规则：
    + tool必须建立一个当程序抛出异常时的处理回调，这个回调会释放之前得到的锁L。可以使用PIN_AddContextChangeFunction()建立这个回调。
    + 为了避免破坏这个层次结构，tool禁止在Pin回调中要求锁。
 - 如果tool从一个分析函数中调用Pin API，如果在调用API时繁盛了下面情况，它可能会要求并占有一个锁L：
    + 锁L不是从任何Pin回调中请求的。这避免了违背这个层次结构。
    + 被调用的Pin API不会导致程序代码被执行。这避免了违背这个层次结构。

# 参考文献
[Intel Pin简介](http://terenceli.github.io/%E6%8A%80%E6%9C%AF/2014/01/02/intro-to-pin/)
[Pin 2.13 User Guide](https://software.intel.com/sites/landingpage/pintool/docs/65163/Pin/html/)
