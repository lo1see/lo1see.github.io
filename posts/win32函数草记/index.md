# Win32函数草记










#### 随便写的帮助回忆的口语化的笔记，还想不起来就去learn.microsoft.com罢



# OpenProcess()

#### 函数体

```cpp
HANDLE OpenProcess(
  [in] DWORD dwDesiredAccess,
  [in] BOOL  bInheritHandle,
  [in] DWORD dwProcessId
);
```

#### 参数

```
[in] dwDesiredAccess
```

获得访问权限等级。



```
[in] bInheritHandle
```

基本没啥用

如果此值为 TRUE，则此进程创建的进程将继承该句柄。 否则，进程不会继承此句柄。

```
[in] dwProcessId
```

进程的标识符，就是pid之类的



# VirtualAllocEx()

#### 函数体

```cpp
LPVOID VirtualAllocEx(
  [in]           HANDLE hProcess,
  [in, optional] LPVOID lpAddress,
  [in]           SIZE_T dwSize,
  [in]           DWORD  flAllocationType,
  [in]           DWORD  flProtect
);
```

#### 参数

```
[in] hProcess
```

想要分配内存所指定的进程的句柄

句柄必须具有 **PROCESS_VM_OPERATION** 访问权限。

```
[in, optional] lpAddress
```

不知道，见了两次都是NULL



如果 *lpAddress* 为 **NULL**，则该函数确定分配区域的位置。

```
[in] dwSize
```

要分配的内存区域的大小（以字节为单位）。

如果 *lpAddress* 为 **NULL**，则该函数将 *dwSize* 舍入到下一页边界。

如果 *lpAddress* 不是 **NULL**，则该函数将分配包含 *lpAddress* 到 *lpAddress*&#43;*dwSize* 范围内一个或多个字节的所有页面。 例如，这意味着跨页边界的 2 字节范围会导致函数分配这两个页面。

```
[in] flAllocationType
```

| **MEM_COMMIT** 0x00001000 | 为指定的保留内存页分配内存费用， (磁盘上的总内存大小和分页文件) 。 该函数还保证当调用方稍后最初访问内存时，内容将为零。  除非/直到实际访问虚拟地址，否则不会分配实际物理页。 若要在一个步骤中保留和提交页面，请使用 `MEM_COMMIT | MEM_RESERVE` **调用 VirtualAllocEx**。 除非已保留整个范围，否则尝试通过指定**不带MEM_RESERVE MEM_COMMIT**来提交特定地址范围，并且非 **NULL***lpAddress* 将失败。 生成的错误代码 **ERROR_INVALID_ADDRESS**。 尝试提交已提交的页面不会使函数失败。 这意味着，无需首先确定每个页面的当前承诺状态即可提交页面。 如果 *lpAddress* 指定 enclave 中的地址，则必须**MEM_COMMIT***flAllocationType*。 |
| ------------------------- | ------------------------------------------------------------ |

```
[in] flProtect
```

页面区域的内存保护权限等级。

- PAGE_NOACCESS
- PAGE_GUARD
- PAGE_NOCACHE
- PAGE_WRITECOMBINE

# WriteProcessMemory()

#### 函数体

```
BOOL WriteProcessMemory(
  HANDLE hProcess,
  LPVOID lpBaseAddress,
  LPVOID lpBuffer,
  DWORD nSize,
  LPDWORD lpNumberOfBytesWritten
);
```

#### 参数

```
[in] HANDLE hProcess
```

OpenProcess()返回的句柄，用来知道要写入哪个文件

```
[in] LPVOID lpBaseAddress
```

指向指定进程中基地址的指针。就是指向刚刚VirtualAlloc分配出来的hProcess进程的内存初始地址的指针。

```
[in] LPVOID lpBuffer
```

指向缓冲区的指针。要把缓冲区里的数据写入上面的基地址里。

```
[in] DWORD nSize
```

指定请求写入指定进程的字节数。这里变量名字都是dwBufferSize。

```
[in] LPDWORD lpNumberOfBytesWritten
```

