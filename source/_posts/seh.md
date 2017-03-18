---
title: 结构化异常处理 SEH
date: 2017-02-19 20:31:24
tags: 
- 加密与解密
- 缓冲区溢出
categories: 加密与解密
---
结构化异常处理（Structured Exception Handing，SEH）是 Windows 操作系统处理程序错误或异常的技术，SEH 是 Windows 操作系统的一种系统机制，与特定的程序设计语言无关。
<!-- more -->

# 1 SEH 分类
当使用SEH时，代码执行与操作系统和编译器紧密相关，且编译器的工作量大于操作系统。
在进入和离开异常处理代码块时，编译器必须生成一些特殊的代码，以及产生一些关于支持 SEH 的数据结构表，还必须提供回调函数给操作系统调用，以便系统遍历异常代码块。变歧义还负责准备进程的栈框架（stack frame）和其他一些内部信息，这些信息都是操作系统需要使用或引用的。

SEH 实际上包含两个方面的功能：
1. <font color="red">终止处理（termination handling）</font>
    语法形式
```
__try {
    // 受保护模块
}
__finally {
    //终止处理模块
}
```
2. <font color="red">异常处理（exception handling）</font>
    语法形式
```
__try {
    // 受保护模块
}
__except(exception filter) {
    //异常处理模块
}
```

<font color="red"> Tips：不要混淆结构化异常处理与 C++ 异常处理。</font>C++异常处理在形式上表现为使用关键字 catch 和 throw ，这和结构化异常处理的形式不同，Microsoft Visual C++ 支持异常处理，它在内部实际上是利用编译器和 Windows 操作系统的结构化异常处理功能。

## 1.1 终止处理(termination handling)
### 1.1.1 语法
```
__try {
    // 受保护模块
} __finally {
    //终止处理模块
}
```
**__try** 和 **__finally** 关键字标记了终止处理程序的两个部分（受保护代码和终止处理程序），不管受保护代码是如何退出的，即无论在受保护代码中使用了 return，还是 goto ，又或者是 longjump语句（除非调用 ExitProcess，ExitThread，TerminateProcess，TerminateThread来终止进程或县城），终止处理程序都会被调用，即__finally 代码块都能执行。

### 1.1.2 局部展开
```
#include <iostream.h>
#include <windows.h>

int func(){
    int num = 0;

    __try {
        num = 5;
        return num;
        num = 15;
        cout << "__try num = " << num << endl;
    }
    __finally {
        num = 25;
        cout << "__finally execution and num = " << num << endl;
    }

    num = 35;
    return num;
}

int main(){
    cout << "main num = " << func() << endl;
    return 0;
}
```

