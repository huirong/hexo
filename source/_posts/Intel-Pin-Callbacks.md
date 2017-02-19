title: Intel Pin 3 ：回调函数
date: 2016-01-12 14:32:20
tags: Pin
---
前几篇 Intel Pin example 已经介绍了几种使用 Pin API 注册回调函数的方法。
<!-- more -->
如：
 - INS_AddInstrumentFunction (INSCALLBACK fun, VOID *val)
 - TRACE_AddInstrumentFunction (TRACECALLBACK fun, VOID *val)
 - RTN_AddInstrumentFunction (RTNCALLBACK fun, VOID *val)
 - IMG_AddInstrumentFunction (IMGCALLBACK fun, VOID *val)
 - PIN_AddFiniFunction (FINICALLBACK fun, VOID *val)
 - PIN_AddDetachFunction (DETACHCALLBACK fun, VOID *val)

前四种分别在 instruction instrumentation（指令插桩）、 trace instrumentation（踪迹插桩）、Routine instrumentation（函数插桩）、Image instrumentation（镜像插桩）时使用。

所有注册函数，第二个参数相同。当不需要注册回调函数时，val 设置为 0 ，通常情况下，val 为指向一个类实例的指针。

注意：所有注册函数返回 PIN_CALLBACK 对象，可调用 PIN_CALLBACK 操作API 修改注册过的回调函数属性。（参见 [PIN callbacks](https://software.intel.com/sites/landingpage/pintool/docs/67254/Pin/html/group__PIN__CALLBACKS.html)）

# 参考文献
[Pin 用户手册 Callbacks](https://software.intel.com/sites/landingpage/pintool/docs/67254/Pin/html/index.html#CALLBACK)
