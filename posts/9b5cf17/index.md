# Win Kernel学习——APC机制


&lt;!--more--&gt;



# Windows内核学习——APC机制



## APC相关结构体

### `_KTHREAD`

详细的结构体看这个网站吧，这里只写了相关的几个成员

https://www.vergiliusproject.com/kernels/x86/windows-7/sp1/_KTHREAD

```c
3: kd&gt; dt _KTHREAD
ntdll!_KTHREAD
	//......
	//此线程是否能执行APC；设为0则关闭，不能执行APC，但是永久设为0会使线程卡死，只能暂时设为0
    &#43;0x03c ApcInterruptRequest : Pos 8, 1 Bit
    //......
    &#43;0x040 ApcState         : _KAPC_STATE
    &#43;0x040 ApcStateFill     : [23] UChar
    //......
    //APC队列锁，插入、删除节点的时候会加锁/释放锁
    &#43;0x060 ApcQueueLock     : Uint4B
    //......
    //关闭普通内核APC，如果非0则关闭，0则开启
    &#43;0x084 KernelApcDisable : Int2B
    //关闭紧急内核APC，如果非0则关闭，0则开启
    &#43;0x086 SpecialApcDisable : Int2B
    //Combined包含上面两个，赋值-1则以上两个全部关闭
    &#43;0x084 CombinedApcDisable : Uint4B
    //......
    //是否有APC队列，置1则为有APC队列，0则无APC队列
    &#43;0x0b8 ApcQueueable     : Pos 5, 1 Bit
    //......
    //线程是否挂靠，索引；为1则挂靠，表示在养父进程上；为0则未挂靠，在亲生进程上
    &#43;0x134 ApcStateIndex    : UChar
    //......
    //重要 指向_KAPC_STATE结构
    &#43;0x168 ApcStatePointer  : [2] Ptr32 _KAPC_STATE
    //1表示挂靠，0表示没有挂靠
    &#43;0x170 SavedApcState    : _KAPC_STATE
    &#43;0x170 SavedApcStateFill : [23] UChar
    //......

```

### `_KAPC_STATE`

`_KTHREAD`的`&#43;0x040 ApcState         : _KAPC_STATE`和`&#43;0x168 ApcStatePointer  : [2] Ptr32 _KAPC_STATE`指向这个结构体

```c
3: kd&gt; dt _KAPC_STATE
ntdll!_KAPC_STATE
   //这个队列有两个链表，0代表内核APC队列，1代表用户APC队列
   &#43;0x000 ApcListHead      : [2] _LIST_ENTRY
   //当前进程对象
   &#43;0x010 Process          : Ptr32 _KPROCESS
   //是否有内核APC在执行
   &#43;0x014 KernelApcInProgress : UChar
   //内核APC队列中是否有元素
   &#43;0x015 KernelApcPending : UChar
   //用户APC队列中是否有元素
   &#43;0x016 UserApcPending   : UChar
```



### `_KAPC`

`_KAPC_STATE`的`&#43;0x000 ApcListHead      : [2] _LIST_ENTRY`指向`_KAPC`结构的`&#43;0x00c ApcListEntry     : _LIST_ENTRY`节点

```
3: kd&gt; dt _KAPC
ntdll!_KAPC
   &#43;0x000 Type             : UChar
   &#43;0x001 SpareByte0       : UChar
   &#43;0x002 Size             : UChar
   &#43;0x003 SpareByte1       : UChar
   &#43;0x004 SpareLong0       : Uint4B
   //指向线程对象，表示是哪个线程的APC
   &#43;0x008 Thread           : Ptr32 _KTHREAD
   //_KAPC_STATE的ApcListHead指向这个元素
   &#43;0x00c ApcListEntry     : _LIST_ENTRY
   //函数以及函数的参数
   &#43;0x014 KernelRoutine    : Ptr32     void 
   &#43;0x018 RundownRoutine   : Ptr32     void 
   &#43;0x01c NormalRoutine    : Ptr32     void 
   &#43;0x020 NormalContext    : Ptr32 Void
   &#43;0x024 SystemArgument1  : Ptr32 Void
   &#43;0x028 SystemArgument2  : Ptr32 Void
   //APC是否挂靠，索引；为1则挂靠，表示在养父进程上；为0则未挂靠，在亲生进程上
   &#43;0x02c ApcStateIndex    : Char
   //描述是内核态APC还是用户态APC
   &#43;0x02d ApcMode          : Char
   //表示这个APC元素是否插入到了APC队列中，已插入则为1
   &#43;0x02e Inserted         : UChar
```

