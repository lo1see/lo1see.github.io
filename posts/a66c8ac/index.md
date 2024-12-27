# Win Kernel学习——DPC机制


&lt;!--more--&gt;



# DPC机制

## 前言

**x86架构设计在上是基于中断思想的，因而从DOS到Win32，操作系统中大量使用中断的概念来表达异步操作的
行为。但与DOS下独占的情况不同，Win32下需要由系统对多任务进行调度，因此中断响应代码必须尽可能地简单，并且尽快的将控制权交还给系统。虽然这样一来系统调度的响应速度和实现过程方便了，但还是有很多功能需要在中断响应中完成。为此，Win32核心提供了`DPC(Deferred Procedure Call)`和`APC(Asynchronous Procedure Call)`两个`IRQL`特殊的软件中断级别，用于实现延迟和异步的过程调用。**



`APC`：即时过程调用机制

`DPC`：延时过程调用机制

`DPC`的栈是独立的，`DPC`的栈空间与线程的栈空间相互独立

`DPC`的栈空间会被替换为堆分页内存中的一段3KB大小的空间

从`IRQL`分层来说，`DPC`和`APC`是介于较高级别的设备中断最低级别的`Passive`中断之间，由操作系统用于完成特殊方法调用的中断级别。与处理硬件操作的设备中断和更高级别的时钟、处理器中断不同，这两级中断纯粹是为了实现功能调用异步性而设计实现的，因此操作系统本身也对它们具有很强的依赖型。

`DPC`在功能上可以理解为`中断服务例程 ISR(Interrupt Service Routine)`的一部分。为每次触发中断，都会关中断，然后执行中断服务例程。由于关中断了，所以`中断服务例程`必须短小精悍，不能消耗过多时间，否则会导致系统丢失大量其他中断。但是有的中断，其`中断服务例程`要做的事情本来就很多，那怎么办？于是，可以在`中断服务例程`中先执行最紧迫的那部分工作，然后把剩余的相对来说不那么重要的工作移入到`DPC`函数中去执行。因此，`DPC`又叫`ISR`的后半部。（比如每次时钟中断后，其`isr`会扫描系统中的所有定时器是否到点，若到点就调用各定时器的函数。但是这个扫描过程比较耗时，因此，时钟中断的`isr`会将扫描工作纳入`dpc`中进行）

每当触发一个中断时，`中断服务例程`可以在当前`cpu`中插入一个`DPC`，当执行完`isr`，退出`isr`后， `cpu`就会扫描它的`dpc`队列，依次执行里面的每个`dpc`，当执行完`dpc`后，才又回到当前线程的中断处继续执行。

## 结构体：

```c
typedef struct _KDPC {
    union {
        ULONG TargetInfoAsUlong;
        struct {
            UCHAR Type;
            UCHAR Importance;
            volatile USHORT Number;
        } DUMMYSTRUCTNAME;
    } DUMMYUNIONNAME;

    SINGLE_LIST_ENTRY DpcListEntry;
    KAFFINITY ProcessorHistory;
    PKDEFERRED_ROUTINE DeferredRoutine;
    PVOID DeferredContext;
    PVOID SystemArgument1;
    PVOID SystemArgument2;
    __volatile PVOID DpcData;
} KDPC, *PKDPC, *PRKDPC;
```

## 开发注意

`DPC`一定要申请`NonPagedPool`非分页内存。`DPC`内部是运行在`Dispatch Level (= 2)`的等级下的，而异常也是运行在`Dispatch Level`等级下的。如果缺页异常`Page Fault Exception`发生在`DPC`的内部时，会发生异常处理无法打断`DPC`的执行，无法修复缺页异常。所以访问分页内存时有概率蓝屏。

## 插入DPC队列

```c
VOID DpcRoutineCallBack(
    _In_ struct _KDPC* Dpc,
    _In_opt_ PVOID DeferredContext,
    _In_opt_ PVOID SystemArgument1,
    _In_opt_ PVOID SystemArgument2 ) {
    
    ULONG cpuNumber = KeGetCurrentProcessorNumberEx(NULL);
    DbgPrint(&#34;[lo1see] %s success!! cpuNumber: %d\n&#34;, __FUNCTION__, cpuNumber);

    //释放DPC内存 插入DPC队列和设置定时器延时触发
    ExFreePool(Dpc);
}
  


//DriverEntry：
    PKDPC kDpc = (PKDPC)ExAllocatePool(NonPagedPool, sizeof(KDPC));

    //插入DPC队列
    KeInitializeDpc(kDpc, DpcRoutineCallBack, NULL);
    //设置目标CPU核心
    KeSetTargetProcessorDpc(kDpc, (cpuNumber &#43; 2) % 4);
    KeInsertQueueDpc(kDpc, NULL, NULL);
```

