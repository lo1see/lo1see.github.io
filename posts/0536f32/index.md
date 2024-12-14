# X64dbg插件HelloWorld编写


&lt;!--more--&gt;

# x64dbg插件开发笔记

## 1.环境配置

解压x64dbg工具包，可以得到pluginsdk文件夹，这是开发插件所需要的SDK包

![](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212204504170.png)

新建一个空项目，创建`pluginmain.h`和`pluginmain.cpp`

![](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241214190145886.png)

打开`项目-&gt;xx项目属性`，在`配置属性-&gt;C/C&#43;&#43;-&gt;常规`中的`附加包含目录`中添加SDK文件夹，添加库函数

![image-20241212204619299](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212204619299.png)

打开`项目-&gt;xx项目属性`，在`配置属性-&gt;链接器-&gt;常规`中的`附加包含库目录`中添加SDK文件夹，添加lib静态链接库

![image-20241212204834661](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241212204834661.png)

修改`配置属性-&gt;常规`中的`配置类型`为`动态库(.dll)`

![image-20241214190340112](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241214190340112.png)



## 2.HelloWorld插件

x64dbg官方插件模板：https://github.com/x64dbg/PluginTemplate

其他都有点复杂，先CV一份现成的helloworld看看样子

pluginmain.h

```C&#43;&#43;
#pragma once
#pragma once

// Plugin information
#define PLUGIN_NAME &#34;Hello_lo1see&#34;
#define PLUGIN_VERSION 1

#include &#34;./bridgemain.h&#34;
#include &#34;./_plugins.h&#34;

#include &#34;./_scriptapi_argument.h&#34;
#include &#34;./_scriptapi_assembler.h&#34;
#include &#34;./_scriptapi_bookmark.h&#34;
#include &#34;./_scriptapi_comment.h&#34;
#include &#34;./_scriptapi_debug.h&#34;
#include &#34;./_scriptapi_flag.h&#34;
#include &#34;./_scriptapi_function.h&#34;
#include &#34;./_scriptapi_gui.h&#34;
#include &#34;./_scriptapi_label.h&#34;
#include &#34;./_scriptapi_memory.h&#34;
#include &#34;./_scriptapi_misc.h&#34;
#include &#34;./_scriptapi_module.h&#34;
#include &#34;./_scriptapi_pattern.h&#34;
#include &#34;./_scriptapi_register.h&#34;
#include &#34;./_scriptapi_stack.h&#34;
#include &#34;./_scriptapi_symbol.h&#34;

#include &#34;./DeviceNameResolver/DeviceNameResolver.h&#34;
#include &#34;./jansson/jansson.h&#34;
#include &#34;./lz4/lz4file.h&#34;
#include &#34;./TitanEngine/TitanEngine.h&#34;
#include &#34;./XEDParse/XEDParse.h&#34;

#ifdef _WIN64
#pragma comment(lib, &#34;./x64dbg.lib&#34;)
#pragma comment(lib, &#34;./x64bridge.lib&#34;)
#pragma comment(lib, &#34;./DeviceNameResolver/DeviceNameResolver_x64.lib&#34;)
#pragma comment(lib, &#34;./jansson/jansson_x64.lib&#34;)
#pragma comment(lib, &#34;./lz4/lz4_x64.lib&#34;)
#pragma comment(lib, &#34;./TitanEngine/TitanEngine_x64.lib&#34;)
#pragma comment(lib, &#34;./XEDParse/XEDParse_x64.lib&#34;)
#else
#pragma comment(lib, &#34;./x32dbg.lib&#34;)
#pragma comment(lib, &#34;./x32bridge.lib&#34;)
#pragma comment(lib, &#34;./DeviceNameResolver/DeviceNameResolver_x86.lib&#34;)
#pragma comment(lib, &#34;./jansson/jansson_x86.lib&#34;)
#pragma comment(lib, &#34;./lz4/lz4_x86.lib&#34;)
#pragma comment(lib, &#34;./TitanEngine/TitanEngine_x86.lib&#34;)
#pragma comment(lib, &#34;./XEDParse/XEDParse_x86.lib&#34;)
#endif //_WIN64

#define Cmd(x) DbgCmdExecDirect(x)
#define Eval(x) DbgValFromString(x)
#define dprintf(x, ...) _plugin_logprintf(&#34;[&#34; PLUGIN_NAME &#34;] &#34; x, __VA_ARGS__)
#define dputs(x) _plugin_logprintf(&#34;[&#34; PLUGIN_NAME &#34;] %s\n&#34;, x)
#define PLUG_EXPORT extern &#34;C&#34; __declspec(dllexport)

//superglobal variables
extern int pluginHandle;
extern HWND hwndDlg;
extern int hMenu;
extern int hMenuDisasm;
extern int hMenuDump;
extern int hMenuStack;

//functions
bool pluginInit(PLUG_INITSTRUCT* initStruct);
void pluginStop();
void pluginSetup();

```