`&#43;0x168 ApcStatePointer  : [2]`其中一个指向`&#43;0x040 ApcState`，另一个指向`&#43;0x170 SavedApcState` 

一个线程要挂靠到其他进程上，之前没有挂靠的APC节点环境需要保存到`SavedApcState`中，挂靠的APC节点保存到`ApcState`中

`ApcStateIndex`挂靠索引，只有`0, 1`两个值，`ApcStatePointer[ApcStateIndex]`永远指向`ApcState`队列，另一个指向`SavedApcState`



## 线程挂靠

线程在Windows内核中运行时有时候需要暂时`“挂靠”`到别的进程的用户空间，即暂时切换到另一个进程的用户空间。这称为`“进程挂靠”`，因为用户空间是一个进程最主要的特征。

任何一个线程都可以通过调用`KeStackAttachProcess`（该函数会接收`KAPC_STATE`对象，并查看它的`ApcState`参数）临时地挂靠到另一个进程上，也可以通过调用`KeUnstackDetachProcess`函数脱离进程。但是这会有会导致问题一点点的可能性，所以开发者需要把注意力放到上面。

因此，有一个十分重要的事情去理解，我们需要通过使用一个未被文档化但是导出的`KeInitializeApc`调用初始化一个`APC`对象：

```c
VOID
KeInitializeApc(
    __out PRKAPC Apc,
    __in PRKTHREAD Thread,
    __in KAPC_ENVIRONMENT Environment,
    __in PKKERNEL_ROUTINE KernelRoutine,
    __in_opt PKRUNDOWN_ROUTINE RundownRoutine,
    __in_opt PKNORMAL_ROUTINE NormalRoutine,
    __in_opt KPROCESSOR_MODE ApcMode,
    __in_opt PVOID NormalContext
);
```

第三个参数`KAPC_ENVIRONMENT`的枚举：

```c
typedef enum _KAPC_ENVIRONMENT {
    OriginalApcEnvironment,     //不管是否挂靠，插入到不挂靠的队列中，即亲生进程的环境中
    AttachedApcEnvironment,     //插入到挂靠环境中，即进程进程父环境
    CurrentApcEnvironment,      //插入到当前环境
    InsertApcEnvironment        //初始化不插入，调用KeInsertQueueApc再插入
} KAPC_ENVIRONMENT;
```

这个参数指定了`APC`环境
`OriginalApcEnvironment`：告诉系统为当前线程激活它
`AttachedApcEnvironment`：在线程挂靠到另一个进程之前为保存的状态（`KTHREAD::SavedApcState`）激活它
该参数将会在后面保存到`KAPC::ApcStateIndex`成员当中。

为了更好的说明这个概念，让我们看如下的`KiInsertQueueApc`代码：

```c
// KiInsertQueueApc() excerpt:

Thread = Apc-&gt;Thread;
PKAPC_STATE ApcState;

if (Apc-&gt;ApcStateIndex == 0 &amp;&amp; Thread-&gt;ApcStateIndex != 0)
{
  ApcState = &amp;Thread-&gt;SavedApcState;
}
else
{
  Apc-&gt;ApcStateIndex = Thread-&gt;ApcStateIndex;
  ApcState = &amp;Thread-&gt;ApcState;
}
```

所以本质上`KAPC::ApcStateIndex`是一个布尔值：

- 非0：指示`APC`插入到当前线程中，话句话说，`APC`应该执行在当前进程的上下文环境中，也就是线程当前运行的环境。
- 0：指示当前`APC`应该仅仅在源进程的环境中运行，或者在线程在进程挂靠之前的进程环境中。

在`KeStackAttachProcess`函数中，有如下逻辑：

```c
// KeStackAttachProcess() excerpt:

if (Thread-&gt;ApcStateIndex != 0)
{
  KiAttachProcess(Thread, Process, &amp;LockHandle, ApcState);
}
else
{
  KiAttachProcess(Thread, Process, &amp;LockHandle, &amp;Thread-&gt;SavedApcState);
  ApcState-&gt;Process = NULL;
}
```

所以，当本线程第一次挂靠到另一个进程，打个比方：如果它的`KAPC::ApcStateIndex`值为0，当前的`KTHREAD::ApcState`存储在`KTHREAD::SavedApcState`当中，并且以前的`ApcState`即现在的`SavedApcState`不会被使用

这种逻辑处理是为了让系统始终可以访问线程的原始`APC`状态。这可以用于将`APC`插入原始线程，或通过调用`KeUnstackDetachProcess`将线程脱离原进程。

