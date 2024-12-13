# 逆核——16章_基址重定位




# 16章  基址重定位表

## PE重定位

指PE文件无法加载到ImageBase所指位置，而是被加载到其他地址时发生的一系列的处理行为。

如：向进程的虚拟内存加载PE文件（EXE/DLL/SYS）时，文件会被加载到PE头的ImageBase所指的地址处。若加载的是DLL（SYS）文件，且在ImageBase位置处已经加载了其他的DLL（SYS）文件，那么PE装载器就会将其加载到其他未被占用的空间。

### DLL/SYS/EXE

在Windows Vista之后的版本映入了ASLR安全机制，每次运行EXE文件都会被加载到随机地址，大大增强了系统安全性。

对于各OS的主要系统DLL，微软会根据不同版本分别赋予不同的ImageBase地址。同一系统的kernel32.dll、user32.dll等会被加载到自身固有的ImageBase，所以，系统的DLL实际不会发生重定位问题。

## 基址重定位偏移表——IMAGE_BASE_RELOCATION

![image-20241212170414443](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212170414443.png)

#1.VirtualAddress：是一个基准地址（Base Address），实际是RVA值。

#2.SizeOfBlock：指重定位块的大小。

#3.TypeOffset数组：不是结构体成员，是以注释形式存在的，表示在该结构体之下会出现WORD类型的数组，并且该数组元素的值就是硬编码在程序中的地址偏移

### 基址重定位表分析方法

**例：**

![image-20241212170457774](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212170457774.png)

VirtualAddress成员（基准地址）的值为1000，SizeOfBlock成员的值为150,。也就是说，TypeOffset数组的基准地址（起始地址）为RVA1000，块的总大小为150。块的末端显示为0。TypeOffset值为2个字节大小，是由4位的Type与12位的Offset合成的。比如，TypeOffset的值为3420。

![image-20241212170511489](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212170511489.png)

高4位用作Type。TypeOffset的低12位才是真正的偏移，该偏移值基于VirtualAddress的偏移。所以程序中硬编码地址的偏移使用下面等式换算：Virtual Address（1000） &#43; Offset（420） = 1420（RVA）

### 文件重定位表

EXE可能也有重定位表的，不能绝对地说都没有。大部分EXE没有重定位表是因为：EXE作为独立的可执行程序，它初始化时系统会给它分配独立的进程内存空间，这个刚分配的内存空间肯定是可以分配一块与EXE的ImageBase相同地址的内存来装载EXE本身，因此通常不需要重定位。而DLL没有独立的进程空间，必须依附于某个EXE，如果两个DLL的映像基址都是0x10000000,第一个可以加载到这个位置，后加载的肯定得换个位置，这时就得进行重定位，因此必须要准备重定位表。MapViewOfFile映射到内存后的文件与硬盘上的组织方式是一样的，而LoadLibrary才是真正的按PE格式加载到内存。


---

> 作者: lo1see  
> URL: http://localhost:1313/posts/%E9%80%86%E6%A0%B816%E7%AB%A0_%E5%9F%BA%E5%9D%80%E9%87%8D%E5%AE%9A%E4%BD%8D/  

