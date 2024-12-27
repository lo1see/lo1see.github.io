# Win Kernel学习——ETW机制


&lt;!--more--&gt;



# ETW机制



## 概述

### 什么是`ETW`？

`ETW`是在`Windows 2000`中引入的，`Microsoft`需要这样的机制，因为`Windows`开始变得越来越复杂，越来越多的东西在操作系统中运行，所以在`NT4`的日子里，很难知道发生了什么。例如，线程上下文切换需要多长时间；什么消耗了大量的`CPU`时间；为什么这个线程在等待，它在等待什么？为什么需要这么长的时间？
因此，`Microsoft`决定在`Windows 2000`中创建了一个时间跟踪和日志记录机制。它开销低，性能极高，本质上意味着，即使`ETW`每秒生成数千个事件，它对整个操作系统的性能的影响应该可以忽略不计。

`ETW`是`Event Tracing for Windows`的简称，它是`Windows`提供的原生的事件跟踪日志系统。由于采用内核`（Kernel）`层面的缓冲和日志记录机制

### 机制

通过`ETW`获取的信息非常丰富，可以通过`ETW`获取到的信息有：

1. 文件类信息，包括文件创建、删除、读写等信息。
2. 注册表信息，包括注册表的创建、删除、读写等信息。
3. 进程线程信息，包括进程创建退出、线程创建退出、模块加载等。
4. 网络信息，`TCP、UDP`协议的发送，接收`ip`地址以及数据长度等。
5. `CPU`的使用情况、内存使用情况以及发生事件时的堆栈信息等。

更多的信息获取参见`MSDN`中[EnableFlags](https://msdn.microsoft.com/en-us/library/windows/desktop/aa363784(v=vs.85).aspx)选项介绍。

由于`ETW`获取的信息非常丰富，且对系统性能影响不是很大，因此用来作为一种系统监控手段非常合适。

`Event Trace for Windows (ETW)` 是一个高效的内核级别的事件追踪机制，它可以记录系统内核或是应用程序的事件到日志文件。我们可以通过实时获取或是从一个日志文件来解析处理这些事件，之后通过这些事件用来调试程序或是找到程序的性能问题。

`ETW`主要由三部分组成，分别是：

1. `Controllers（事件控制器）`，用来开关`event trace `会话 和 `Providers`。
2. `Providers（事件提供器）`， 用来提供事件。
3. `Consumers（事件消耗器）`，用来处理事件。

另有一个关键的概念`Sessions`。

这四者的关系图如下图所示：

![13](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/19327-20160914082309273-200274130.jpg)

整个`ETW`系统由`Provider，Customer，Controller`三个部分构成： 

#### Provider

所谓的`Provider`，就是事件的提供者，它可以是系统组件，驱动程序或者是我们开发的应用程序。首先，它需要向系统进行注册一个`Event Trace`，然后当这个`Provider`被`Controller`启动`(Enable)`后，它就可以开始向相应的`Event Trace Session`发送事件了。 

#### Controller

顾名思义，`Controller`就是一个控制器。它的主要任务有两个：一是`Event Trace Session`的控制管理。它利用`StartTrace`在内存中创建一个`Event Trace session`，这样`Provider`就知道该往哪里发生事件。而`Controller`也会负责将`Session`里记录的事件送到`Consumer`。`Controller`的第二个任务就是对`Provider`进行管理，启动或是停止`Provider`。为了避免额外的开销，`Provider`不会一直都在工作，只有当被`Enable`的时候，才开始工作。 

#### Consumer

`Consumer`实时地从`Event Trace Session`或者是日志文件中订阅事件。`Consumer`主要的作用是提供`Event Trace Callback`。我们可以设计一个通用的`Callback`来处理所有的事件，也可以为特定的我们感兴趣的事件设计`callback`。对于通用事件的`callback`，我们可以在`OpenTrace`的时候通过参数指定，而对于特定时间的`callback`，则可以通过`SetTraceCallback`指定。

#### Sessions

除了上述三个组件外，还有一个概念也较为重要，这便是`Session`。`Session`由`Controllers`定义，`Session`记录了一个或多个`Providers`输出的事件，其主要用来管理和刷新事件的缓存。`ETW Session`提供了一个接收、存储、处理和分发事件的执行上下文。

系统同时最多支持64个`Session`，这些Session中有两个`Session`比较特殊，由操作系统直接定义，可直接使用，分别是：

1. `Global Logger Session`, 它用来记录操作系统早期启动过程中的事件，例如设备驱动相关的事件。
2. `NT Kernel Logger Session`，它用来记录操作系统生成的预定义系统事件，例如磁盘IO或页面错误事件。





## 结构体

关键部分

```c
0: kd&gt; dt _WMI_LOGGER_CONTEXT
nt!_WMI_LOGGER_CONTEXT
   &#43;0x000 LoggerId         : Uint4B
   &#43;0x004 BufferSize       : Uint4B
   &#43;0x008 MaximumEventSize : Uint4B
   &#43;0x00c CollectionOn     : Int4B
   &#43;0x010 LoggerMode       : Uint4B
   &#43;0x014 AcceptNewEvents  : Int4B
   // win10上是&#43;0x28，与事件记录相关的函数地址，当产生日志时会调用一次这里的函数
   &#43;0x018 GetCpuClock      : Ptr64     int64 
   &#43;0x020 StartTime        : _LARGE_INTEGER
   // .......
```





## 调用分析和`Hook`思路

这里是学习的老版本的`hook`思路，有些地方已经被`PG`保护了，不能适用在新版本的`Windows`系统上了

在`KiSystemCall64`中的日志记录功能部分

![image-20241226192216328](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241226192216328.png)

在`KiSystemServiceCopyEnd`中判断标志位，查看是否开启了`ETW`，这里是针对系统调用的日志记录功能。若开启则跳转到日志记录部分稍后进行系统调用，未开启则直接`call r10`进行系统调用



![image-20241226193434090](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241226193434090.png)

找到这个变量，这个变量就是存储的`_WMI_LOGGER_CONTEXT`结构体，对`GetCpuClock`成员进行修改，修改为自己的回调函数

![image-20241226203649202](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241226203649202.png)



![image-20241226194543896](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241226194543896.png)

开启日志记录功能之后，可以发现暂时将系统调用的函数地址暂存到了栈空间中，在进行日志记录后再从栈空间中读取到`r10`寄存器中，再进行调用



![image-20241226195035750](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241226195035750.png)

在`PerfInfoLogSysCallEntry`函数中的`EtwpTraceKernelEvent`函数参数中有很多的常数，作为参数时会被推入栈中，这些常数可以作为特征码进行搜索



![image-20241226195421847](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241226195421847.png)

经过`PerfInfoLogSysCallEntry -&gt; EtwpTraceKernelEvent -&gt; EtwpLogKernelEvent -&gt; EtwpReserveTraceBuffer -&gt; WmipLoggerContext.GetCpuClock`的调用流程来到了`GetCpuClock`指定的函数。这里的函数已经在前面被修改成自定义的函数内容了，这里实现对堆栈的搜索，找到前面在堆栈中存储的系统调用函数地址，将其修改，就可以实现`ETW Hook`了





## 参考`ETW Hook`项目

https://github.com/everdox/InfinityHook

https://github.com/Oxygen1a1/InfinityHook_latest

https://github.com/mapozyan/etw_hook


---

> 作者:   
> URL: http://localhost:1313/posts/83055c4/  

