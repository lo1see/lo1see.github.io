# PE文件加载器




# PELoader

## PELoader？

首先要清楚PELoader是干什么的。

PELoader就相当于一个PE文件加载器，通过运行PELoader可以打开一个PE程序，该PE程序并不是系统为你打开的。运行起来的效果一般和双击是一样的。个人认为也是可以通过PEloader修改文件在内存中的数据的。

通过写PEloader可以让我们更清楚的了解PE文件的结构，以及运行原理，这对于理解计算机底层是十分有帮助的。

## IDE选择

推荐使用Visual Studio

师傅在辅导我写PEloader时，指导我使用了Visual Studio 2022。VS的调试功能十分强大，可以在逐步调试时查看内存，将C代码反汇编等等。在代码不能顺利运行时可以更好的进行调试。

VS的安全检查比较严格，自己的代码运行时需要关闭安全检查。还有完成后要注意自己要运行的程序是32位还是64位的程序，以及VS本身是x86运行还是x64运行。

## main()函数

因为代码量不小，所以我选择了分为main函数和头文件，再调用头文件中的函数。

在main函数中

### 1.文件路径

我们需要获得文件的地址，也就是文件路径字符串。

### 2.寻找文件

根据文件路径，通过`fopen()`函数找到文件，并给予权限。

```
if ((fp = fopen(file, &#34;rb&#34;)) == NULL)
{
	printf(&#34;Open file fail&#34;);
	exit(1);
}
```

### 3.分配&#34;磁盘&#34;

计算出文件大小，并分配出一块等大小的内存区域(我声明为`pBuf`)

```
fseek(fp, 0, SEEK_END);
last = ftell(fp);
fsize = last;
```

### 4.复制

将文件的二进制信息全部复制到这块内存区域中

```
fseek(fp, 0, SEEK_SET);
pBuf = (PBYTE)malloc(fsize);
fread(pBuf, fsize, 1, fp);
```

## Check一下PE

在将文件读入`malloc()`给予的内存空间中后，我们拥有了对文件进行操作的权限。在此我将`pBuf`视为在磁盘中，稍后再将“磁盘”中的文件信息传入内存。

对`pBuf`进行一个检查，check以下该文件是不是PE文件，若不是，则退出程序。

判断PE文件方法即为判断DOS头中的`e_magic`和NT头中的`Signature`签名。

```
if (pDOSheader-&gt;e_magic == IMAGE_DOS_SIGNATURE) {
	if (pNTheader-&gt;Signature == IMAGE_NT_SIGNATURE) {
		return true;
		}
	}
```

## LoadPE

这里开始进行最主要的加载过程

### 1.定位

利用传入的参数`pBuf`，得到DOS头，NT头，节区头并赋值

```
PIMAGE_DOS_HEADER pDOSheader = (PIMAGE_DOS_HEADER)pBuf;//赋值DOS头

PIMAGE_NT_HEADERS32 pNTheader = (PIMAGE_NT_HEADERS32)(pBuf &#43; pDOSheader-&gt;e_lfanew);//赋值NT头

PIMAGE_SECTION_HEADER pSecheader = (PIMAGE_SECTION_HEADER)((PBYTE)(pNTheader)&#43; 0x18 &#43; (pNTheader-&gt;FileHeader.SizeOfOptionalHeader));//赋值节区头
```

### 2.分配&#34;内存&#34;

通过`VirtualAlloc()`函数分配出一片和PE文件同样大的内存区域~，这个大小在可选头的`SizeOfImage`有保存。我们将这片区域`pAlloc`视为“内存”中。

```
PBYTE pAlloc = (PBYTE)VirtualAlloc(NULL, pNTheader-&gt;OptionalHeader.SizeOfImage, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
```

然后对这片区域利用`memset()`进行初始化。之后就可将在“磁盘”中的`pBuf`的文件头加载到“内存”区域的pAlloc。`pAlloc`的地址就是程序运行时的加载地址。

### 3.加载节区

通过一二步之后，准备工作就完成了。接下来要开始循环加载节区信息。

通过学习PE文件的知识，我们知道了磁盘文件或内存的节区大小必定为`FileAlignment`或`SectionAlignment`值的整数倍，若不为整数倍，计算机会自动补00对齐。

通过010 editor观察文件易发现节区头的`VirtualAddress`和`PointToRawData`是已经对齐好的，我们不需要写偏移函数来手动对齐。所以只需要

```
memmove(pAlloc &#43; pSecheader-&gt;VirtualAddress, pBuf &#43; pSecheader-&gt;PointerToRawData, pSecheader-&gt;Misc.VirtualSize);
```

就可将节区加载进内存。但后面要补的0该怎么办呢？其实早在初始化`pAlloc`时就已经将不加载数据的区域补了0，并不需要担心。

加载完一个节区之后不要忘记进入下一个节区继续加载

```
pSecheader = (PIMAGE_SECTION_HEADER)((PBYTE)pSecheader &#43; (BYTE)(sizeof(IMAGE_SECTION_HEADER)));
```

### 4.加载重定位表