pluginmain.cpp

```C&#43;&#43;
#include &#34;pluginmain.h&#34;
#include &lt;Windows.h&gt;
#include &lt;process.h&gt;

int pluginHandle;
HWND hwndDlg;
int hMenu;
int hMenuDisasm;
int hMenuDump;
int hMenuStack;

// 导出函数
extern &#34;C&#34; __declspec(dllexport) void CBMENUENTRY(CBTYPE cbType, PLUG_CB_MENUENTRY * info);
extern &#34;C&#34; __declspec(dllexport) void plugsetup(PLUG_SETUPSTRUCT * setupStruct);
extern &#34;C&#34; __declspec(dllexport) bool pluginit(PLUG_INITSTRUCT * initStruct);

// 在这里初始化插件数据。
bool pluginInit(PLUG_INITSTRUCT* initStruct)
{
    // 返回false以取消加载插件。
    return true;
}

// 在此处取消初始化插件数据。
void pluginStop()
{
}

// 在这里做GUI/菜单相关的事情。
void pluginSetup()
{
}

// 菜单被点击回调
void CBMENUENTRY(CBTYPE cbType, PLUG_CB_MENUENTRY* info)
{
    switch (info-&gt;hEntry)
    {
    case 1:    //注册时填的菜单ID
        MessageBoxA(0, &#34;插件 1 运行成功咯&#34;, &#34;hello lo1see&#34;, 0);
        break;
    case 2:
        MessageBoxA(0, &#34;插件 2 运行成功咯&#34;, &#34;hello lo1see&#34;, 0);
        break;
    case 3:
        // 此菜单用于实现功能，并测试
        MessageBoxA(0, &#34;插件 3 运行成功咯&#34;, &#34;hello lo1see&#34;, 0);
    default:
        break;
    }
}

PLUG_EXPORT bool pluginit(PLUG_INITSTRUCT* initStruct)
{
    initStruct-&gt;pluginVersion = PLUGIN_VERSION;
    initStruct-&gt;sdkVersion = PLUG_SDKVERSION;
    strncpy_s(initStruct-&gt;pluginName, PLUGIN_NAME, _TRUNCATE);
    pluginHandle = initStruct-&gt;pluginHandle;

    // 插件初始化
    initStruct-&gt;sdkVersion = PLUG_SDKVERSION;
    initStruct-&gt;pluginVersion = 1;
    const char* name = &#34;CheckPlugin --&gt;&#34;;
    memset(initStruct-&gt;pluginName, 0, 128);
    memcpy(initStruct-&gt;pluginName, name, strlen(name));

    return pluginInit(initStruct);
}

PLUG_EXPORT bool plugstop()
{
    pluginStop();
    return true;
}

PLUG_EXPORT void plugsetup(PLUG_SETUPSTRUCT* setupStruct)
{
    hwndDlg = setupStruct-&gt;hwndDlg;
    hMenu = setupStruct-&gt;hMenu;
    hMenuDisasm = setupStruct-&gt;hMenuDisasm;
    hMenuDump = setupStruct-&gt;hMenuDump;
    hMenuStack = setupStruct-&gt;hMenuStack;

    // 增加二级菜单
    char sub_menu01[] = { &#34;Hello Lo1see 01&#34; };
    char sub_menu02[] = { &#34;Hello Lo1see 02&#34; };
    char sub_menu03[] = { &#34;Hello Lo1see 03&#34; };
    _plugin_menuaddentry(setupStruct-&gt;hMenu, 1, sub_menu01);
    _plugin_menuaddentry(setupStruct-&gt;hMenu, 2, sub_menu02);
    _plugin_menuaddentry(setupStruct-&gt;hMenu, 3, sub_menu03);

    pluginSetup();
}

```

编译成功后得到的.dll文件，根据不同版本需要将其更改为`*.dp32`或者`*.dp64`以此来代表这是一个插件，并将更改好的插件放入到`x32/plugins`目录下，重启`x64dbg`至此即可看到插件已经被加载成功。

![](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241214192207849.png)

### 基本导出函数

是作为xdbg插件必须导出的函数，供xdbg来调用

#### pluginit

```C&#43;&#43;
extern &#34;C&#34; __declspec(dllexport) bool pluginit(PLUG_INITSTRUCT * initStruct);
```

