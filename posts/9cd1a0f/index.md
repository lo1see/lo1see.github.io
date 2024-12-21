# LLVM学习——OLLVM混淆配置


&lt;!--more--&gt;



# OLLVM环境搭建

## 移植OLLVM到LLVM中

OLLVM官方项目：https://github.com/obfuscator-llvm/obfuscator/tree/llvm-4.0

但是只支持到了LLVM 4，太旧了，需要移植到最新的LLVM中才能使用（或者就使用LLVM 4）

参考文章：[【清羽】Windows10下编译OLLVM-14.x_ollvm 编译优化后-CSDN博客](https://blog.csdn.net/qq_41923691/article/details/123258565)

这个师傅有移植好的OLLVM14：https://github.com/buffcow/ollvm-project/tree/14.x

可以直接下载编译这份LLVM14.0.0，或者看Commits看看是如何进行的移植

![image-20241221160335977](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221160335977.png)

中间的几个

![image-20241221160502104](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221160502104.png)

把cpp文件放到Pass目录下`llvm/lib/Transforms/`

![image-20241221160541580](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221160541580.png)

把头文件放到Pass的头文件目录下`llvm/include/llvm/Transforms/`

![image-20241221160713844](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221160713844.png)

对着这个改

![image-20241221160853324](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221160853324.png)

把OLLVM注册到LLVM中

![image-20241221160832348](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221160832348.png)

最后进入与`llvm-project-llvmorg-14.0.6`同级的目录，如最初的方法一样重新编译

```bash
cd build
cmake -G Ninja -DLLVM_ENABLE_PROJECTS=&#34;clang;lld&#34; -DLLVM_TARGETS_TO_BUILD=&#34;X86;ARM;AArch64&#34; -DCMAKE_BUILD_TYPE=Debug -DLLVM_ENABLE_RTTI=ON -DLLVM_INCLUDE_TESTS=OFF -DENABLE_LLVM_SHARED=ON ../llvm-project-llvmorg-14.0.6/llvm
```

这样就把OLLVM编译到clang里面可以正常使用了



## 脱离源码构建`OLLVM.so`

构建一个和源码目录下相同的文件树

![image-20241221162154284](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221162154284.png)

`ollvm2/ollvm/lib/Transforms/Obfuscation/CMakeLists.txt`，这里还有个`entry.cpp`，是用来把命令行注册到`clang`的

![image-20241221162459450](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221162459450.png)

```cmake
add_llvm_component_library(LLVMOllvm MODULE
  CryptoUtils.cpp
  Substitution.cpp
  StringObfuscation.cpp
  BogusControlFlow.cpp
  Utils.cpp
  SplitBasicBlocks.cpp
  Flattening.cpp

  entry.cpp

)

add_dependencies(LLVMOllvm  intrinsics_gen)
```

`ollvm2/ollvm/lib/Transforms/CMakeLists.txt`

![image-20241221162516137](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221162516137.png)

```cmake
add_subdirectory(Obfuscation)
```

`ollvm2/ollvm/lib/CMakeLists.txt`

![](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221162548968.png)

```cmake
add_subdirectory(Transforms)
```

`ollvm2/ollvm/CMakeLists.txt`

![image-20241221162624997](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221162624997.png)

```cmake
# 设置项目名和版本
cmake_minimum_required(VERSION 3.10)
project(MyProject VERSION 1.0)

# 设置 C&#43;&#43; 标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# 设置头文件搜索路径
include_directories(include)

# 添加子目录
add_subdirectory(lib)
```

`ollvm2/CMakeLists.txt`

![image-20241221163000842](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221163000842.png)

```cmake
# 指定最低的 CMake 版本要求为 3.26
cmake_minimum_required(VERSION 3.10)

# 定义项目名称为 &#34;ollvm&#34;
project(llvm)
# 设置 LLVM 的路径，用于在后续的 find_package 中定位 LLVM
set(LLVM_DIR /home/loise/myDocu/LLVM/LLVM14.0.6/build/lib/cmake/llvm/)
# 使用 find_package 查找 LLVM，并使用 CONFIG 模式定位其配置信息
find_package(LLVM REQUIRED CONFIG)
# 将 LLVM 的 CMake 模块路径添加到 CMAKE_MODULE_PATH 中
list(APPEND CMAKE_MODULE_PATH &#34;${LLVM_CMAKE_DIR}&#34;)
# 引入 LLVM 的辅助功能
include(AddLLVM)
# 添加 LLVM 的定义到项目中
add_definitions(${LLVM_DEFINITIONS})
# 添加 LLVM 的头文件路径到项目中
include_directories(${LLVM_INCLUDE_DIRS})
# 添加项目自己的头文件路径（ollvm/include）
include_directories(ollvm/include)
# 在项目中添加子目录 &#34;ollvm&#34;，该目录下应包含项目源代码
add_subdirectory(ollvm)
```

前面提到的在`OLLVM Pass`源码目录下的`entry.cpp`

```C&#43;&#43;
#include &#34;llvm/Transforms/Obfuscation/BogusControlFlow.h&#34;
#include &#34;llvm/Transforms/Obfuscation/Flattening.h&#34;
#include &#34;llvm/Transforms/Obfuscation/Split.h&#34;
#include &#34;llvm/Transforms/Obfuscation/Substitution.h&#34;
#include &#34;llvm/Transforms/Obfuscation/CryptoUtils.h&#34;

#include &#34;llvm/IR/LegacyPassManager.h&#34;
#include &#34;llvm/Transforms/IPO/PassManagerBuilder.h&#34;

#include &#34;llvm/Transforms/Utils.h&#34;

using namespace llvm;

// Flags for obfuscation
static cl::opt&lt;bool&gt; Flattening(&#34;fla&#34;, cl::init(false),
    cl::desc(&#34;Enable the flattening pass&#34;));

static cl::opt&lt;bool&gt; BogusControlFlow(&#34;bcf&#34;, cl::init(false),
    cl::desc(&#34;Enable bogus control flow&#34;));

static cl::opt&lt;bool&gt; Substitution(&#34;sub&#34;, cl::init(false),
    cl::desc(&#34;Enable instruction substitutions&#34;));

static cl::opt&lt;std::string&gt; AesSeed(&#34;aesSeed&#34;, cl::init(&#34;&#34;),
    cl::desc(&#34;seed for the AES-CTR PRNG&#34;));

static cl::opt&lt;bool&gt; Split(&#34;split&#34;, cl::init(false),
    cl::desc(&#34;Enable basic block splitting&#34;));

static llvm::RegisterStandardPasses Y(
    llvm::PassManagerBuilder::EP_EarlyAsPossible,
    [](const llvm::PassManagerBuilder &amp;Builder,
    llvm::legacy::PassManagerBase &amp;PM) {

        if(!AesSeed.empty()) {
            if(!llvm::cryptoutils-&gt;prng_seed(AesSeed.c_str()))
            exit(1);
        }

        PM.add(createSplitBasicBlock(Split));
        PM.add(createBogus(BogusControlFlow));
        if (Flattening) {
            PM.add(createLowerSwitchPass());
        }

        PM.add(createFlattening(Flattening));
        PM.add(createSubstitution(Substitution));
});
```



给clang设置环境变量，然后就可以用类似如下的命令加载Pass玩了

```bash
clang -Xclang -load -Xclang /home/loise/myDocu/LLVM/myPass/flattening/cmake-build-debug/ollvm/lib/Transforms/Obfuscation/LLVMOllvm.so -flegacy-pass-manager -mllvm -fla  /home/loise/myDocu/LLVM/myPass/flattening/src/hello.cpp -o /home/loise/myDocu/LLVM/myPass/flattening/src/hello
```

把这样的参数放入CLion的运行/调试设置里就可以很方便的编译运行了

![image-20241221163740983](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221163740983.png)



## 使用参数说明

1. `-fla` 用于 [控制流平坦化（control flow flattening）](https://github.com/obfuscator-llvm/obfuscator/wiki/Control-Flow-Flattening) 的Pass
   - `-mllvm -fla`: 激活控制流平坦化
   - `-mllvm -split`: 激活基本块分割。一起使用时可提高平坦度。
   - `-mllvm -split_num=3`: 如果激活这个Pass，则在每个基本块上应用 3 次。默认值：1
2. `-sub` 用于 [指令替换（instruction substitution）](https://github.com/obfuscator-llvm/obfuscator/wiki/Instructions-Substitution) 的Pass
   - `-mllvm -sub`: 激活指令替换
   - `-mllvm -sub_loop=3`: 如果这个Pass被激活，则每个函数使用 3 次。默认值：1。
3. `-bcf` 用于 [虚假控制流（bogus control flow）](https://github.com/obfuscator-llvm/obfuscator/wiki/Bogus-Control-Flow) 的Pass
   - `-mllvm -bcf`: 激活虚假控制流
   - `-mllvm -bcf_loop=3`: 如果这个Pass被激活，则每个函数使用 3 次。默认值：1。
   - `-mllvm -bcf_prob=40`: 如果激活这个Pass，基本区块将以 40% 的概率被混淆。默认值：30%


---

> 作者:   
> URL: http://localhost:1313/posts/9cd1a0f/  