## APC类型

`APC`有两个基础类型：内核`APC`和用户`APC`。`KAPC_STATE::ApcListHead`里面包含了两个链表用来存放内核`APC`和用户`APC`。这两个链表分别有`APC`排队等待线程处理：

```
   //这里有两个链表，0代表内核APC队列，1代表用户APC队列
   &#43;0x000 ApcListHead      : [2] _LIST_ENTRY
```

`KPROCESSOR_MODE`的模式：

```c
typedef enum _MODE {
  KernelMode = 0x0,
  UserMode = 0x1,
  MaximumMode = 0x2
}MODE;
```

好像也有  紧急APC 和 普通APC

紧急APC会插入到APC链表中第一个普通APC前



## 开发细节

### 内核 APC 的内存使用

`KAPC`结构体只能使用从非分页内存分配的内存（或者从类似`NonPagedPool*`类型分配）。即使在`PASSIVE_LEVEL`的`IRQL`初始化并插入`APC`，也可以避免蓝屏死机（`BSOD`）。

为什么要有这样的内存类型限制呢？其他一些`APC`也可以插入到运行在更高调度级别`IRQL`的同一线程中。在插入双链接`APC`列表期间，系统将尝试访问列表中已经存在的其他`KAPC`结构。因此，如果其中任何一个使用的是从分页内存分配的内存，你将会从`DISPATCH_LEVEL`间接访问分页内存，这也是一种会导致蓝屏保护原因。

```c
    PKAPC kApc = ExAllocatePool(NonPagedPool, sizeof(KAPC));
    memset(kApc, 0, sizeof(KAPC));
    
    KeInitializeApc(kApc, thread, OriginalApcEnvironment, KernelApcRoutineCallBack, NULL, NULL, KernelMode, NULL);
```

### 中断和阻塞内核 APC

关于内核模式`APC`，需要记住的重要一点是，它通过中断实现，这意味着它可以发生在代码中的任意两个`CPU`指令之间。

内核开发允许我们阻止`APC`的执行。只有在代码的某些特殊部分起作用：将`IRQL`提升到`APC_LEVEL`或更高级别或将写的代码放在`KeEnterCriticalRegion`和`KeLeaveCriticalRegion`的调用之间。（请注意，这些函数不会阻止所谓的特殊内核`APC`的执行，只有提高`IRQL`级别才能阻止这些`APC`的执行）。

关于我在上面展示的`IRQL`条件限制，一个有趣的事实是，如果`APC`到达关键区域，它不会丢失，稍后将在以下任一函数中处理：`KeLeaveGuardedRegion`、 `KeLeaveCriticalRegion`、`KeLowerIrql`或者在临界区的结尾。

### APC 和驱动卸载的细微差别

在调用内核`APC`回调例程时，有一个微妙的时刻。例如，必须始终提供内核例程（`KernelRoutine`）回调。因此当驱动程序的`APC`回调仍在运行时，自己可能无法从内存中卸载，这将一定会导致蓝屏。

理想情况下，`APC`的系统实现应如下所示：

- 必须在`KAPC`中有对`DriverObject`的引用，在插入`APC`之前，`KeInsertQueueApc`函数应该已经完成了`ObfReferenceObject(Apc-&gt;DriverObject)`（此外，如果`KeInsertQueueApc`失败，也可以在内部调用`ObfDereferenceObject(Apc-&gt;DriverObject)`），通过这些步骤，当有正在排队的`APC`时，驱动程序将不会被卸载。
- 那么，在最后调用`KernelRoutine`/ `NormalRoutine`/`RundownRoutine`之前，系统应该已经将`DriverObject = Apc-&gt;DriverObject`读入本地堆栈，调用适当的`Apc`回调，然后调用`ObfDereferenceObject(DriverObject)`，因为回调返回后Apc本身将无效。
- 此外，如果`RundownRoutine`是无条件调用的，而不是现在的调用方式，也会非常有用。

## 内核中的用户 APC

对于用户模式`APC`，情况在以下方面有所不同：

- 它不能在任何两条`CPU`指令之间执行，或者换句话说，它不是通过`CPU`中断传递的。
- 它必须在3环代码或用户模式上下文中运行。
- 它仅在线程处于警报状态时执行特定的可等待`Windows`函数后运行。

