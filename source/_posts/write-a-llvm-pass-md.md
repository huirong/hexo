---
title: 编写简单的LLVM Pass
date: 2017-03-18 09:59:44
tags: LLVM
categories: LLVM
---
LLVM Pass 框架是 LLVM 系统重要的组成部分，每个 Pass 完成特定的优化或转变工作，多个 Pass 协同工作实现 LLVM 系统的优化和转换。
编写 LLVM 的官方教程参见 [Writing an LLVM Pass](http://llvm.org/docs/WritingAnLLVMPass.html)。
<!-- more -->

# 1 设置 build 环境
llvm 源码包含 HelloPass 源码，编译安装 llvm 后，HelloPass 已编译链接为 LLVMHello.so ，因此本文为了重现编写简单 LLVM Pass 的全过程，从 0 开始编写 LLVM Pass，功能和Hello相同，只是换个名字 Test。

- 在 llvm/lib/Transforms 目录下新建 Test 文件夹
- `cd llvm/lib/Transforms/Test`
- 新建 CMakeLists.txt，内容如下
    ```
    add_llvm_loadable_module( LLVMTest
        Test.cpp

        PLUGIN_TOOL
        opt
    )
    ```
- 在 lib/Transforms/CMakeLists.txt 中添加代码 `add_subdirectory(Test)`

通过这种方式，当前文件夹下的 star.cpp 被编译链接为共享对象 $(LEVEL)/lib/LLVMTest.so，使用 opt 工具的 -load 选项动态加载。

# 2 基本代码
在 cd llvm/lib/Transforms/Test 目录下新建 Test.cpp。

```
#include "llvm/Pass.h"
#include "llvm/IR/Function.h"
#include "llvm/Support/raw_ostream.h"

using namespace llvm;

namespace {
struct Test : public FunctionPass {
  static char ID;
  Test() : FunctionPass(ID) {}

  bool runOnFunction(Function &F) override {
    errs() << "Test: ";
    errs().write_escaped(F.getName()) << '\n';
    return false;
  }
}; // end of struct Test
}  // end of anonymous namespace

char Test::ID = 0;
static RegisterPass<Test> X("Test", "Test Pass",
                             false /* Only looks at CFG */,
                             false /* Analysis Pass */);
```

llvm为了防止编译的中间结果分布在码源目录中，影响码源的结构。因此不支持目录内编译。需要在码源目录外创建额外的编译目录，文中为 build。

进入 build 文件夹，重新执行 make
![](https://ws1.sinaimg.cn/large/005CA6ZCly1fdqsfrhdkuj30k3039my0.jpg)
结束之后在同一目录下执行 make
![](https://ws1.sinaimg.cn/large/005CA6ZCgy1fdqsls2e5dj30k400l3ye.jpg)

然后在 build/lib/ 文件夹下会生成 LLVMTest.so
![](https://ws1.sinaimg.cn/large/005CA6ZCly1fdqsoj7pfqj30jz01yq38.jpg)

# 3 使用 opt 运行 Pass
新建 cpp 文件，使用 clang++ 编译为 bc 文件
```
opt -load ..ildb/LLVMTest.so -Test < star.bc > /dev/null
```

![](https://ws1.sinaimg.cn/large/005CA6ZCly1fdqsz74o6rj30k104gaas.jpg)

# 4 常见的 Pass
- [ImmutablePass](http://llvm.org/doxygen/classllvm_1_1ImmutablePass.html)
- [ModulePass](http://llvm.org/doxygen/classllvm_1_1ModulePass.html)
- [CallGraphSCCPass](http://llvm.org/doxygen/classllvm_1_1CallGraphSCCPass.html)
- [FunctionPass](http://llvm.org/doxygen/classllvm_1_1Pass.html)
- [LoopPass](http://llvm.org/docs/WritingAnLLVMPass.html#the-looppass-class)
- [RegionPass](http://llvm.org/docs/WritingAnLLVMPass.html#the-regionpass-class)
- [BasicBlockPass](http://llvm.org/docs/WritingAnLLVMPass.html#the-basicblockpass-class)
- [MachineFunctionPass](http://llvm.org/docs/WritingAnLLVMPass.html#the-machinefunctionpass-class)

# 5 参考文献
[Writing an LLVM Pass](http://llvm.org/docs/WritingAnLLVMPass.html)