加载完节区之后，我们还有重定位表和导入表没有处理。先处理重定位表。

值得注意的是有的PE文件并**不存在重定位表**，若重定位表地址为00000000，则我们应直接跳过重定位表的处理；有的PE文件**重定位表也不止一个**，不过可选头中给的重定位表的大小是所有重定位表的总和，利用这个大小即可知道是否处理完了重定位表。

先定位找到重定位表，并确定重定位表的大小

```
PIMAGE_BASE_RELOCATION pBaseReloc = (PIMAGE_BASE_RELOCATION)(pNTheader-&gt;OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_BASERELOC].VirtualAddress &#43; pAlloc);

int SizeOfBaseReloc = pNTheader-&gt;OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_BASERELOC].Size;
```

这样就算有多个重定位表，我们也能找到各个重定位表的地址和大小了。

```
pBaseReloc-&gt;VirtualAddress;
pBaseReloc-&gt;SizeOfBlock;
```

通过这两个数据即可得到块的数量和每个块的数据。

通过对重定位表的学习，我们已清楚重定位原理。先利用位操作得到每个块的前四位，判断每个块的类型，一般为3；并同时得到后12位的偏移地址

```
WORD type = TypeOffset[i] &gt;&gt; 12;
WORD offset = TypeOffset[i] &amp; 0x0FFF;
```

偏移地址加上`VirtualAddress`可得出一个地址，该地址存储着程序的硬编码地址，应读取该值之后减去`ImageBase`值，再加上实际加载地址pAlloc，得到实际重定位后的硬编码地址。

```
*((DWORD*)(offset &#43; (pBaseReloc-&gt;VirtualAddress) &#43; pAlloc)) - pNTheader-&gt;OptionalHeader.ImageBase &#43; (DWORD)pAlloc;
```

我们将这个值覆盖存储到原先的硬编码地址，重定位就完成了。



*若还有多个重定位表，记得在加载完成后给`pBaseReloc`加上一个`SizeOfBlock`*

```
pBaseReloc = (PIMAGE_BASE_RELOCATION)((PBYTE)pBaseReloc &#43; pBaseReloc-&gt;SizeOfBlock);
```

接着回去加载下一个重定位表

### 5.加载导入表

重定位表处理完成后，就该处理导入表了。

导入表拥有多个dll文件，dll文件中也有很多函数，要循环加载。

先声明得到导入表的位置

```
PIMAGE_IMPORT_DESCRIPTOR pImport = (PIMAGE_IMPORT_DESCRIPTOR)(pNTheader-&gt;OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT].VirtualAddress &#43; pAlloc);

```

`pImport-&gt;Name`就是dll文件的文件名字符串的地址，再加上加载地址`pAlloc`找到字符串，利用win32的LoadLibrary()函数加载dll文件进入内存。

加载完dll之后就可以计算IAT和INT的地址，开始加载dll中的函数了。

**IAT和INT下都有一个联合体(union)，为u1**

```
typedef struct _IMAGE_THUNK_DATA32 {
    union {
        DWORD ForwarderString;      // PBYTE 
        DWORD Function;             // PDWORD
        DWORD Ordinal;
        DWORD AddressOfData;        // PIMAGE_IMPORT_BY_NAME
    } u1;
} IMAGE_THUNK_DATA32;
typedef IMAGE_THUNK_DATA32 * PIMAGE_THUNK_DATA32;
```

拥有四个元素，这四个元素公用了同一个`DWORD`，即为同一个值。但该值可拥有多种作用，便有了多种写法。

在导入函数时，先判断一下函数是使用哪种方法导入的。

若是用序号导入函数，则INT中存储的地址最高位即为1。故进行判断

```
if (pINT-&gt;u1.AddressOfData &amp; IMAGE_ORDINAL_FLAG32)
```

若为真，则函数使用序号导入，而不使用字符串来查找函数

```
pIAT-&gt;u1.AddressOfData = (DWORD)(GetProcAddress(hProcess, (LPCSTR)(pINT-&gt;u1.Ordinal)));
```

若为假，就可能需要使用字符串查找函数地址

```
pIAT-&gt;u1.AddressOfData = (DWORD)(GetProcAddress(hProcess, pFucname-&gt;Name));
```

函数导入完毕，别忘了

```
pINT&#43;&#43;;
pIAT&#43;&#43;;
```

导入完一个dll之后也要

```
pImport&#43;&#43;;
```

*这里的&#43;1是加了`IMAGE_IMPORT_DESCRIPTOR`一个结构体的大小，而不是一个字节，所以可以跳到下一个dll的导入表。*

## 进入main()函数

至此，已load完成。接下来就是利用函数指针，指向`EntryPoint`，即可调用主函数，运行程序。

## Congratulation！

顺利的话程序这样就已经运行起来了，恭喜。



源码：

https://github.com/lo1see/PELoader


---

> 作者: lo1see  
> URL: http://localhost:1313/posts/%E6%99%AE%E9%80%9A%E7%9A%84pe%E5%8A%A0%E8%BD%BD%E5%99%A8/  

