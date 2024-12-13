# 反调试笔记


# **静态反调试技术**

## PEB

### PEB.BeingDebugged(&#43;0x2)

进程处于调试状态时，PEB.BeingDebugged成员(&#43;0x2)的值被设为1（TRUE）；在非调试状态时，其值被设为0（FALSE）。

IsDebuggerPresent()API可以获取PEB.BeingDebugged的值来判断是否处于被调试状态。

### Ldr(&#43;0xC)

调试进程时，**堆内存区域**中会出现一些特殊标识，表示它正处于被调试状态。其中最醒目的是，未使用的**堆内存区域**全部填充着0xEEFEEEFE，这证明正在调试进程。

（可能不适用于windows XP以后的系统）

### Process Heap(&#43;0x18)

PEB.ProcessHeap成员(&#43;0x18)是指向HEAP结构体的指针。

![8cef8381404910ca18bb17f442e17e1a.png](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/8cef8381404910ca18bb17f442e17e1a.png)

进程处于被调试状态时，Flags(&#43;0xC)与Force Flags成员(&#43;0x10)被设置为特定值。

PEB.ProcessHeap成员(0x18)既可以从PEB结构体直接获取，也可以通过GetProcessHeap() API获取。

#### Flags(&#43;0xC) &amp; Force Flags(&#43;0x10)

进程正常运行（非调试运行）时，Heap.Flags成员(&#43;0xC)的值为0x2，Heap.ForceFlags成员(&#43;0x10)的值为0x0。进程处于被调试状态时，这些值也会随之改变。所以，比较这些值就可以判断进程是否处于被调试状态。

- 破解：只要将HEAP.Flags与HEAP.ForceFlags的值重新设置为2与0计科（HEAP.Flags=2，HEAP.ForceFlags=0）。

（可能不适用于windows XP以后的系统）

### NtGlobalFlag(&#43;0x68)

在调试进程时，PEB.NtGlobalFlag成员(&#43;0x68)的值会被设置为0x70。所以检测该成员的值计科判断进程是否处于被调试状态。

NtGlobalFlag 0x70是下列Flags值进行bit OR（位或）运算的结果。

![1c4941e3f4bb73dab1bf9d6b6974adaf.png](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/1c4941e3f4bb73dab1bf9d6b6974adaf.png)

被调试进程的堆内存中存在（不同于非调试运行进程的）特别标识，因此在PEB.NtGlobalFlag成员中添加了上述标志。

- 破解：重设PEB.NtGlobalFlag值为0即可（PEB.NtGlobalFlag = 0）。

### NtQueryInformationProcess()

通过NtQueryInformationProcess() API可以获取各种与进程调试相关的信息。



---

> 作者: lo1see  
> URL: http://localhost:1313/posts/%E5%8F%8D%E8%B0%83%E8%AF%95%E7%AC%94%E8%AE%B0/  