`CPU`首先进入负责路由系统调用的`Windows`内核部分`KiFastCallEntry`，然后调用`KiDeliverApc`根据参数模式处理APC。系统服务调度程序检查用户模式`APC`的存在并调整内核堆栈上的`KTRAP_FRAME`以稍后处理用户模式`APC`。

![image-20241222204816140](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241222204816140.png)

检查并执行`APC`是在`nt!KiDeliverApc`函数中完成。根据第一个参数`mode`，如果传入`0`则为`kernelmode`内核模式，执行内核`APC`，不会执行3环的用户`APC`；如果传入`1`则为`usermode`，会执行用户`APC`，但执行用户模式`APC`时同时也会执行内核模式的`APC`。

在`KiFastCallEntry`中调用`KiDeliverApc`进行用户APC函数的分发与执行。在处理线程的内核模式`APC`之后，它检查`KTHREAD::PreviousMode == UserMode`和`KTHREAD.SpecialApcDisable`是否未设置。如果是，则检查`KTHREAD.ApcState.UserApcPending`是否不为零，表示用户模式`APC`的存在。然后它调用`nt!KiInitializeUserApc`修改用户模式上下文从系统调用返回以处理用户模式`APC`。

为此，在调整`KTRAP_FRAME`以执行返回到本机子系统中的特殊`ntdll!KiUserApcDispatcher`函数之前，`nt!KiInitializeUserApc`会保存系统调用应该返回的原始3环上下文，之后再由`nt!KiInitializeUserApc`返回。

只是稍后由它返回，在执行`sysexit`的`CPU`指令时，由于修改了`KTRAP_FRAME`上下文，`CPU`返回到3环中的`ntdll!KiUserApcDispatcher`函数。该函数依次处理单个用户模式`APC`，然后调用`ntdll!NtContinue(context, TRUE)`将执行返回给内核。我上面描述的循环一直持续到线程队列中没有更多的用户模式`APC`。

用户模式`APC`的一些特殊点：

- 尽管`CPU`可以在中断后的任意两条指令之间的任何时刻进入内核模式，但此时不会调用用户模式`APC`回调。用户模式`APC`只能在执行特殊的`Windows API`调用后才能调用，正如我在此处所描述的。

- 假设任何需要`sysenter`的`Windows API`都可用于在返回时处理用户模式`APC`，前提是某些内核代码为线程设置了`KTHREAD.ApcState.UserApcPending`，并且用户模式`APC`在调用之前排队。

- 设置`KTHREAD.ApcState.UserApcPending`是`MSDN`称为线程的警报状态。这是一个有点令人困惑的术语，大概表示用户APC队列中有元素在等待执行吧。

- 哪些`API`可以设置`KTHREAD.ApcState.UserApcPending`标志？显然，以下记录的函数可以做到这一点：`SleepEx`、`SignalObjectAndWait`、`MsgWaitForMultipleObjectsEx`、`WaitForMultipleObjectsEx`或`WaitForSingleObjectEx`。但也有这些未记录的函数也可以做到这一点：

  - `ntdll!NtTestAlert`：没有输入参数。似乎它的唯一功能是准备所有排队的用户模式`APC`。它在内部调用`nt!KiInitializeUserApc`本身，我在这里描述：

    ```cpp
    NTSTATUS NtTestAlert();
    ```

  - `ntdll!NtContinue`：它将执行返回给内核以继续处理（就像我在此处描述的那样），然后将执行传递给提供的用户模式`ThreadContext`，同时如果设置了`RaiseAlert`，则可以选择设置`KTHREAD.ApcState.UserApcPending`：

    ```c
    NTSTATUS NtContinue(
      IN PCONTEXT ThreadContext,
      IN BOOLEAN RaiseAlert
      );
    ```



## APC执行分析

从`KiFastCallEntry`函数中调用到`KiDeliverApc`进行分发与执行`APC`

若是在其他地方执行到了`KiDeliverApc`函数，大部分都是只执行内核态`APC`，这里以`KiFastCallEntry`调用的`KiDeliverApc`为例

![image-20241222205737302](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241222205737302.png)

将`Alerted`赋值为`0`，以免被其他`APC`打断

比较`UserApcPending`的值，检查用户`APC`队列中是否有元素等待执行

给段寄存器赋值，恢复环境

提升`irql`等级，避免被打断



![image-20241222210138981](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241222210138981.png)

判断第三个参数`_KTRAP_FRAME`是否为空，不为空则执行`KiCheckForSListAddress`检查一下中断上下文中返回的`eip`地址

![image-20241222174642903](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241222174642903.png)

`KiCheckForSListAddress`根据中断上下文中的段选择子判断发生中断前的地址状态

