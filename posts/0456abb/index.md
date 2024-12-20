# Windows内核—系统描述符笔记—门


&lt;!--more--&gt;



# windows内核—系统描述符笔记—门

当段描述符的S标志（描述符类型）为0，该描述符为系统描述符。处理器可以识别以下类型的系统描述符：

- 局部描述符表（LDT）段描述符
- 任务状态段（TSS）描述符
- 调用门描述符
- 中断门描述符
- 陷阱门描述符
- 任务门描述符

这些描述符又可以分为两类：系统段描述符和门描述符。系统段描述符指向系统段（LDT和TSS段）。系统段描述符指向系统段（LDT和TSS段）。门描述符或者持有指向在代码段的过程的入口点的指针，或者持有TSS（任务门）的段选择符。

![d4f5a6807f7117c9d783c119dddae6b5](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/d4f5a6807f7117c9d783c119dddae6b5.png)

根据type数值选择门



## 调用门

![1730119067171](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/1730119067171.jpg)

Param.Count：4位 表示参数的个数，最多可以有32个参数

Type：门的类型

S：为0，表示系统段

DPL：CS.DPL == SS.DPL == 调用门.DPL

P：表示段是否有效

通过调用门，会压入`ss, sp, cs, ip`寄存器，retf会恢复寄存器，如有参数需要retf 8平衡堆栈
eip
cs
esp
ss

jmp 在调用门只能同权限跳转

retf 规定只能只能同权限或向低权限跳转

call 是同权限或提权跳转



## 中断门

IDT表

![6bf4a45d0905d745236ea44fd64ac0c5](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/6bf4a45d0905d745236ea44fd64ac0c5.png)



中断表描述符

![1730125254096](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/1730125254096.png)

Type.D：默认操作数，32位运行为1，64位系统为0



 外设触发的是中断，软件触发的是异常

中断函数使用`iret*`，16位系统使用`iret`，32位系统使用`iretd`，64位系统使用`iretq`

`sgdt, sidt`保存gdt, idt寄存器信息

`lgdt, lidt`修改gdt, idt寄存器信息



中断门和陷阱门的区别：

1. 进入中断门会清除`eflag`的`VM NT IF TF`位

   进入陷阱门会清除`eflag`的`VM NT TF`位

2. 由于中断门会清除IF位，会导致一些中断会触发悬挂，陷阱门则不会

 


---

> 作者:   
> URL: http://localhost:1313/posts/0456abb/  