## 设置定时器延时触发

```c
VOID DpcRoutineCallBack(
    _In_ struct _KDPC* Dpc,
    _In_opt_ PVOID DeferredContext,
    _In_opt_ PVOID SystemArgument1,
    _In_opt_ PVOID SystemArgument2 ) {

    ULONG cpuNumber = KeGetCurrentProcessorNumberEx(NULL);
    DbgPrint(&#34;[lo1see] %s success!! cpuNumber: %d\n&#34;, __FUNCTION__, cpuNumber);

    //释放DPC内存 插入DPC队列和设置定时器延时触发，如果设置定时器循环触发不要释放内存
    ExFreePool(Dpc);
}



//DriverEntry：
    PKDPC kDpc = (PKDPC)ExAllocatePool(NonPagedPool, sizeof(KDPC));
    //设置定时器延时触发
    KeInitializeDpc(kDpc, DpcRoutineCallBack, NULL);
    ULONG cpuNumber = KeGetCurrentProcessorNumberEx(NULL);
    PKTIMER kTimer = ExAllocatePool(NonPagedPool, sizeof(KTIMER));
    KeInitializeTimer(kTimer);
    LARGE_INTEGER inTime = { 0 };
    //三秒后执行
    inTime.QuadPart = -10000 * 3000;
	//下面两个二选一都可
    //KeSetTimer(kTimer, inTime, kDpc);
	//3秒钟后每秒触发一次
    KeSetTimerEx(kTimer, inTime, 1000, kDpc);
```

## 多核侵染

未公开的相关导出函数

```c
VOID
KeGenericCallDpc(
    __in PKDEFERRED_ROUTINE Routine,
    __in_opt PVOID Context
);		//每个CPU核心都调用一次

VOID
KeSignalCallDpcDone(
    __in PVOID SystemArgument1
);		//通知调用DPC完毕

LOGICAL
KeSignalCallDpcSynchronize(
    __in PVOID SystemArgument2
);		//通知调用DPC完毕
```

代码：

```c
VOID DpcRoutineCallBack(
    _In_ struct _KDPC* Dpc,
    _In_opt_ PVOID DeferredContext,
    _In_opt_ PVOID SystemArgument1,
    _In_opt_ PVOID SystemArgument2 ) {

    ULONG cpuNumber = KeGetCurrentProcessorNumberEx(NULL);
    DbgPrint(&#34;[lo1see] %s success!! cpuNumber: %d\n&#34;, __FUNCTION__, cpuNumber);

    //多核侵染，通知DPC调用已完成
    KeSignalCallDpcDone(SystemArgument1);
    KeSignalCallDpcSynchronize(SystemArgument2);
}



//DriverEntry：
    //多核侵染
	KeGenericCallDpc(DpcRoutineCallBack, NULL);
```



## 队列执行

更隐蔽的一种方式，利用线程执行代码会留下很多痕迹，但是利用队列执行不会。系统中专门有几个系统线程在等待队列链表执行，代码会插入到这个结构体中进行执行

```c
typedef struct _WORK_QUEUE_ITEM {
    LIST_ENTRY List;						//链表
    PWORKER_THREAD_ROUTINE WorkerRoutine;
    __volatile PVOID Parameter;
} WORK_QUEUE_ITEM, *PWORK_QUEUE_ITEM;
```

枚举参数：

```c
typedef _Enum_is_bitflag_ enum _WORK_QUEUE_TYPE {
    CriticalWorkQueue,			//立即执行
    DelayedWorkQueue,			//延迟执行
    HyperCriticalWorkQueue,
    NormalWorkQueue,
    BackgroundWorkQueue,
    RealTimeWorkQueue,
    SuperCriticalWorkQueue,
    MaximumWorkQueue,
    CustomPriorityWorkQueue = 32
} WORK_QUEUE_TYPE;
```

队列执行

```c
VOID WorkItemCallBack(
    _In_ PVOID Parameter
) {
    LARGE_INTEGER inTime = { 0 };
    //三秒后执行
    inTime.QuadPart = -10000 * 3000;
    KeDelayExecutionThread(KernelMode, FALSE, &amp;inTime);

    ULONG cpuNumber = KeGetCurrentProcessorNumberEx(NULL);
    DbgPrint(&#34;[lo1see] %s success!! cpuNumber: %d\n&#34;, __FUNCTION__, cpuNumber);
}
    
    

//DriverEntry：
    //队列执行代码
    PWORK_QUEUE_ITEM pitem = ExAllocatePool(PagedPool, sizeof(WORK_QUEUE_ITEM));
    ExInitializeWorkItem(pitem, WorkItemCallBack, NULL);
    ExQueueWorkItem(pitem, CriticalWorkQueue);
```





