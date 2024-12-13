# 加密与解密系统篇笔记




# 内核函数主要前缀

- `Ex`：管理层。&#34;`Ex`&#34;是&#34;`Executive`&#34;的开头两个字母
- `Ke`：核心层。&#34;`Ke`&#34;是&#34;`Kernel`&#34;的开头两个字母
- `Hal`：硬件抽象层。&#34;`Hal`&#34;是&#34;`Hardware Abstraction Layer`&#34;的缩写
- `Ob`：对象管理。&#34;`Ob`&#34;是&#34;`Object`&#34;的开头两个字母
- `Mm`：内存管理。&#34;`Mm`&#34;是&#34;`Memory Manager`&#34;的缩写
- `Ps`：进程（线程）管理。&#34;`Ps`&#34;是&#34;`Process`&#34;
- `Se`：安全管理。&#34;`Se`&#34;是&#34;`Security`&#34;的开头两个字母
- `Io`：`I/O`管理
- `Fs`：文件系统。&#34;`Fs`&#34;是&#34;`File System`&#34;的缩写
- `Cc`：文件缓存管理。&#34;`Cc`&#34;表示&#34;`Cache`&#34;
- `Cm`：系统配置管理。&#34;`Cm`&#34;是&#34;`Configuration Manager`&#34;的缩写
- `Pp`：即插即用管理。&#34;`Pp`&#34;表示&#34;`Pnp`&#34;
- `Rtl`：运行时程序库。&#34;`Rtl`&#34;是&#34;`Runtime Library`&#34;的缩写
- `Zw/Nt`：对应于`SSDT`中的服务函数，例如与文件或者注册表相关的操作函数
- `Flt`：`Minifilter`文件过滤驱动中调用的函数
- `Ndis`：`Ndis`网络框架中调用的函数

# 异常处理

## SEH

常见异常产生原因

| 异常产生原因                  | 对应值     | 说明                         |
| ----------------------------- | ---------- | ---------------------------- |
| STATUS_GUARD_PAGE_VIOLATION   | 080000001h | 读写属性为PAGE_GUARD的页面   |
| EXCEPTION_VREAKPOINT          | 080000003h | 断点异常                     |
| EXCEPTION_SINGLE_STEP         | 080000004h | 单步中断                     |
| EXCEPTION_INVALID_HANDLE      | 0C0000008h | 向一个函数传递了一个无效句柄 |
| EXCEPTION_ACCESS_VIOLATION    | 0C0000005h | 读写内存冲突                 |
| EXCEPTION_ILLEGAL_INSTRUCTION | 0C000001Dh | 遇到无效指令                 |
| EXCEPTION_INPAGE_ERROR        | 0C0000006h | 存取不存在的页面             |
| EXCEPTION_INT_DIVIDE_BY_ZERO  | 0C0000094h | 除零错误                     |
| EXCEPTION_STACK_OVERFLOW      | 0C00000FDh | 栈溢出                       |



---

> 作者: lo1see  
> URL: http://localhost:1313/posts/%E5%8A%A0%E5%AF%86%E4%B8%8E%E8%A7%A3%E5%AF%86%E7%B3%BB%E7%BB%9F%E7%AF%87%E7%AC%94%E8%AE%B0/  