- 如果当前段是`8`即内核态，判断当前EIP是否在`_ExpInterlockedPopEntrySListResume ~ _ExpInterlockedPopEntrySListEnd`之间，如果是，则将EIP置为`_ExpInterlockedPopEntrySListResume`
- 如果当前段是`1B`即用户态，判断当前EIP是否在`_KeUserPopEntrySListResume ~ _KeUserPopEntrySListEnd`之间，如果是，则将EIP置为`_KeUserPopEntrySListResume`

如果中断发生在条件的地址中间，就再跑一遍这个函数



![image-20241222211514507](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241222211514507.png)

执行`KiCheckForSListAddress`之后，会循环获取当前线程的`APC`队列，判断内核队列是是否为空，如果不为空则提升`irql`等级到`Dispatch Level`，避免有其他线程打断后续的执行。

![image-20241222214213494](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241222214213494.png)

判断这个`APC`节点有没有`NormalRoutine`函数，没有则继续往下执行。恢复`irql`等级，清除这个`APC`节点，执行`KernelRoutine`函数

![image-20241222214732920](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241222214732920.png)

跳转回到循环头判断当前`APC`队列是否为空，若不为空则继续循环，提升`irql`等级，添加原子锁，执行`KernelRoutine`函数



若有`NormalRoutine`函数，则会跳转到这里

![image-20241222215613140](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241222215613140.png)

先检查当前的内核`APC`是否在执行，检查用户`APC`是否关闭。没有问题则降低`irql`等级，前后分别执行`KernelRoutine`函数和`NormalRoutine`函数

- 执行内核APC函数`KernelRoutine`时`irql = 1 (APC Level)`，执行前后要提升、降低`irql`等级，执行时要给`_KTHREAD._KAPC_STATE.KernelApcInProgress`置1
- 执行用户APC函数`NormalRoutine`时`irql = 0 (Passive Level)`



![image-20241222222128527](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241222222128527.png)

![image-20241223144543635](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241223144543635.png)

在执行完内核`APC`队列后，判断第一个`mode`参数是否为1，执行用户`APC`队列。先判断是否为空，若为空则降低`irql`等级然后返回

![image-20241223144938320](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241223144938320.png)

再次执行当前节点的内核`APC`函数

![image-20241223145513538](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241223145513538.png)

1. 警惕性是一次性的，如果本来就是警惕性为1的时候，在APC节点函数中又没有R3的函数，就会清除警惕性，等待下一次有警惕性再执行
2. 如果没有警惕性，强行唤醒线程去执行用户APC，但又没有R3函数，那么会修改`UserApcPending`，等待下一次立马执行

![image-20241223145915355](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241223145915355.png)

如果有`NormalRoutine`函数，就会进入`KiInitializeUserApc`函数

在`KiInitializeUserApc`会返回`R3`层分发执行用户`APC`。再返回`R0`层继续遍历`APC`节点，循环执行。



内核态中有四条线

1. R3 - R0  fastcallentry
2. APC
3. 异常
4. KeUserModelCallback



## 示例代码

