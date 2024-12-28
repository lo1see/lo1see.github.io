# Windows内核—系统描述符笔记—段


&lt;!--more--&gt;



# windows内核—系统描述符笔记—段

CS	段代码寄存器，描述代码

SS	栈段，描述栈，比如局部变量

DS	数据段，堆地址，全局变量

ES	扩展段

FS	上下文环境段，R3代表TEB，R0代表KPCR



## 段选择子

![1730102555963](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/1730102555963.png)



## 段描述符

![1730102954233](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/1730102954233.jpg)

AVL：保留，提供给操作系统的软件使用

D/B：默认操作数大小。例如为0则push默认压入2字节数据，1则压入4字节数据

DPL：Descriptor privilege level	描述权限等级，所需权限等级
CPL：Current privilege level	看CS段的权限等级
RPL：request privilege level	请求权限等级，段选择子的后两位

P：1表示段为有效段，0表示段为无效段

G：是否以页为单位

TYPE - Segment type

![1730104261837](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/1730104261837.jpg)



内存向下生长：被limit包含的内存区域不可访问，未被包含的区域可以访问

内存向上生长：被limit包含的内存区域可以访问，未被包含的区域不可访问

纯段模式：非一致代码段：应用层不能调用内核代码

​				一直代码段：应用层可以直接调用内核代码，CS描述的一致代码段



跨段call far需要用retf返回


---

> 作者:   
> URL: http://localhost:1313/posts/72b7e58/  