需要在pluginit函数中填充参数和基本信息

```C&#43;&#43;
PLUG_EXPORT bool pluginit(PLUG_INITSTRUCT* initStruct)
{
    initStruct-&gt;pluginVersion = PLUGIN_VERSION;
    initStruct-&gt;sdkVersion = PLUG_SDKVERSION;
    strncpy_s(initStruct-&gt;pluginName, PLUGIN_NAME, _TRUNCATE);
    pluginHandle = initStruct-&gt;pluginHandle;

    // 插件初始化
    initStruct-&gt;sdkVersion = PLUG_SDKVERSION;
    initStruct-&gt;pluginVersion = 1;
    const char* name = &#34;CheckPlugin --&gt;&#34;;
    memset(initStruct-&gt;pluginName, 0, 128);
    memcpy(initStruct-&gt;pluginName, name, strlen(name));

    return pluginInit(initStruct);
}
```

#### plugsetup和plugstop

- plugstop：插件被移除的时候被调用，可以用来清除注册的回调和命令，清理资源
- plugsetup：当插件初始化成功的时候调用，可以在这里添加菜单、做其他的界面相关的事情

```C&#43;&#43;
PLUG_EXPORT bool plugstop()
{
    pluginStop();
    return true;
}

PLUG_EXPORT void plugsetup(PLUG_SETUPSTRUCT* setupStruct)
{
    hwndDlg = setupStruct-&gt;hwndDlg;
    hMenu = setupStruct-&gt;hMenu;
    hMenuDisasm = setupStruct-&gt;hMenuDisasm;
    hMenuDump = setupStruct-&gt;hMenuDump;
    hMenuStack = setupStruct-&gt;hMenuStack;

    // 增加二级菜单
    char sub_menu01[] = { &#34;Hello Lo1see 01&#34; };
    char sub_menu02[] = { &#34;Hello Lo1see 02&#34; };
    char sub_menu03[] = { &#34;Hello Lo1see 03&#34; };
    _plugin_menuaddentry(setupStruct-&gt;hMenu, 1, sub_menu01);
    _plugin_menuaddentry(setupStruct-&gt;hMenu, 2, sub_menu02);
    _plugin_menuaddentry(setupStruct-&gt;hMenu, 3, sub_menu03);

    pluginSetup();
}
```

也可以什么都不做，非必须

#### 事件回调函数

```C&#43;&#43;
extern &#34;C&#34; __declspec(dllexport) void CBINITDEBUG(CBTYPE cbType, PLUG_CB_INITDEBUG* info); //初始化调试
extern &#34;C&#34; __declspec(dllexport) void CBSTOPDEBUG(CBTYPE cbType, PLUG_CB_STOPDEBUG* info); //停止调试
extern &#34;C&#34; __declspec(dllexport) void CBEXCEPTION(CBTYPE cbType, PLUG_CB_EXCEPTION* info); //异常
extern &#34;C&#34; __declspec(dllexport) void CBDEBUGEVENT(CBTYPE cbType, PLUG_CB_DEBUGEVENT* info); //调试事件
extern &#34;C&#34; __declspec(dllexport) void CBMENUENTRY(CBTYPE cbType, PLUG_CB_MENUENTRY* info); //点击子菜单
```

参数中所用到的结构体可以在这里找到：https://help.x64dbg.com/en/latest/developers/plugins/Callbacks/index.html

只要是满足CB*的导出函数，且属于这里面的类型https://help.x64dbg.com/en/latest/developers/plugins/API/registercallback.html，应该都可以注册成功

本例只实现了点击子菜单，弹窗显示`hello world`

```C&#43;&#43;
// 菜单被点击回调
void CBMENUENTRY(CBTYPE cbType, PLUG_CB_MENUENTRY* info)
{
    switch (info-&gt;hEntry)
    {
    case 1:    //注册时填的菜单ID
        MessageBoxA(0, &#34;插件 1 运行成功咯&#34;, &#34;hello lo1see&#34;, 0);
        break;
    case 2:
        MessageBoxA(0, &#34;插件 2 运行成功咯&#34;, &#34;hello lo1see&#34;, 0);
        break;
    case 3:
        // 此菜单用于实现功能，并测试
        MessageBoxA(0, &#34;插件 3 运行成功咯&#34;, &#34;hello lo1see&#34;, 0);
    default:
        break;
    }
}
```

## 3.库函数

如果有需要实现的功能再查文档找找api罢

文档：https://help.x64dbg.com/en/latest/developers/functions/debug/index.html


---

> 作者:   
> URL: http://localhost:1313/posts/0536f32/  