是可选的参数，见了几次都是NULL。            //指向传输到指定进程的字节数的指针。

# GetModuleHandle()和LoadLibrary()

这俩参数一样，都是类似&#34;kernel.dll&#34;之类的文件名，之后进行搜索然后加载。如果文件名没有后缀，自动添加.dll

### 区别

Loadlibrary是用来将普通的dll这个文件载入到系统进程中，使得其成为一个系统模块，而GetModuleHandle是需要在进程已经加载至内存获取其中dll的句柄。

# GetProcAddress()

#### 函数体

```cpp
FARPROC GetProcAddress(
  [in] HMODULE hModule,
  [in] LPCSTR  lpProcName
);
```

#### 参数

```
[in] HMODULE hModule
```

要寻找的函数的进程的句柄。比如寻找kernel.dll里的LoadLibrary函数，就要

```
hMod = GetModuleHandle(&#34;kernel32.dll&#34;);
pThreadProc = GetProcAddress(hMod, &#34;Loadlibrary&#34;)
```

 [LoadLibrary、LoadLibraryEx](https://learn.microsoft.com/zh-cn/windows/desktop/api/libloaderapi/nf-libloaderapi-loadlibrarya)、[LoadPackagedLibrary](https://learn.microsoft.com/zh-cn/windows/desktop/api/winbase/nf-winbase-loadpackagedlibrary) 或 [GetModuleHandle](https://learn.microsoft.com/zh-cn/windows/desktop/api/libloaderapi/nf-libloaderapi-getmodulehandlea) 函数返回此句柄。 

```
[in] lpProcName
```

你想找的函数名称，是个字符串。

# CreateRemoteThread()

#### 函数体

```cpp
HANDLE CreateRemoteThread(
  [in]  HANDLE                 hProcess,
  [in]  LPSECURITY_ATTRIBUTES  lpThreadAttributes,
  [in]  SIZE_T                 dwStackSize,
  [in]  LPTHREAD_START_ROUTINE lpStartAddress,
  [in]  LPVOID                 lpParameter,
  [in]  DWORD                  dwCreationFlags,
  [out] LPDWORD                lpThreadId
);
```

#### 参数

```
[in]  HANDLE hProcess
```

想创建线程所在的进程的句柄。该句柄需要有一定的权限，详见MSDN

```
[in] LPSECURITY_ATTRIBUTES lpThreadAttributes
```

见了两次都是NULL。

线程将获取默认的安全描述符，并且无法继承句柄

```
[in]  SIZE_T dwStackSize
```

见了两次这个参数都是0，使用PE文件的默认大小

如果此参数为 0 (零) ，则新线程将使用可执行文件的默认大小。

```
[in] lpStartAddress
```

&lt;font color = &#34;red&#34;&gt;这个我不懂，看不明白解释&lt;/font&gt;



指向由线程执行的 **LPTHREAD_START_ROUTINE** 类型的应用程序定义函数的指针，表示远程进程中线程的起始地址。 函数必须存在于远程进程中

```
[in] lpParameter
```

这些数据是原本存在缓冲区中的数据，这些数据通过WriteProcessMemory写入所给指针的内存，也是这里的指针

指向要传递给线程函数的变量的指针。

```
[in] dwCreationFlags
```

见了两次都是NULL

# GetThreadContext()

```cpp
BOOL GetThreadContext(
  [in]      HANDLE    hThread,
  [in, out] LPCONTEXT lpContext
);
```

可将指定线程(hThread)的CONTEXT存储到指定结构体变量(lpContext)中

# GetModuleFileNameA

```cpp
DWORD GetModuleFileNameA(
  [in, optional] HMODULE hModule,
  [out]          LPSTR   lpFilename,
  [in]           DWORD   nSize
);
```

若第一个参数为NULL，则检索当前进程的可执行文件的路径


---

> 作者: lo1see  
> URL: http://localhost:1313/posts/win32%E5%87%BD%E6%95%B0%E8%8D%89%E8%AE%B0/  