```c
#include &lt;ntifs.h&gt;


typedef enum _KAPC_ENVIRONMENT {
    OriginalApcEnvironment,     //不管是否挂靠，插入到不挂靠的队列中，即亲生进程的环境中
    AttachedApcEnvironment,     //插入到挂靠环境中，即进程进程父环境
    CurrentApcEnvironment,      //插入到当前环境
    InsertApcEnvironment        //初始化不插入，调用KeInsertQueueApc再插入
} KAPC_ENVIRONMENT;


typedef
VOID
(*PKNORMAL_ROUTINE) (
    IN PVOID NormalContext,
    IN PVOID SystemArgument1,
    IN PVOID SystemArgument2
    );

typedef
VOID
(*PKKERNEL_ROUTINE) (
    IN struct _KAPC* Apc,
    IN OUT PKNORMAL_ROUTINE* NormalRoutine,
    IN OUT PVOID* NormalContext,
    IN OUT PVOID* SystemArgument1,
    IN OUT PVOID* SystemArgument2
    );

typedef
VOID
(*PKRUNDOWN_ROUTINE) (
    IN struct _KAPC* Apc
    );


VOID
KeInitializeApc(
    __out PRKAPC Apc,
    __in PRKTHREAD Thread,
    __in KAPC_ENVIRONMENT Environment,
    __in PKKERNEL_ROUTINE KernelRoutine,
    __in_opt PKRUNDOWN_ROUTINE RundownRoutine,
    __in_opt PKNORMAL_ROUTINE NormalRoutine,
    __in_opt KPROCESSOR_MODE ApcMode,
    __in_opt PVOID NormalContext
);

BOOLEAN
KeInsertQueueApc(
    __inout PRKAPC Apc,
    __in_opt PVOID SystemArgument1,
    __in_opt PVOID SystemArgument2,
    __in KPRIORITY Increment
);

VOID KernelApcRoutineCallBack(
    IN struct _KAPC* Apc,
    IN OUT PKNORMAL_ROUTINE* NormalRoutine,
    IN OUT PVOID* NormalContext,
    IN OUT PVOID* SystemArgument1,
    IN OUT PVOID* SystemArgument2) {

    DbgBreakPoint();
    DbgPrint(&#34;[lo1see] %s success!!\n&#34;, __FUNCTION__);

    // 获取 DriverObject 并释放其引用
    PDRIVER_OBJECT DriverObject = (PDRIVER_OBJECT)(*SystemArgument2);
    if (DriverObject) {
        DbgPrint(&#34;[lo1see] APC executed for DriverObject at: %p\n&#34;, DriverObject);
        ObfDereferenceObject(DriverObject);
    }
    else {
        DbgPrint(&#34;[lo1see] SystemArgument2 is NULL.\n&#34;);
    }

    PUNICODE_STRING unPath = NULL;
    NTSTATUS st = SeLocateProcessImageName(IoGetCurrentProcess(), &amp;unPath);

    if (NT_SUCCESS(st)) {
        DbgPrintEx(77, 0, &#34;[lo1see] APC执行进程 unPath: %wZ\r\n&#34;, unPath);
        ExFreePoolWithTag(unPath, 0);
    }

    ExFreePool(Apc);
}


PETHREAD GetThreadByID(HANDLE tid) {
    PETHREAD pthread = NULL;

    NTSTATUS st = PsLookupThreadByThreadId(tid, &amp;pthread);
    if (NT_SUCCESS(st)) {
        return pthread;
    }
}


VOID DriverUnload(_In_ PDRIVER_OBJECT DriverObject)
{
    UNREFERENCED_PARAMETER(DriverObject);
    DbgPrint(&#34;[lo1see] %s success!!\tHello World Driver Unload\n&#34;, __FUNCTION__);
}


NTSTATUS DriverEntry(_In_ PDRIVER_OBJECT DriverObject, _In_ PUNICODE_STRING RegistryPath)
{
    UNREFERENCED_PARAMETER(RegistryPath);
    //UNREFERENCED_PARAMETER(DriverObject);

    DbgPrint(&#34;[lo1see] %s success!!\tHello World Driver Loaded\n&#34;, __FUNCTION__);
    // 设置驱动程序卸载例程
    DriverObject-&gt;DriverUnload = DriverUnload;

    PUNICODE_STRING unPath = NULL;
    NTSTATUS st = SeLocateProcessImageName(IoGetCurrentProcess(), &amp;unPath);

    if (NT_SUCCESS(st)) {
        DbgPrintEx(77, 0, &#34;[lo1see] main执行进程 unPath: %wZ\r\n&#34;, unPath);
        ExFreePoolWithTag(unPath, 0);
    }

    PKAPC kApc = ExAllocatePool(NonPagedPool, sizeof(KAPC));
    memset(kApc, 0, sizeof(KAPC));

    //PETHREAD thread = GetThreadByID(2564);
    PETHREAD thread = PsGetCurrentThread();
    if (!thread) {
        ExFreePool(kApc);
        DbgPrint(&#34;[lo1see] Failed to find thread by ID.\n&#34;);
        return STATUS_NOT_FOUND;
    }

    ObfReferenceObject(DriverObject);
    KeInitializeApc(kApc, thread, OriginalApcEnvironment, KernelApcRoutineCallBack, NULL, NULL, KernelMode, DriverObject);

    if (!KeInsertQueueApc(kApc, NULL, DriverObject, 0)) {
        // 插入失败，释放资源
        ObfDereferenceObject(DriverObject);
        ExFreePool(kApc);
        DbgPrint(&#34;[lo1see] Failed to insert APC into queue.\n&#34;);
        return STATUS_UNSUCCESSFUL;
    }

    DbgPrint(&#34;[lo1see] APC successfully inserted into queue.\n&#34;);

    return STATUS_SUCCESS;
}
```



---

> 作者:   
> URL: http://localhost:1313/posts/9b5cf17/  

