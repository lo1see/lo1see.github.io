# LLVM学习——Hello MyPass


&lt;!--more--&gt;



# 第一个Pass——MyPass

## LLVM相关文件的相互转换命令

### C语言转

C语言转可执行文件

```bash
clang hello.c -o hello
```

![image-20241220205754782](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220205754782.png)

C语言转汇编

```bash
clang -S hello.c -o hello.asm
```

![image-20241220210257887](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220210257887.png)

C语言转`LLVM IR`

```bash
clang -emit-llvm -S hello.c -o hello.ll
```

![image-20241220210100185](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220210100185.png)

C语言转`LLVM Bitcode`

```bash
clang -emit-llvm -c hello.c -o hello.bc
```

![image-20241220210356815](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220210356815.png)



### 汇编转

汇编转可执行文件

```bash
clang hello.asm -o hello
```



### `LLVM IR`转

`LLVM IR`转汇编

```bash
clang -S hello.ll -o hello.s
```

`LLVM IR`转可执行文件

```bash
clang hello.ll -o hello
```

`LLVM IR`转`LLVM Bitcode`

```bash
clang -emit-llvm -c hello.ll -o hello.bc

llvm-as hello.ll -o hello.bc
```

执行`LLVM IR`

```bash
lli hello.ll
```



### `LLVM Bitcode`转

`LLVM Bitcode`转可执行文件

```bash
clang hello.bc -o hello
```

`LLVM Bitcode`转汇编

```bash
clang -S hello.bc -o hello.s

llc hello.bc -o hello.s
```

`LLVM Bitcode`转`LLVM IR`

```bash
llvm-dis hello.bc -o hello.ll
```

执行`LLVM Bitcode`

```bash
lli hello.bc
```



## 加载第一个`Pass`

在LLVM源码根目录下的Pass目录，即`llvm/lib/Transforms`目录下可以看到官方的Pass例子`Hello`，之后编写可以以此为例

![image-20241220213201003](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220213201003.png)

该Hello Pass编译好后在`build/lib`目录下可以找到`.so`文件

![image-20241220213309372](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220213309372.png)

测试一下这个Pass能不能加载

```bash
opt -load /home/loise/myDocu/LLVM/LLVM14.0.6/build/lib/LLVMHello.so -help |grep hello
```

![image-20241220213945861](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220213945861.png)

成功，以这段代码为例进行测试

```C
#include&lt;stdio.h&gt;

void test_hello1(){
    printf(&#34;test_hello1\n&#34;);
    return ;
}
void test_hello2(){
    printf(&#34;test_hello2\n&#34;);
    return ;
}
int main(int argc ,char const *argv[]){
    printf(&#34;hello clang\n&#34;);
    return 0;
}
```

先编译得到`LLVM Bitcode`

```bash
clang -emit-llvm -c hello.c -o hello.bc
```

再用`opt`工具进行测试

```bash
opt -load /home/loise/myDocu/LLVM/LLVM14.0.6/build/lib/LLVMHello.so --hello -enable-new-pm=0 hello.bc
```

![image-20241220214112086](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220214112086.png)

成功
`--hello`：是Pass注册的名称
`-enable-new-pm=0`：是关闭新的PassManager，使用旧的Pass管理器（the legacy pass manager）



## 编译

需要设置环境变量

```bash
vim ~/.bashrc
export LLVM_HOME=/mnt/c/Users/Qfrost/Desktop/code/LLVM/build    // 这个是生成的build目录
export PATH=${LLVM_HOME}/bin:$PATH
source ~/.bashrc
```



### 独立编译

独立编译一个Pass最简单的方法是

```bash
clang `llvm-config --cxxflags` `llvm-config --ldflags` -I /LLVM_CODE/llvm/include -I /LLVM_HOME/include -shared  -fPIC MyPass.cpp -o MyPass.so
```

### 用`CMakeLists.txt`编译

```cmake
cmake_minimum_required(VERSION 3.2)
if(NOT DEFINED ENV{LLVM_HOME})
    message(FATAL_ERROR &#34;$LLVM_HOME is not defined&#34;)
endif()
if(NOT DEFINED ENV{LLVM_DIR})
    set(ENV{LLVM_DIR} $ENV{LLVM_HOME}/lib/cmake/llvm)
endif()
find_package(LLVM REQUIRED CONFIG)
add_definitions(${LLVM_DEFINITIONS})
include_directories(${LLVM_INCLUDE_DIRS})
link_directories(${LLVM_LIBRARY_DIRS})

add_library(LLVMMyPass MODULE
    # List your source files here.
    MyPass.cpp
)
```

然后cmake &amp;&amp; make 同样可以在当前目录下生成LLVMMyPass.so文件

### 源码目录编译

在源码`llvm/lib/Transforms`的`Pass`目录下写自己的`Pass`代码和一个`CMakeLists.txt`

```cmake
add_library(LLVMMyPass MODULE
    # List your source files here.
    MyPass.cpp
)
```

然后修改上层的，即Transforms目录下的CMakeLists.txt，为其添加一行，将自己写的Pass目录添加进去

```cmake
add_subdirectory(MyPass)
```

最后对LLVM整个项目进行编译，因为这个Pass没有注册到clang里，其实只是对这个Pass编译了个so文件，很快，最后结果在`LLVM_HOME/lib`下



## 加载`MyPass`

`MyPass.cpp`：

```c&#43;&#43;
// MyPass
#include &#34;llvm/Pass.h&#34;
#include &#34;llvm/IR/Function.h&#34;
#include &#34;llvm/Support/raw_ostream.h&#34;

#include &#34;llvm/IR/LegacyPassManager.h&#34;
#include &#34;llvm/Transforms/IPO/PassManagerBuilder.h&#34;

using namespace llvm;

namespace {
  struct MyPass : public FunctionPass {
    static char ID;
    MyPass() : FunctionPass(ID) {}

    int i = 0;

    bool runOnFunction(Function &amp;F) override {
      errs() &lt;&lt; &#34;MyPass: &#34; &lt;&lt; i&#43;&#43; &lt;&lt; &#34; Function: &#34;;
      errs().write_escaped(F.getName()) &lt;&lt; &#39;\n&#39;;
      return false;
    }
  };
}

char MyPass::ID = 0;
static RegisterPass&lt;MyPass&gt; X(&#34;MyPass&#34;, &#34;Hello MyPass&#34;, false, false);

static RegisterStandardPasses Y1(
    PassManagerBuilder::EP_EarlyAsPossible,
    [](const PassManagerBuilder &amp;Builder,
       legacy::PassManagerBase &amp;PM) { PM.add(new MyPass()); });


```

用最简单的命令编译

```bash
clang `llvm-config --cxxflags` `llvm-config --ldflags` -I /LLVM_CODE/llvm/include -I /LLVM_HOME/include -shared  -fPIC MyPass.cpp -o MyPass.so
```

最后用`opt`命令加载

```bash
opt -load ./MyPass.so -MyPass -enable-new-pm=0 hello.bc
```

![image-20241220223304674](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220223304674.png)

如果没有配置`LLVM_HOME`的环境变量编译出来的so文件会有问题，不能加载自己的Pass


---

> 作者:   
> URL: http://localhost:1313/posts/8cffabc/  

