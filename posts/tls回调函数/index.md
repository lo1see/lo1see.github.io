# TLS回调函数




# TLS介绍

## 定义

引用官方的释义：[Thread Local Storage (TLS)](https://learn.microsoft.com/en-us/cpp/parallel/thread-local-storage-tls?view=msvc-160)

线程本地存储 (TLS) 是给定多线程进程中的每个线程分配存储线程特定数据的位置的方法。通过 TLS API ( [TlsAlloc](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-tlsalloc) ) 支持动态绑定（运行时）线程特定数据。除了现有的 API 实现之外，Win32 和 Microsoft C&#43;&#43; 编译器现在还支持静态绑定（加载时）每线程数据。

## 作用

在TLS中定义的变量，同一个线程里的所有函数均可访问，但其他线程无法访问

## 运行

每当创建/终止线程时会自动调用执行的函数（**创建进程的主线程时也会自动调用回调函数，且回调函数的执行顺序是先于EP代码的执行，所以TLS回调函数的这个特性通常被用于反调试技术**）由于是创建和终止线程时都会调用，所以在程序从打开到结束这个TLS回调函数会被执行两次。

# 文件结构

在NT头的 IMAGE_DATA_DIRECTORY 数组中可以查到 IMAGE_DATA_DIRECTORY  TLSDirectory，知晓TLS结构体所在的RVA和RAW以及大小。

![](https://s1.ax1x.com/2023/04/02/ppfGs41.png)



TLS根据编译时也分为了x86和x64两种结构体

```C
typedef struct _IMAGE_TLS_DIRECTORY32 {
    DWORD   StartAddressOfRawData;
    DWORD   EndAddressOfRawData;
    DWORD   AddressOfIndex;             // PDWORD
    DWORD   AddressOfCallBacks;         // PIMAGE_TLS_CALLBACK *
    DWORD   SizeOfZeroFill;
    union {
        DWORD Characteristics;
        struct {
            DWORD Reserved0 : 20;
            DWORD Alignment : 4;
            DWORD Reserved1 : 8;
        } DUMMYSTRUCTNAME;
    } DUMMYUNIONNAME;

} IMAGE_TLS_DIRECTORY32;
typedef IMAGE_TLS_DIRECTORY32 * PIMAGE_TLS_DIRECTORY32;

typedef struct _IMAGE_TLS_DIRECTORY64 {
    ULONGLONG StartAddressOfRawData;
    ULONGLONG EndAddressOfRawData;
    ULONGLONG AddressOfIndex;         // PDWORD
    ULONGLONG AddressOfCallBacks;     // PIMAGE_TLS_CALLBACK *;
    DWORD SizeOfZeroFill;
    union {
        DWORD Characteristics;
        struct {
            DWORD Reserved0 : 20;
            DWORD Alignment : 4;
            DWORD Reserved1 : 8;
        } DUMMYSTRUCTNAME;
    } DUMMYUNIONNAME;

} IMAGE_TLS_DIRECTORY64;
```

结构体中各个参数的意义大致如下：

- StartAddressOfRawData：TLS模板的起始位置的VA，这个所谓的模板其实就是用于初始化TLS函数的数据

- EndAddressOfRawData：TLS模板终止位置的VA

- AddressOfIndex：存储TLS索引的位置

  （以上3个参数指向的地址可以存储为00000000即NULL区域）

- **AddressOfCallBacks：指向TLS注册的回调函数的函数指针(地址)数组**

- SizeOfZeroFill：用于指定非零初始化数据后面的空白空间的大小

- Characteristics：属性

# 调试技巧

因为TLS函数会在main()函数之前执行，所以要设置系统断点或TLS回调函数断点，这样调试器就可以在TLS函数执行之前断下。

也可以用PEview或010 editor等程序找到TLS结构体的AddressOfCallBacks成员所储存的地址，直接跳到那里设下断点，再Ctrl &#43; F2重新调试就可以运行到断点处。

# 程序编写

## 声明TLS回调函数

告诉编译器要使用TLS回调函数，即声明TLS回调函数。以下代码用了条件语句判定要编写x64还是x86的程序：

```C
#ifdef _WIN64       //64位
     #pragma comment (linker, &#34;/INCLUDE:_tls_used&#34;)  
     #pragma comment (linker, &#34;/INCLUDE:tls_callback_func&#34;) 
#else               //32位
     #pragma comment (linker, &#34;/INCLUDE:__tls_used&#34;) 
     #pragma comment (linker, &#34;/INCLUDE:_tls_callback_func&#34;)
#endif
```

## 回调函数定义：

```C
typedef VOID
(NTAPI *PIMAGE_TLS_CALLBACK) (
    PVOID DllHandle,
    DWORD Reason,
    PVOID Reserved
);
```

参数1： `PVOID DllHandle` ：指向DLL实例句柄

参数2：`DWORD Reason` ：调用TLS回调函数的原因：

```C
#define DLL_PROCESS_ATTACH   1
#define DLL_THREAD_ATTACH    2
#define DLL_THREAD_DETACH    3
#define DLL_PROCESS_DETACH   0
```

参数3： `PVOID Reserved` ：保留，固定为0

## 注册TLS回调函数

```C
#ifdef _WIN64                           //64位
    #pragma const_seg(&#34;.CRT$XLF&#34;)
    EXTERN_C const
#else
    #pragma data_seg(&#34;.CRT$XLF&#34;)        //32位
    EXTERN_C
#endif
PIMAGE_TLS_CALLBACK tls_callback_func[] = { tls_callback,0 };
#ifdef _WIN64                           //64位
    #pragma const_seg()
#else
    #pragma data_seg()                  //32位
#endif //_WIN64
```

`&#34;.CRT$XLF&#34;`的含义：

CRT表示使用C Runtime 机制

X表示 标识名随机

L表示 TLS Callback section

F也可以替换成B~Y的任意一个字符

## TLS回调函数实例：

```c
#include &lt;stdio.h&gt;
#include &lt;Windows.h&gt;

#ifdef _WIN64       //64位
#pragma comment (linker, &#34;/INCLUDE:_tls_used&#34;)  
#pragma comment (linker, &#34;/INCLUDE:tls_callback_func&#34;) 
#else               //32位
#pragma comment (linker, &#34;/INCLUDE:__tls_used&#34;) 
#pragma comment (linker, &#34;/INCLUDE:_tls_callback_func&#34;)
#endif

void print_console(char* szMsg)
{
    HANDLE hStdout = GetStdHandle(STD_OUTPUT_HANDLE);

    WriteConsoleA(hStdout, szMsg, strlen(szMsg), NULL, NULL);
}

void NTAPI TLS_CALLBACK(PVOID DllHandle, DWORD Reason, PVOID Reserved)
{
    char szMsg[80] = { 0, };

    switch (Reason)
    {
    case DLL_PROCESS_ATTACH: {
        wsprintfA(szMsg, &#34;DLL_PROCESS_ATTACH \n&#34;);
        print_console(szMsg);
        break;
    }
    case DLL_THREAD_ATTACH: {
        wsprintfA(szMsg, &#34;DLL_THREAD_ATTACH\n&#34;);
        print_console(szMsg);
        break;
    }
    case DLL_THREAD_DETACH: {
        wsprintfA(szMsg, &#34;DLL_THREAD_DETACH\n&#34;);
        print_console(szMsg);
        break;
    }
    case DLL_PROCESS_DETACH: {
        wsprintfA(szMsg, &#34;DLL_PROCESS_DETACH\n&#34;); 
        print_console(szMsg);
        break;
    }
    default:
        wsprintfA(szMsg, &#34;NOTHING\n&#34;); break;
    	print_console(szMsg);
    }
}


#ifdef _WIN64                           //64位
#pragma const_seg(&#34;.CRT$XLF&#34;)
EXTERN_C const
#else
#pragma data_seg(&#34;.CRT$XLF&#34;)        //32位
EXTERN_C
#endif
PIMAGE_TLS_CALLBACK tls_callback_func[] = { TLS_CALLBACK, 0 };
#ifdef _WIN64                           //64位
#pragma const_seg()
#else
#pragma data_seg()                  //32位
#endif //_WIN64


int main()
{
	printf(&#34;Is TLS successful ?\n&#34;);

	return 0;
}
```

## 注：

TLS函数中建议不要使用printf()函数，因为开启特定编译选项（/MT）编译源程序时，先于主线程调用执行的TLS回调函数中可能发生Run-Time Error（运行时错误）。此时可以直接调用`WriteConsole()` API来以防万一

# 相关资料

[TLS回调函数的学习](https://xz.aliyun.com/t/12057)

[**[调试逆向]** **【原创】反调试实战系列二 TLS反调试&#43;CheckRemoteDebuggerPresent原理**](https://www.52pojie.cn/thread-1490663-1-1.html)

以及相关[MSDN](learn.microsft.com)内的资料



---

> 作者: lo1see  
> URL: http://localhost:1313/posts/tls%E5%9B%9E%E8%B0%83%E5%87%BD%E6%95%B0/  