![](https://ws1.sinaimg.cn/large/005CA6ZCly1fcwpuh4b7qj30ci02gmwz)

结果分析：
- func 中的 try 代码块中有一个 return 语句，这个return语句告诉编译器，在这里要退出当前函数并返回 num 变量的值，因此 num = 15 这个赋值语句和 cout 语句<font color="red"> 不会执行 </font> 。
- 根据 1.1.1 小节中说法，无论 try 是如何退出的，finally 都会执行。当 return 语句试图退出 try 块时，编译器会让 finally 代码块在它之前执行。即  <font color="red">编译器保障 finally 代码块在 try 块中的函数退出语句 return 之前执行</font>。因此会先执行赋值语句 num = 35，然后输出 "__finally execution and num = 35"。
- finally 代码块执行完以后，函数就可以返回了。因为 try 代码块中包含一个 return 语句，所以 <font color="red">finally 块之后的代码都没有机会执行 </font> ，因此这个函数的返回值是 5 ，而不是 35。

编译器是如何保证 finally 块可以在 try 代码块退出前被执行？
当编译器检查程序代码时，会发现 try 代码块里有个 return 语句，于是，生成一些代码现将返回值（本例中为 5 ）保存在一个由编译器创建的临时变量里，然后在执行 finally 代码块，这个过程称为 **<font color="red">局部展开</font>** 。更确切地说，当系统因为 try 代码块中的代码提前退出而执行 finally 代码块时，就会发生局部展开。一旦 finally 代码块执行完毕，编译器所创建的临时变量的值就会返回给函数的调用者。

 **Tips：应该尽量避免在 try 块中使用 return，goto，break，continue 语句，减少系统开销。**
- 因为，为了让整个机制运行起来，编译器必须生成一些额外代码，而系统也必须执行一些额外的工作。在不同 CPU 体系结构上，让终止处理工作起来的步骤也不同。
- 如果代码控制流正常低离开 try 代码块进入 finally 代码块，那么进入 finally 代码块的额外开销是最小的。

### 1.1.3 关键字 __leave
在try块中使用__leave关键字会使程序跳转到try块的结尾，从而自然的进入finally块，所以不会产生任何额外开销。

## 1.2 异常处理（exception handling）
### 1.2.1 语法
```
__try {
    // 受保护模块
}
__except(exception filter) {
    //异常处理模块
}
```

<font color="red">Tips：</font> try 块后面必须跟一个 finally 代码块或 except 代码块。但是 try 块后不能同时有 finally 块和except 块，也不能同时有多个 finally 块或 except 块。不过，却可以将 try-finally 块嵌套于 try-except 块中，反之亦然。

exception filter 必定为以下三个标识符之一，这些标识符在 Microsoft Windows 的 Excpt.h 文件中定义

标识 | 值 
:----:|------
EXCEPTION_EXECUTE_HANDLER | 1  
EXCEPTION_EXECUTE_SEARCH | 0  
EXCEPTION_CONTINUE_EXECUTION | -1 

下面是两种基本使用方法：
**(1)直接使用过滤器的三个返回值之一**
```
__try {   
    …… 
} 
__except ( EXCEPTION_EXECUTE_HANDLER ) {
    …… 
}
```

**(2)自定义过滤器**
```
__try {     
    ……
}  
__except ( MyFilter( GetExceptionCode() ) ) {
    …… 
}   
LONG MyFilter ( DWORD dwExceptionCode )  {    
    if ( dwExceptionCode == EXCEPTION_ACCESS_VIOLATION ){
        return EXCEPTION_EXECUTE_HANDLER
    }else{
        return EXCEPTION_CONTINUE_SEARCH
    }        
}
```

### 1.2.2 系统异常处理流程
不同的标识符，系统处理流程不一样，系统处理异常的过程如下图所示：
![](https://ws1.sinaimg.cn/large/005CA6ZCly1fcx0hkd0syj30f00fi74c)
 Tips：1.1 小节中提到，不要在终止处理程序中使用 return，goto，continue，break 语句， ** <font color="red"> 但是在异常处理程序的 try 块中，这些语句不会带来局部展开这样的额外开销。 </font> **

### 1.2.3 EXCEPTION_EXECUTE_HANDLER
EXCEPTION_EXECUTE_HANDLER 相当于告诉系统，“我知道这个异常，并预计这个异常在某种情况下会发生，同时已经写了一些代码来处理它，让这些代码现在就执行吧。”于是，系统执行全局展开（global unwind）,程序跳转到except代码块中执行， ** <font color="red"> 当 except 块中的代码执行完毕后，代码从 except 块后的第一句代码继续。 </font> **

### 1.2.4 全局展开
全局展开流程图如下:

![](https://ws1.sinaimg.cn/large/005CA6ZCgy1fcx5enheb8j30e70crt8t)

```
#include <iostream.h>
#include <windows.h>

static unsigned int nStep = 1;

void FunB(){
    int x,y = 0;
    __try{
        x = 5 / y;
    }
    __finally{
        cout << "Step " << nStep++ << " : 执行Function_B的finally块的内容" << endl;
    }
}

void FunA(){
    __try{
        FunB();
    }
    __finally{
        cout << "Step " << nStep++ << " : 执行Function_A的finally块的内容" << endl;
    }
}

long MyExcepteFilter() {
    cout << "Step " << nStep++ << " : 执行main的异常过滤器" << endl;
    return EXCEPTION_EXECUTE_HANDLER;
}

int main(){
    __try{
        FunA();
    }
    __except ( MyExcepteFilter() ){
         cout << "Step " << nStep++ << " : 执行main的except块的内容" << endl;
    }

    return 0;
}
```

![](https://ws1.sinaimg.cn/large/005CA6ZCgy1fcx0q8di86j30c803emwx)

**暂停全局展开**    
如果程序中出现异常，且已经找到过滤器值为EXCEPTION_EXECUTE_HANDLER所对应的try块，此时系统会进行全局展开，在正常情况下，系统会执行该try块以内的所有finally过程，然后再执行该try块对应的异常处理过程。但如果在某个finally中放入一个return，就可以阻止全局展开。

# 2 SEH 相关数据结构
## 2.1 TIB 结构
 TEB（Thread Environment Block，线程环境块）在Windows9X系列中称为TIB（Thread Information Block，线程信息块），它记录着线程的重要信息。每一个线程对应一个TEB结构，在Windows2000 DDK中定义为：
```
typedef struct _NT_TIB {
    struct _EXCEPTION_REGISTRATION_RECORD *ExceptionList;
    PVOID StackBase;
    PVOID StackLimit;
    PVOID SubSystemTib;
    union {
        PVOID FiberData;
        DWORD Version;
    };
    PVOID ArbitraryUserPointer;
    struct _NT_TIB *Self;
} NT_TIB;
```
与异常处理相关的项是指向_EXCEPTION_REGISTRATION_RECORD 结构的指针*ExceptionList，正好位于TEB的偏移0处。Windows在创建线程时，操作系统均会为每个线程分配TEB，而且都将FS段选择器指向当前线程的TEB数据，即TEB总是由[FS：0]指向的。

## 2.2 EXCEPTION_REGISTRATION结构
TEB 偏移量为 00h 的 `_EXCEPTION_REGISTRATION_RECORD` 主要用于描述线程异常处理过程的地址，多个结果的链表表述多个线程异常李处过程的嵌套层次关系，其定义如下：
```
_EXCEPTION_REGISTRATION struc
    prev dd ?       ;指向前一个EXCEPTION_REGISTRATION结构的指针
    handler dd ?        ;当前异常处理回调函数的地址
_EXCEPTION_REGISTRATION ends
```

其中，prev是指向前一个EXCEPTION_REGISTRATION（简称ERR）的指针，形成一个链状结构；handler指向异常处理代码，当系统遇到一个它不知道如何处理的异常时，它就查找异常处理链表。每个线程都有它自己的异常处理链表。

## 2.3 EXCEPTION_POINTERS、EXCEPTION_RECORD、CONTEXT结构
当一个异常发生时，操作系统向一起异常的线程的栈中压入 **EXCEPTION_POINTERS** 结构，EXCEPTION_POINTERS 结构包含两个指针，一个指向 EXCEPTION_RECORD 结构，一个指向 CONTEXT 结构.
```
typedef struct _EXCEPTION_POINTERS {
    PEXCEPTION_RECORD ExceptionRecord; //指向ExceptionRecord的指针
    PCONTEXT ContextRecord; //指向Context结构的指针
} EXCEPTION_POINTERS, *PEXCEPTION_POINTERS;
```

**EXCEPTION_RECORD 结构**包含有关最近发生的异常的详细信息，这些信息独立于 CPU，其结构如下：
```
typedef struct _EXCEPTION_RECORD {
    DWORD    ExceptionCode;     //异常码，以STATUS_或EXCEPTION_开头，可自定义。（sehdef.inc）
    DWORD ExceptionFlags;      //异常标志。0可修复；1不可修复；2正在展开，不要试图修复
    struct _EXCEPTION_RECORD *ExceptionRecord; //指向嵌套的异常结构，通常是异常中又引发异常
    PVOID ExceptionAddress;    //异常发生的地址
    DWORD NumberParameters;    //下面ExceptionInformation所含有的dword数目
    ULONG_PTR ExceptionInformation[EXCEPTION_MAXIMUM_PARAMETERS];    //附加消息，如读或写冲突
} EXCEPTION_RECORD;
```

CONTEXT 结构是 Win32 API 一个几乎唯一与处理器相关的结构，包括了线程运行时处理各主要寄存器的完整镜像，用于保存线程运行时环境。

```
typedef struct _CONTEXT {
     DWORD ContextFlags; //用来表示该结构中的哪些域有效
     DWORD   Dr0, Dr2, Dr3, Dr4, Dr5, Dr6, Dr7; //调试寄存器
     FLOATING_SAVE_AREA FloatSave; //浮点寄存器区
     DWORD   SegGs, SegFs, SegEs, Seg Ds; //段寄存器
     DWORD   Edi, Esi, Ebx, Edx, Ecx, Eax; //通用寄存器组
     DWORD   Ebp, Eip, SegCs, EFlags, Esp, SegSs; //控制寄存器组

     //扩展寄存器，只有特定的处理器才有
     BYTE    ExtendedRegisters[MAXIMUM_SUPPORTED_EXTENSION];
} CONTEXT;
```

## 2.4 异常处理链结构图
![](https://ws1.sinaimg.cn/large/005CA6ZCgy1fcx28cgmgxj31kw0yeagl)

# 3 参考文献
- 《Windows 核心编程（第 5 版）》
- 《加密与解密》