## 完整示例代码

```c
#include &lt;ntifs.h&gt;

VOID
KeGenericCallDpc(
    __in PKDEFERRED_ROUTINE Routine,
    __in_opt PVOID Context
);

VOID
KeSignalCallDpcDone(
    __in PVOID SystemArgument1
);

LOGICAL
KeSignalCallDpcSynchronize(
    __in PVOID SystemArgument2
);



VOID DpcRoutineCallBack(
    _In_ struct _KDPC* Dpc,
    _In_opt_ PVOID DeferredContext,
    _In_opt_ PVOID SystemArgument1,
    _In_opt_ PVOID SystemArgument2 ) {

    //DbgBreakPoint();
    ULONG cpuNumber = KeGetCurrentProcessorNumberEx(NULL);
    DbgPrint(&#34;[lo1see] %s success!! cpuNumber: %d\n&#34;, __FUNCTION__, cpuNumber);

    //释放DPC内存 插入DPC队列和设置定时器延时触发
    ExFreePool(Dpc);

    //多核侵染
    //KeSignalCallDpcDone(SystemArgument1);
    //KeSignalCallDpcSynchronize(SystemArgument2);
}

VOID WorkItemCallBack(
    _In_ PVOID Parameter
) {
    LARGE_INTEGER inTime = { 0 };
    //三秒后执行
    inTime.QuadPart = -10000 * 3000;
    KeDelayExecutionThread(KernelMode, FALSE, &amp;inTime);

    ULONG cpuNumber = KeGetCurrentProcessorNumberEx(NULL);
    DbgPrint(&#34;[lo1see] %s success!! cpuNumber: %d\n&#34;, __FUNCTION__, cpuNumber);
}

VOID DriverUnload(_In_ PDRIVER_OBJECT DriverObject)
{
    UNREFERENCED_PARAMETER(DriverObject);
    DbgPrint(&#34;[lo1see] %s success!!\tHello World Driver Unload\n&#34;, __FUNCTION__);
}


NTSTATUS DriverEntry(_In_ PDRIVER_OBJECT DriverObject, _In_ PUNICODE_STRING RegistryPath)
{
    UNREFERENCED_PARAMETER(RegistryPath);
    UNREFERENCED_PARAMETER(DriverObject);

    DbgPrint(&#34;[lo1see] %s success!!\tHello World Driver Loaded\n&#34;, __FUNCTION__);
    // 设置驱动程序卸载例程
    DriverObject-&gt;DriverUnload = DriverUnload;

    ULONG cpuNumber = KeGetCurrentProcessorNumberEx(NULL);
    DbgPrint(&#34;[lo1see] %s  cpuNumber: %d\n&#34;, __FUNCTION__, cpuNumber);

    PKDPC kDpc = (PKDPC)ExAllocatePool(NonPagedPool, sizeof(KDPC));

    //插入DPC队列
    KeInitializeDpc(kDpc, DpcRoutineCallBack, NULL);
    KeSetTargetProcessorDpc(kDpc, (cpuNumber &#43; 2) % 4);
    KeInsertQueueDpc(kDpc, NULL, NULL);
    
    //设置定时器延时触发
    //KeInitializeDpc(kDpc, DpcRoutineCallBack, NULL);
    //ULONG cpuNumber = KeGetCurrentProcessorNumberEx(NULL);
    //PKTIMER kTimer = ExAllocatePool(NonPagedPool, sizeof(KTIMER));
    //KeInitializeTimer(kTimer);
    //LARGE_INTEGER inTime = { 0 };
    ////三秒后执行
    //inTime.QuadPart = -10000 * 3000;
    ////KeSetTimer(kTimer, inTime, kDpc);
    //KeSetTimerEx(kTimer, inTime, 1000, kDpc);


    //多核侵染
    //KeGenericCallDpc(DpcRoutineCallBack, NULL);


    //队列执行代码
    PWORK_QUEUE_ITEM pitem = ExAllocatePool(PagedPool, sizeof(WORK_QUEUE_ITEM));
    ExInitializeWorkItem(pitem, WorkItemCallBack, NULL);
    ExQueueWorkItem(pitem, CriticalWorkQueue);


    return STATUS_SUCCESS;
}
```

















---

> 作者:   
> URL: http://localhost:1313/posts/a66c8ac/  

