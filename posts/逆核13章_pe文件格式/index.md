# 逆核——13章_PE文件格式


# **13章  PE文件格式**

## **DOS头**

### **IMAGE_DOS_HEADER**

![image-20241212171007206](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212171007206.png)

e_magic：DOS签名

e_lfanew：只是NT头的偏移（根据不同的文件拥有可变值）

## **DOS存根**

40~4D区域是16位汇编指令。Windows下不会运行该命令。

## **NT头**：IMAGE_NT_HEADERS

![image-20241212165704463](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212165704463.png)

Signature：签名，值为50 45 00 00（PE）

IMAGE_FILE_HEADER FileHeader：文件头

IMAGE_OPTIONAL _HEADER32 OptionalHeader：可选头

### **NT头：文件头**——IMAGE_FILE_HEADER

文件头是表现文件大致属性的IMAGE_FILE_HEAD结构体

![image-20241212165631487](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212165631487.png)

\#1.Machine：每个CPU都拥有唯一的Machine码

![image-20241212165719203](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212165719203.png)

![image-20241212165732046](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212165732046.png)

\#2.NumberOfSections：指出文件中存在的节区数量。该值一定要大于0，且当定义的节区数量与实际节区不同时，将发生运行错误。

\#3.SizeOfOptionalHeader：指出IMAGE_OPTIONAL_HEADER32结构体的长度。IMAGE_OPTIONAL_HEADER32结构体大小已经确定。但Windows的PE装载器需要查看IMAGE_FILE_HEADER的SizeOfOptionalHeader值，从而识别出IMAGE_OPTIONAL_HEADER32结构体的大小

\#4.Characteristics：该字段用于表示文件的属性，文件是否是可运行的形态、是否位DLL文件等信息，以bit OR形式组合起来。

![image-20241212165748781](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212165748781.png)

![image-20241212165801429](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212165801429.png)

### **NT头：可选头**——IMAGE_OPTIONAL_HEADER

以下值是文件运行必须的，设置错误将导致文件无法正常运行

![image-20241212165818143](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212165818143.png)

![image-20241212165830990](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212165830990.png)

\#1.Magic:为IMAGE_OPTIONAL_HEADER32结构体时，Magic码为10B；为IMAGE_OPTIONAL_HEADER64结构体时，Magic码为20B。

\#2.AddressOfEntryPoint：持有EP的RVA值。指出程序最先执行的代码起始地址。

\#3.ImageBase：指出文件的优先装入地址。（EIP = ImageBase &#43; AddressOfEntryPoint）

\#4.SectionAlignment，FileAlignment：FileAlignment制定了节区在磁盘文件中的最小单位，而SectionAlignment指定了节区在内存中的最小单位。磁盘文件或内存的节区大小必定为FileAlignment或SectionAlignment值的整数倍。

\#5.SizeOfImage：指定了PE Image在虚拟内存中所占空间的大小。一般而言，文件的大小与加载到内存中的大小是不同的。

\#6.SizeOfHeaders：指出整个PE头的大小（该值必须是FileAlignment的整数倍）

\#7.Subsystem：用来区分系统驱动文件和普通的可执行文件。

\#8.NumberOfRvaAndSizes.用来指定DataDirectory（IMAGE_OPTIONAL_HEADER32结构体的最后一个成员）数组的个数。PE装载器通过查看NumberOfRvaAndSizes值来识别数组大小，换言之，数组大小也可能不是16。

\#9.DataDirectory：是由IMAGE_DATA_DIRECTORY结构体组成的数组，数组每项都有被定义的值。

![image-20241212165846883](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212165846883.png)

## 节区头——IMAGE_SECTION_HEADER

![image-20241212165910607](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212165910607.png)

#1.VirtualSize：内存中节区所占大小

#2.&lt;font color = &#34;red&#34;&gt;VirtualAddress&lt;/font&gt;：（不带有任何值）内存中节区起始地址（RVA）

#3.SizeOfRawData：磁盘文件中节区所占大小

#4.&lt;font color = &#34;red&#34;&gt;PointerToRawData&lt;/font&gt;：（不带有任何值）磁盘文件中节区起始位置

#5.Characteritics：节区属性（bit OR）

&lt;font color = &#34;red&#34;&gt;VirtualAddress与PointerToRawData不带有任何值，分别由（定义在IMAGE_OPTIONAL_HEADER32中的）SectionAlignment与FileAlegnment确定。&lt;/font&gt;

## RVA to RAW

```
RAW - PointerToRawData = RVA - VirtualAddress
                   RAW = RVA - VirtualAddress &#43; PointerToRawData
```



## IMAGE_IMPORT_DESCRIPTOR

![image-20241212170016551](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212170016551.png)

IMAGE_IMPORT_DESCRIPTOR结构体中记录着PE文件要导入哪些库文件

#1.OriginalFirstThunk：INT的地址（RVA）

#2.Name：库名称字符串的地址（RVA）

#3.FirstThunk：IAT的地址（RVA）

导入多少库就存在多少个IMAGE_IMPORT_DESCRIPTOR结构体

## IMAGE_EXPORT_DIRECTORY

![image-20241212170050800](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212170050800.png)

#1.NumberOfFunctions：实际Export函数的个数

#2.NumberOfNames：Export函数中具名的函数个数

#3.AddressOfFunctions：Export函数地址数组（数组元素个数 = NumberOfFunctions）

#4.AddressOfNames：函数名称地址数组（数组元素个数 - NumberOfNames）

#5.AddressOfNameOrdinals：Ordinal地址数组（数组元素个数 = NumberOfNames）

![image-20241212170233131](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212170233131.png)

![image-20241212170226394](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212170226394.png)

## 基址重定位偏移表——IMAGE_BASE_RELOCATION

![image-20241212170311852](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212170311852.png)

#1.VirtualAddress：是一个基准地址（Base Address），实际是RVA值。

#2.SizeOfBlock：指重定位块的大小。

#3.TypeOffset数组：不是结构体成员，是以注释形式存在的，表示在该结构体之下会出现WORD类型的数组，并且该数组元素的值就是硬编码在程序中的地址偏移


---

> 作者: lo1see  
> URL: http://localhost:1313/posts/%E9%80%86%E6%A0%B813%E7%AB%A0_pe%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F/  

