---
title: ".NET运行时中的监测和可观测性"
date:  2019-03-30T23:18:36+08:00
lastmod: 2019-03-31T20:05:51+08:00
draft: false
author: "WithLin"
tags: ["net", "可观测性"]
categories: ["net"]
toc: true
comment: true
autoCollapseToc: false
---

>今年5月份的时候研究分布式追踪的问题知道了的拦截方式比较零散， 刚好8月份的时候看到这篇文章，这个文章总结的比较完整。存档了很久，趁今天有空翻译给大家。[原文地址](http://mattwarren.org/2018/08/21/Monitoring-and-Observability-in-the-.NET-Runtime/)

.NET是一个[*托管运行时*](https://en.wikipedia.org/wiki/Managed_code)，这意味着它提供了“管理”您的程序的高级功能，从[简介到公共语言运行时（CLR）](https://github.com/mattwarren/coreclr/blob/master/Documentation/botr/intro-to-clr.md#fundamental-features-of-the-clr)（2007年编写）：

 运行时具有许多功能，因此按如下方式对它们进行分类很有用：
1. **基本功能**  对其他功能设计有广泛影响的功能。这些包括：
  1.垃圾收集
  2.记忆安全和类型安全
  3.对编程语言的高级支持。
2. **辅助功能** - 许多有用的程序可能不需要基本特性所支持的功能：
1.使用AppDomains进行程序隔离
2.程序安全和沙盒
3.  **其他功能** - 所有运行时环境都需要但不利用CLR基本功能的功能。相反，它们是创建完整编程环境的结果。其中包括：
1.版本
2.**Debugging/Profiling**
3.互操作


您可以看到，“**Debugging/Profiling**”虽然不是基本或辅助功能，但由于“ 需要创建完整的编程环境 ” ，它仍然会进入列表。

**这篇文章的其余部分将看*什么* [监测](https://en.wikipedia.org/wiki/Application_performance_management)，[可观测性](https://zh.wikipedia.org/wiki/%E5%8F%AF%E8%A7%80%E6%B8%AC%E6%80%A7)和[内省](https://zh.wikipedia.org/wiki/%E5%86%85%E7%9C%81_(%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6))功能核心CLR提供，*为什么*他们是有用的，*如何*提供他们。**


为了便于浏览，帖子分为3个主要部分（最后有一些“额外阅读材料”）：

*   [诊断(Diagnostics)](http://mattwarren.org/2018/08/21/Monitoring-and-Observability-in-the-.NET-Runtime/#diagnostics)
    *   Perf View(性能分析工具)
    *   共同基础设施
    *   未来的计划
*   [剖析(Profiling)](http://mattwarren.org/2018/08/21/Monitoring-and-Observability-in-the-.NET-Runtime/#profiling)
    *   ICorProfiler API
    *   分析 v .调试
*   [调试(Debugging)](http://mattwarren.org/2018/08/21/Monitoring-and-Observability-in-the-.NET-Runtime/#debugging)
    *   ICorDebug API
    *   SOS和DAC
    *   第三方调试器
    *   记忆转储


## 诊断(Diagnostics)

首先，我们将查看CLR提供的**诊断**信息，传统上这些信息是通过[“Windows事件跟踪”](https://docs.microsoft.com/en-us/windows/desktop/etw/about-event-tracing)（ETW）提供的。

[CLR提供的](https://docs.microsoft.com/en-us/dotnet/framework/performance/clr-etw-keywords-and-levels)各种事件涉及：

*   垃圾收集（GC）
*   即时（JIT）编译
*   模块和AppDomains
*   线程和锁争用
*   以及更多

例如，这是[触发AppDomain Load事件的](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/corhost.cpp#L649)地方，这是[Exception Thrown事件](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/exceptionhandling.cpp#L203)，这里是[GC Allocation Tick事件](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/gctoclreventsink.cpp#L139-L144)。

### Perf View

如果你想看到来自你的.NET程序的ETW事件，我建议使用优秀的[PerfView工具](https://github.com/Microsoft/perfview)，从这些[PerfView教程](https://channel9.msdn.com/Series/PerfView-Tutorial)开始，或者这个优秀的演讲[PerfView：终极.NET性能工具](https://www.slideshare.net/InfoQ/perfview-the-ultimate-net-performance-tool)。PerfView被广泛认可，因为它提供了宝贵的信息，例如Microsoft工程师经常将其用于[性能调查](https://github.com/dotnet/corefx/issues/28834)。

![image.png](https://upload-images.jianshu.io/upload_images/5590759-af39d031358bb86b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 共同基础设施

但是，如果从名称中不清楚，ETW事件仅在Windows上可用，这并不适合新的.NET“跨平台”世界。您可以[在Linux上](https://github.com/dotnet/coreclr/blob/release/2.1/Documentation/project-docs/linux-performance-tracing.md)使用[PerfView进行性能跟踪](https://github.com/dotnet/coreclr/blob/release/2.1/Documentation/project-docs/linux-performance-tracing.md)（通过[LTTng](https://lttng.org/)），但这只是cmd-line集合工具，称为“PerfCollect”，分析和丰富的UI（包括[flamegraphs](https://github.com/Microsoft/perfview/pull/502)）目前仅适用于Windows。

但是如果你想分析.NET Performance Linux，还有其他一些方法：

*   [在Linux上使用.NET Core获取LTTng事件的堆栈](https://blogs.microsoft.co.il/sasha/2018/02/06/getting-stacks-for-lttng-events-with-net-core-on-linux/)
*   [Linux性能问题](https://github.com/dotnet/coreclr/issues/18465)

上面的第二个链接讨论了在.NET Core中正在使用的新**“EventPipe”基础架构**（以及EventSources和EventListeners，你能发现一个主题！），你可以看到它在[跨平台性能监控设计中的目标](https://github.com/dotnet/designs/blob/master/accepted/cross-platform-performance-monitoring.md)。在高层次上，它将为CLR提供一个单独的位置来推动与诊断和性能相关的“事件”。然后，这些“事件”将被路由到一个或多个记录器，例如，可能包括ETW，LTTng和BPF，精确记录器由CLR运行的OS /平台确定。[.NET Cross-Plat性能和事件设计](https://github.com/dotnet/coreclr/blob/master/Documentation/coding-guidelines/cross-platform-performance-and-eventing.md)中还有更多背景信息可以解释不同日志记录技术的优缺点。

“事件管道”中正在进行的所有工作都在[“性能监控”项目](https://github.com/dotnet/coreclr/projects/5)和相关的[“EventPipe”问题中](https://github.com/dotnet/coreclr/search?q=EventPipe&type=Issues)进行跟踪。

### 未来的计划

最后，还有一个[性能分析控制器的(Performance Profiling Controller )](https://github.com/dotnet/designs/blob/master/accepted/performance-profiling-controller.md)未来计划，其目标如下：

> 控制器负责以简单和跨平台的方式控制性能分析基础结构和.NET性能诊断组件生成的性能数据。

我们的想法是[通过](http://mattwarren.org/2018/08/21/Monitoring-and-Observability-in-the-.NET-Runtime/(https://github.com/dotnet/designs/blob/master/accepted/performance-profiling-controller.md#functionality-exposed-through-controller))从“事件管道”中提取所有相关数据，通过[HTTP服务器](http://mattwarren.org/2018/08/21/Monitoring-and-Observability-in-the-.NET-Runtime/(https://github.com/dotnet/designs/blob/master/accepted/performance-profiling-controller.md#functionality-exposed-through-controller))公开[以下功能](http://mattwarren.org/2018/08/21/Monitoring-and-Observability-in-the-.NET-Runtime/(https://github.com/dotnet/designs/blob/master/accepted/performance-profiling-controller.md#functionality-exposed-through-controller))：

> **REST API**
> 
> *   Pri 1：简单分析：为运行时间配置X个时间并返回跟踪。
> *   Pri 1：高级分析：开始跟踪（以及配置）
> *   Pri 1：高级分析：停止跟踪（对此调用的响应将是跟踪本身）
> *   Pri 2：获取与所有EventCounters或指定EventCounter相关的统计信息。
> 
> **可浏览的HTML页面**
> 
> *   Pri 1：流程中所有托管代码堆栈的文本表示。
>    *   提供当前正在运行的用作简单诊断报告的快照概述。
>  *   Pri 2：显示EventCounters的当前状态（可能具有历史记录）。
>     *   提供现有计数器及其值的概述。
>     *   开放性问题：我不相信存在必要的公共API来枚举EventCounters。

我很高兴看到“**性能分析控制器(Performance Profiling Controller)**”（PPC？）的位置，我认为将这种内置到CLR中确实非常有价值，这是[其他运行时的内容](https://github.com/golang/go/wiki/Performance)。


## 剖析(Profiling)

CLR提供的另一个强大功能是[Profiling API](https://docs.microsoft.com/en-us/previous-versions/dotnet/netframework-4.0/ms404386(v%3dvs.100))，它（大部分）被第三方工具用于在非常低级别挂钩到运行时。您可以在[此概述中](https://docs.microsoft.com/en-us/previous-versions/dotnet/netframework-4.0/bb384493(v%3dvs.100))找到有关API的更多信息，但在较高级别，它允许您连接在以下情况下触发的回调：

*   GC相关事件发生
*   抛出异常
*   装配/卸载装配
*   [更多，更多](https://docs.microsoft.com/en-us/previous-versions/dotnet/netframework-4.0/ms230818%28v%3dvs.100%29)


![image.png](https://upload-images.jianshu.io/upload_images/5590759-e1e6f4dade78fa92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**来自BOTR页面的图像[分析API - 概述](https://github.com/dotnet/coreclr/blob/master/Documentation/botr/profiling.md#profiling-api--overview)**

此外还有其他**非常强大的功能**。首先，您可以**设置每次执行.NET方法时调用的挂钩，**无论是在运行时还是用户代码中。这些回调被称为“进入/离开”钩子，并且有一个[很好的示例](https://github.com/Microsoft/clr-samples/tree/master/ProfilingAPI/ReJITEnterLeaveHooks)显示如何使用它们，但为了使它们工作，您需要了解[不同操作系统和CPU架构的“调用约定”](https://github.com/dotnet/coreclr/issues/19023)，这[并不总是容易的](https://github.com/dotnet/coreclr/issues/18977)。另外，作为警告，Profiling API是一个只能通过C / C ++代码访问的COM组件，你不能在C＃/ F＃/ VB.NET中使用它！

其次，Profiler能够通过[SetILFunctionBody（）API](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilerfunctioncontrol-setilfunctionbody-method)在[JIT ](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilerfunctioncontrol-setilfunctionbody-method)**之前重写任何.NET方法的IL代码**。这个API功能非常强大，构成了许多.NET [APM工具](https://stackify.com/application-performance-management-tools/)的基础，您可以在我之前的文章中了解更多关于如何使用它的方法。[如何模拟密封类和静态方法](http://mattwarren.org/2014/08/14/how-to-mock-sealed-classes-and-static-methods/)以及[随附的代码](https://github.com/mattwarren/DDD2011_ProfilerDemo/commit/9f804cec8ef11b802e020e648180b436a429833f?w=1)。[](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilerfunctioncontrol-setilfunctionbody-method)[](https://stackify.com/application-performance-management-tools/)[](http://mattwarren.org/2014/08/14/how-to-mock-sealed-classes-and-static-methods/)[](https://github.com/mattwarren/DDD2011_ProfilerDemo/commit/9f804cec8ef11b802e020e648180b436a429833f?w=1)

### ICorProfiler API

事实证明，运行时必须执行各种疯狂的技巧才能使Profiling API正常工作，只需查看进入此PR的内容[允许重新连接](https://github.com/dotnet/coreclr/pull/19054)（有关'ReJIT'的详细信息，请参阅[ReJIT：A How-To指南](https://blogs.msdn.microsoft.com/davbr/2011/10/12/rejit-a-how-to-guide/)）。

所有Profiling API接口和回调的总体定义可在[\vm\inc\corprof.idl中找到](https://github.com/dotnet/coreclr/blob/master/src/inc/corprof.idl)（请参阅[接口说明语言](https://en.wikipedia.org/wiki/Interface_description_language)）。但它分为2个逻辑部分，一个是**Profiler - >'Execution Engine'（EE）**接口，称为`ICorProfilerInfo`：

```
// Declaration of class that implements the ICorProfilerInfo* interfaces, which allow the
// Profiler to communicate with the EE.  This allows the Profiler DLL to get
// access to private EE data structures and other things that should never be exported
// outside of the EE.

```

这在以下文件中实现：

*   [\VM\proftoeeinterfaceimpl.h](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/proftoeeinterfaceimpl.h)
*   [\VM\proftoeeinterfaceimpl.inl](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/proftoeeinterfaceimpl.inl)
*   [\VM\proftoeeinterfaceimpl.cpp](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/proftoeeinterfaceimpl.cpp)

另一个主要部分是**EE - > Profiler**回调，它们在`ICorProfilerCallback`界面下组合在一起：

```
// This module implements wrappers around calling the profiler's 
// ICorProfilerCallaback* interfaces. When code in the EE needs to call the
// profiler, it goes through EEToProfInterfaceImpl to do so.

```

这些回调在以下文件中实现：

*   [VM\eetoprofinterfaceimpl.h](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/eetoprofinterfaceimpl.h)
*   [VM\eetoprofinterfaceimpl.inl](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/eetoprofinterfaceimpl.inl)
*   [VM\eetoprofinterfaceimpl.cpp](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/eetoprofinterfaceimpl.cpp)
*   [VM\eetoprofinterfacewrapper.inl](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/eetoprofinterfacewrapper.inl)

最后，值得指出的是，Profiler API可能无法在.NET Core运行的所有操作系统和CPU-arch上运行，例如[Linux上的ELT调用存根问题](https://github.com/dotnet/coreclr/issues/18977)，有关详细信息，请参阅[CoreCLR Profiler API的状态](https://github.com/dotnet/coreclr/blob/release/2.1/Documentation/project-docs/profiling-api-status.md)。

### 分析和调试(Profiling v. Debugging)

除此之外，“分析”和“调试”确实有一些重叠，因此从[CLR调试与CLR分析](https://blogs.msdn.microsoft.com/jmstall/2004/10/22/clr-debugging-vs-clr-profiling/)*中了解.NET运行时上下文中*不同的API提供*了*什么是有帮助的。


![image.png](https://upload-images.jianshu.io/upload_images/5590759-5eb562e65f76707e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 调试(Debugging)

调试意味着不同的事情不同的人，比如我问在Twitter上“ *什么是你调试的.NET程序的途径* ”，并得到了[广泛](https://mobile.twitter.com/matthewwarren/status/1030444463385178113)的[不同反应](https://mobile.twitter.com/matthewwarren/status/1030580487969038344)，虽然反应两组含有一个很好的工具清单和技术，所以他们值得一试，谢谢#LazyWeb！

但也许这句话最好总结一下**Debugging究竟是**什么😊

![image.png](https://upload-images.jianshu.io/upload_images/5590759-abbcc4717f35ce03.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

CLR提供了与调试相关的非常广泛的功能，但为什么需要提供这些服务，优秀的帖子[为什么托管调试与本机调试不同？](https://blogs.msdn.microsoft.com/jmstall/2004/10/10/why-is-managed-debugging-different-than-native-debugging/)提供了3个理由：

1.  可以在硬件级别抽象本机调试，但**需要在IL级别抽象管理调试**
2.  托管调试需要大量的信息，**直到运行时才可用**
3.  托管调试器需要**与垃圾收集器（GC）协调**

所以给一个体面的经验，CLR *具有*提供[更高级别的调试API](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/debugging/)称`ICorDebug`，这将在下面从“常用的调试方案”的图像中显示[的BOTR](https://github.com/dotnet/coreclr/blob/master/Documentation/botr/dac-notes.md#marshaling-specifics)：


![image.png](https://upload-images.jianshu.io/upload_images/5590759-a322f6aae4d89b10.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

此外，还有很好的描述了不同部分[如何在管理断点如何工作中](https://blogs.msdn.microsoft.com/jmstall/2004/12/28/how-do-managed-breakpoints-work/)相互作用[？](https://blogs.msdn.microsoft.com/jmstall/2004/12/28/how-do-managed-breakpoints-work/)，虽然描述*左*和*右*是上图中的相反！

```
Here’s an overview of the pipeline of components:
1) End-user
2) Debugger (such as Visual Studio or MDbg).
3) CLR Debugging Services (which we call "The Right Side"). This is the implementation of ICorDebug (in mscordbi.dll).
---- process boundary between Debugger and Debuggee ----
4) CLR. This is mscorwks.dll. This contains the in-process portion of the debugging services (which we call "The Left Side") which communicates directly with the RS in stage #3.
5) Debuggee's code (such as end users C# program)

```

### ICorDebug API

但是如何实现所有这些以及从[CLR Debugging简要介绍](https://github.com/Microsoft/clrmd/blob/master/Documentation/GettingStarted.md#clr-debugging-a-brief-introduction)的不同组件是什么：

> 所有.Net调试支持都在我们称之为“The Dac”的dll之上实现。此文件（通常命名`mscordacwks.dll`）是我们的公共调试API（`ICorDebug`）以及两个私有调试API 的构建块：SOS-Dac API和IXCLR。
> 
> 在一个完美的世界中，每个人都会使用`ICorDebug`我们的公共调试API。但是，像您这样的工具开发人员所需的绝大多数功能都缺乏`ICorDebug`。这是我们正在修复的问题，但这些改进将进入CLR v.next，而不是旧版本的CLR。实际上，`ICorDebug`API仅在CLR v4中添加了对故障转储调试的支持。任何调试CLR v2崩溃转储的人根本无法使用`ICorDebug`！

（有关其他文章，请参阅[SOS和ICorDebug](https://github.com/dotnet/coreclr/blob/master/src/ToolBox/SOS/SOSAndICorDebug.md)）

该`ICorDebug`API实际上是分成多个接口，也有在他们的70！我不会在这里列出所有内容，但是我将展示它们所属的类别，有关更多信息，请参阅[ICorDebug的分区，](https://blogs.msdn.microsoft.com/jmstall/2006/01/04/partition-of-icordebug/)其中包含此列表，因为它更详细。

*   **顶级(Debugging)：** ICorDebug + ICorDebug2是顶级接口，有效地充当ICorDebugProcess对象的集合。
*   **回调(Callbacks)：**通过调试器实现的回调对象上的方法调度托管调试事件
*   **进程(Process)：**这组接口表示正在运行的代码，并包含与事件相关的API。
*   **代码/类型检查(Code / Type Inspection)：** 主要可以在静态PE映像上运行，但实时数据有一些便​​捷方法。
*   **执行控制(Execution Control)：**执行是“检查”线程执行的能力。实际上，这意味着放置断点（F9）和踩踏（F11步入，F10步进，S + F11步出）等。ICorDebug的执行控制仅在托管代码中运行。
*   **线程+调用堆栈(Threads + Callstacks)：**调用堆栈是调试器检查功能的支柱。以下接口与获取callstack有关。ICorDebug仅公开调试托管代码，因此堆栈跟踪仅受管理。
*   **对象检查(Object Inspection)：**对象检查是API的一部分，它允许您在整个调试对象中查看变量的值。对于每个接口，我列出了“MVP”方法，我认为必须简洁地传达该接口的用途。

另外需要注意的是，与Profiling APIs一样，调试API的支持级别因操作系统和CPU架构而异。例如，截至2018年8月，[“没有针对Linux ARM进行托管调试和诊断的解决方案”](https://github.com/dotnet/diagnostics/issues/58#issuecomment-414182115)。有关“Linux”支持的更多信息，请参阅这篇很棒的文章，[在Linux上使用LLDB调试.NET Core，](https://www.raydbg.com/2018/Debugging-Net-Core-on-Linux-with-LLDB/)并从Microsoft 检出[诊断存储库](https://github.com/dotnet/diagnostics)，其目标是更容易在Linux上调试.NET程序。

最后，如果你想看看`ICorDebug`API在C＃中的样子，看一下[CLRMD库中包含](https://github.com/Microsoft/clrmd/blob/master/src/Microsoft.Diagnostics.Runtime/ICorDebug/ICorDebugWrappers.cs)的[包装器](https://github.com/Microsoft/clrmd/blob/master/src/Microsoft.Diagnostics.Runtime/ICorDebug/ICorDebugWrappers.cs)，包括所有[可用的回调](https://github.com/Microsoft/clrmd/blob/c81a592f3041a9ae86f4c09351d8183801e39eed/src/Microsoft.Diagnostics.Runtime/ICorDebug/ICorDebugHelpers.cs)（CLRMD将在后面的文章中进行更深入的介绍）。

### SOS和DAC

“数据访问组件(Data Access Component)”（DAC）在[BOTR页面](https://github.com/dotnet/coreclr/blob/master/Documentation/botr/dac-notes.md)中有详细讨论，但实际上它提供了对CLR数据结构的“进程外”访问，因此可以从*另一个进程*读取其内部详细信息。这允许调试器（via `ICorDebug`）或['Son of Strike'（SOS）扩展](https://docs.microsoft.com/en-us/dotnet/framework/tools/sos-dll-sos-debugging-extension)进入CLR的运行实例或内存转储，并找到如下内容：

*   所有正在运行的线程
*   托管堆上有哪些对象
*   有关方法的完整信息，包括机器代码
*   当前的'堆栈跟踪'

**除此之外**，如果您想要解释所有奇怪的名称和一点'.NET历史课'，请参阅[此Stack Overflow答案](https://stackoverflow.com/questions/21361602/what-the-ee-means-in-sos/21363245#21363245)。

[SOS命令](https://github.com/dotnet/coreclr/blob/master/Documentation/building/debugging-instructions.md#sos-commands)的完整列表非常令人印象深刻，并且在WinDBG旁边使用它可以让您非常低级地了解程序和CLR中发生的情况。要了解它是如何实现的，让我们看一下这个`!HeapStat`命令，该命令可以为您提供.NET GC正在使用的不同堆大小的摘要：

![image.png](https://upload-images.jianshu.io/upload_images/5590759-9a6b9693acb576d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

（来自[SOS的](https://blogs.msdn.microsoft.com/tom/2008/06/30/sos-upcoming-release-has-a-few-new-commands-heapstat/)图片[：即将发布的版本有一些新命令 - HeapStat](https://blogs.msdn.microsoft.com/tom/2008/06/30/sos-upcoming-release-has-a-few-new-commands-heapstat/)）

这是代码流，显示了SOS和DAC如何协同工作：

*   **SOS**完整`!HeapStat`命令（[链接](https://github.com/dotnet/coreclr/blob/release/2.1/src/ToolBox/SOS/Strike/strike.cpp#L4605-L4782)）
*   **SOS**`!HeapStat`处理'Workstation GC' 的命令中的代码（[链接](https://github.com/dotnet/coreclr/blob/release/2.1/src/ToolBox/SOS/Strike/strike.cpp#L4631-L4667)）
*   **SOS** `GCHeapUsageStats(..)`功能，重负荷（[链接](https://github.com/dotnet/coreclr/blob/release/2.1/src/ToolBox/SOS/Strike/eeheap.cpp#L768-L850)）
*   **共享**`DacpGcHeapDetails`包含指向GC堆中主数据的指针的数据结构，例如段，卡表和各代（[链接](https://github.com/dotnet/coreclr/blob/release/2.1/src/inc/dacprivate.h#L690-L722)）
*   `GetGCHeapStaticData`填充`DacpGcHeapDetails`结构的**DAC**函数（[链接](https://github.com/dotnet/coreclr/blob/release/2.1/src/inc/dacprivate.h#L690-L722)）
*   **共享**`DacpHeapSegmentData`包含GC堆的单个“段”的详细信息的数据结构（[链接](https://github.com/dotnet/coreclr/blob/release/2.1/src/inc/dacprivate.h#L738-L771)）
*   `GetHeapSegmentData(..)`填充`DacpHeapSegmentData`结构的**DAC**（[链接](https://github.com/dotnet/coreclr/blob/release/2.1/src/debug/daccess/request.cpp#L2829-L2868)）


### 第三方'调试器'(3rd Party ‘Debuggers’)

由于Microsoft发布了调试API，它允许第三方使用`ICorDebug`接口，这里列出了我遇到的一些内容：

*   [调试器.NET Core运行时](https://github.com/Samsung/netcoredbg)来自[三星](https://github.com/Samsung)
    *   调试器提供GDB / MI或VSCode调试适配器接口，并允许在.NET Core运行时下调试.NET应用程序。
    *   *可能*是他们[将.NET Core移植到他们的Tizen OS](https://developer.tizen.org/blog/celebrating-.net-core-2.0-looking-forward-tizen-4.0)的工作的一部分[](https://developer.tizen.org/blog/celebrating-.net-core-2.0-looking-forward-tizen-4.0)
*   [dnSpy](https://github.com/0xd4d/dnSpy) - “.NET调试器和汇编编辑器”
    *   一个[**非常**令人印象深刻的工具](https://github.com/0xd4d/dnSpy#features-see-below-for-more-detail)，它是一个'调试器'，'汇编编辑器'，'十六进制编辑器'，'反编译器'等等！
*   [MDbg.exe（.NET Framework命令行调试程序）](https://docs.microsoft.com/en-us/dotnet/framework/tools/mdbg-exe)
    *   可以作为[NuGet包](https://www.nuget.org/packages/Microsoft.Samples.Debugging.MdbgEngine)和[GitHub存储库使用](https://github.com/SymbolSource/Microsoft.Samples.Debugging/tree/master/src)，也可以[从Microsoft下载](https://www.microsoft.com/en-us/download/details.aspx?id=2282)。
    *   但是，目前MDBG似乎不适用于.NET Core，请参阅[端口MDBG到CoreCLR](https://github.com/dotnet/coreclr/issues/1145)和[ETA以将mdbg移植到coreclr](https://github.com/dotnet/coreclr/issues/8999)以获取更多信息。
*   [JetBrains'Rider'](https://blog.jetbrains.com/dotnet/2017/02/23/rider-eap-18-coreclr-debugging-back-windows/)允许在Windows上进行.NET Core调试
    *   虽然由于许可问题引起了[一些争议](https://blog.jetbrains.com/dotnet/2017/02/15/rider-eap-17-nuget-unit-testing-build-debugging/)
    *   有关更多信息，请参阅[此HackerNews主题](https://news.ycombinator.com/item?id=17323911)

### 记忆转储(Memory Dumps)

我们要看的最后一个区域是“内存转储”，可以从*实时*系统中捕获并离线分析。.NET运行时一直很好地支持[在Windows上创建“内存转储”](https://msdn.microsoft.com/en-us/library/dn342825.aspx?f=255&MSPPError=-2147217396#BKMK_Collect_memory_snapshots)，现在.NET Core是“跨平台”，也可以[在其他操作系统上使用相同的](https://github.com/dotnet/coreclr/blob/master/Documentation/botr/xplat-minidump-generation.md)工具。

“内存转储”的一个问题是，获取SOS和DAC文件的正确匹配版本可能会非常棘手。幸运的是，Microsoft刚刚发布了以下[`dotnet symbol`CLI工具](https://github.com/dotnet/symstore/tree/master/src/dotnet-symbol)：

> 可以下载任何给定核心转储，minidump或任何支持平台的文件格式（如ELF，MachO，Windows DLL，PDB和便携式PDB）的调试所需的所有文件（给出coreclr模块的符号，模块，SOS和DAC）。

最后，如果你花费任何时间**分析'内存转储'，**你真的应该看看微软几年前发布的优秀的[CLR MD库](https://github.com/Microsoft/clrmd)。我之前[已经写过](http://mattwarren.org/2016/09/06/Analysing-.NET-Memory-Dumps-with-CLR-MD/)你可以用它做什么，但简而言之，它允许你通过一个直观的C＃API与内存转储交互，其中的类可以访问[ClrHeap](https://github.com/Microsoft/clrmd/blob/master/src/Microsoft.Diagnostics.Runtime/ClrHeap.cs#L16)，[GC Roots](https://github.com/Microsoft/clrmd/blob/6735e1012d11c244874fa3ba3af6e73edc0da552/src/Microsoft.Diagnostics.Runtime/GCRoot.cs#L105)，[CLR Threads](https://github.com/Microsoft/clrmd/blob/master/src/Microsoft.Diagnostics.Runtime/ClrThread.cs#L103)，[Stack Frames](https://github.com/Microsoft/clrmd/blob/master/src/Microsoft.Diagnostics.Runtime/ClrThread.cs#L37)和[更多](https://github.com/Microsoft/clrmd/tree/master/src/Samples)。实际上，除了实现工作所需的时间之外，CLR MD还可以[实现*大多数*（如果不是全部）SOS命令](https://github.com/Microsoft/clrmd/issues/33)。

但是从[宣布帖子](https://blogs.msdn.microsoft.com/dotnet/2013/05/01/net-crash-dump-and-live-process-inspection/)来看它是如何运作[的](https://blogs.msdn.microsoft.com/dotnet/2013/05/01/net-crash-dump-and-live-process-inspection/)：

> ClrMD托管库是CLR仅内部调试API的包装器。虽然这些仅内部API对于诊断非常有用，但我们不支持它们作为公开的，有文档的版本，因为它们非常难以使用并且与CLR的其他实现细节紧密耦合。ClrMD通过围绕这些低级调试API提供易于使用的托管包装来解决此问题。

通过在官方支持的库中提供这些API，Microsoft使开发人员能够在CLRMD之上构建[各种工具](http://mattwarren.org/2018/06/15/Tools-for-Exploring-.NET-Internals/#tools-based-on-clr-memory-diagnostics-clrmd)，这是一个很好的结果！

* * *

**总而言之，.NET Runtime提供了广泛的诊断，调试和分析功能，可以深入了解CLR内部的情况。**
