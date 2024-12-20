# VMP IAT修复学习


&lt;!--more--&gt;

# VMP导入表恢复

## 一、VMP对导入表的处理方式：

### 方式一：`call	段寄存器:[xxxx]`

原来是`FF 15`的6字节`call`指令，保护后变为1字节的`push/pop  reg` &#43; 5字节`E8 ??`的`call  vmp0`，或5字节`E8 ??`的`call  vmp0` &#43; 1字节的任意垃圾指令

原：

![image-20241219220332317](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241219220332317.png)

1&#43;5类型，VMP后：

![image-20241219220348670](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241219220348670.png)

原：

![image-20241219220142180](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241219220142180.png)

5&#43;1类型，VMP后：

![image-20241219220226703](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241219220226703.png)



### 方式二：`jmp	段寄存器:[xxxx]`

原来是`FF 25`的6字节`jmp`指令，保护后变为1字节的`push/pop  reg` &#43; 5字节`E8 ??`的`call  vmp0`，或5字节`E8 ??`的`call  vmp0` &#43; 1字节的任意垃圾指令

原：

![image-20241219220951445](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241219220951445.png)

1&#43;5类型，VMP后：

![image-20241219221003187](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241219221003187.png)

原：

![image-20241219220733711](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241219220733711.png)

5&#43;1类型，VMP后

![image-20241219220748028](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241219220748028.png)

### 方式三：`mov reg, 段寄存器:[xxxx]	 &#43;	call reg`

如果是5字节的mov，则直接变为5字节 E8的`call vmp0`

如果是6字节的mov，则变为1&#43;5或者5&#43;1类型 E8的`call vmp0`



## 二、整体修复思路

根据前面找到的VMP对IAT的保护方式，可以发现统一的特征就是`E8`开头的`call vmp0`，于是利用这个特征，

对text段进行搜索`E8`，得到一系列地址；
反汇编并计算`call`的跳转目标地址，判断目标地址是不是跳出了当前节区，如果未跳出则过滤这条地址；
对进一步得到的一系列地址进行`unicorn`模拟执行，对返回地址进行判断；
如果是其他模块的导出函数，认为是原IAT表项



## 三、逻辑

### 寻找VMP节区

计算节区内数据的熵值（Entropy），衡量数据的不确定性或信息量。因为VMP虚拟机中的代码有很多垃圾代码，随机性很高，所以可以用计算熵值的方式过滤得到VMP的节区，以免节区名称被修改找不到VMP节区

```C&#43;&#43;
double calcEntropy(std::vector&lt;uint8_t&gt; bytes)
{
	double Entropy{};
	std::map&lt;std::uint8_t, double&gt; mapByteProbabilities;
	std::map&lt;std::uint8_t, int&gt; mapByteFrequencies;

	for (const auto&amp; byte : bytes)
		mapByteFrequencies[byte]&#43;&#43;;

	for (const auto&amp; pair : mapByteFrequencies)
		mapByteProbabilities[pair.first] = static_cast&lt;double&gt;(pair.second) / bytes.size();

	for (const auto&amp; pair : mapByteProbabilities)
		Entropy -= pair.second * log2(pair.second);

	return Entropy;
}
```

### 搜索并过滤得到`call vmp0`

搜索以0xE8为第一字节的五字节特征码，再反编译计算call的跳转真实地址，超出当前节区且未超出当前模块，保存。

### 模拟执行判断保护方式

保存模拟执行前的起始地址，对`call vmp0`进行`unicorn`模拟执行，设置`UC_HOOK_CODE`

判断VMP对IAT是哪种保护类型：

#### `call	段寄存器:[xxxx]`

返回时直接`retn`返回到目标函数地址

例：

![image-20241220121759312](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220121759312.png)

读取`sp &#43; sizeof(ULONG_PTR)`即栈中第二个值的地址，该地址若是`原addr &#43; 5`则为1&#43;5类型，若是`原addr &#43; 6`则为5&#43;1类型，保存记录

#### `jmp	段寄存器:[xxxx]`

因为原`jmp`指令不会对堆栈进行操作，保护后变为了`call`影响了堆栈，所以在最后`ret`返回的时候会平衡堆栈  `ret sizeof(ULONG_PTR)`，32位是`ret 4`，64位是`ret 8`，最终返回地址也是目标函数地址

![image-20241220122018990](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220122018990.png)

对于`jmp`类型不能取到返回地址，但是通过分析`call vmp0`内的指令可以发现两种保护类型的特点：

##### 5&#43;1类型：

直接执行`call vmp0`进入保护函数

![image-20241220125729792](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220125729792.png)

`push rdi`保存本次用来寻找目标导入函数的`vRip寄存器`

![image-20241220125840313](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220125840313.png)

经过一系列运算最终得到目标函数地址，计算结果保存在`rdi`中；通过`xchg`指令取出`push rdi`保存的值；最后`ret 8`平衡堆栈

![image-20241220125908657](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220125908657.png)

##### 1&#43;5类型：

先`push rbp`后`call vmp0`，`rbp`作为本次用来寻找目标导入函数的`vRip寄存器`

![image-20241220130145723](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220130145723.png)

把栈顶两数据调换位置，以便最终恢复`rbp`

![image-20241220130518044](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220130518044.png)

![image-20241220130543619](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220130543619.png)

经过一系列解密后，最终`xchg [rsp], rbp`将计算后`rbp`中的`vRip`与栈顶存储的`rbp`原值置换，可以返回到目标导入函数中且恢复环境

![image-20241220130701456](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220130701456.png)

##### 特点：

5&#43;1类型是进入`call vmp0`之后保存`vRip寄存器`，计算`vRip值`之后利用`xchg`交换回原来的值，`call vmp0`执行前后除`sp`寄存器的值以外不会变动

1&#43;5类型是进入`call vmp0`之前保存`vRip寄存器`，进入之后交换栈顶两数据，计算`vRip值`之后用`xchg`将栈顶数据与`sp`寄存器交换，`call vmp0`执行前后除`sp`寄存器外还有`vRip寄存器`的值也发生了变动

那么可以在unicorn模拟执行配置时，将除`sp`寄存器以外的寄存器设为0，将栈中数据设有值如0xFF，最终模拟执行到`ret sizeof(ULONG_PTR)`时判断是否有寄存器的值为非0值，若无则为`5&#43;1类型`的`call vmp0`，若有则为`1&#43;5类型`的`call vmp0`

#### `mov reg, 段寄存器:[xxxx]	 &#43;	call reg`

因为这种方式保护的是`mov`指令，所以最终返回地址是`原地址 &#43; 5/6`的地址，对应1&#43;5或5&#43;1的情况。

同时在这个call中只会修改除`sp寄存器`外的一个寄存器，该寄存器就作为`vRip`寄存器存储目标导入函数。

## 参考

[VMP导入表修复-加壳脱壳-看雪-安全社区|安全招聘|kanxue.com](https://bbs.kanxue.com/thread-269008.htm)

https://github.com/KuNgia09/vmp3-import-fix

https://github.com/colby57/VMP-Imports-Deobfuscator

https://github.com/PlaneJun/VMP-Import-Fix

## 改动

学习整合了一下优秀VMP IAT修复工具的源码，修改添加了对x86架构的支持，以及添加了修复后的dump功能

仓库：https://github.com/lo1see/VMP-IAT-recover


---

> 作者:   
> URL: http://localhost:1313/posts/c4731d7/  

