# LLVM学习——环境搭建


&lt;!--more--&gt;

# LLVM环境搭建

忙忙忙最近好忙，学习了有段时间了，回过头来整理下

## 下载

官方 LLVM releases：https://releases.llvm.org/

github：https://github.com/llvm/llvm-project

因为后续找到的OLLVM版本是14.x的，所以这里下载LLVM 14进行编译

![image-20241220185412957](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220185412957.png)

拉到最下面下载源码

![image-20241220185435681](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220185435681.png)

移动到Ubuntu中解压

```bash
tar -xzvf llvm-project-llvmorg-14.0.6.tar.gz
```

![image-20241220185631740](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220185631740.png)



## 配置编译环境

确保机子符合编译环境要求

https://llvm.org/docs/GettingStarted.html#requirements

![image-20241220185750341](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220185750341.png)

除此以外还要安装

```bash
sudo apt-get install g&#43;&#43;
sudo apt-get install make
sudo apt-get install cmake
sudo apt install ninja-build
```

## 编译

进入与`llvm-project-llvmorg-14.0.6`同级的目录

```bash
mkdir build
cd build
cmake -G Ninja -DLLVM_ENABLE_PROJECTS=&#34;clang;lld&#34; -DLLVM_TARGETS_TO_BUILD=&#34;X86;ARM;AArch64&#34; -DCMAKE_BUILD_TYPE=Debug -DLLVM_ENABLE_RTTI=ON -DLLVM_INCLUDE_TESTS=OFF -DENABLE_LLVM_SHARED=ON ../llvm-project-llvmorg-14.0.6/llvm
```

`-G Ninja`：指定使用 Ninja 构建工具生成构建文件。

`-DLLVM_TARGETS_TO_BUILD`：指定要一起构建的子项目，默认情况下，LLVM 只构建核心库，不包含`clang`

`-DLLVM_TARGETS_TO_BUILD`：指定要支持的目标架构（后端），限定需要的几个架构可以少编译不少东西，不限定默认编译全部，那样会多占几十GB的硬盘

`../llvm-project-llvmorg-14.0.6/llvm`：指定源代码目录

`-DCMAKE_BUILD_TYPE`：设置编译类型，如果只是编译OLLVM可以设置为`Release`。如果要自己写pass调试，一定一定要设置为`Debug`版，不然不能调试。（第一次编译`Release`弄了两天才知道区别天塌了。编译`Debug`版巨吃性能、硬盘和内存，我Ubuntu虚拟机分配了16核、25GB物理内存、25GB交换内存，160GB硬盘，最后高峰只能用单线程，编译了3个小时产生了近100GB的文件。期间有一次重新编译Ubuntu虚拟机占了350GB把硬盘占满了虚拟机寄了只能重装虚拟机😭😭😭）

最后

```bash
ninja -j8
```

最后链接阶段会爆内存，内存够大就多给些内存，内存给不够大了就只开单线程`-j1`罢，不过也会占用几十个GB

编译成功

![image-20241220192753312](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220192753312.png)

## 测试Clang

给clang添加环境变量

![image-20241220192941275](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220192941275.png)

到自己的路径下写一个`helloworld`测试一下编译出来的`clang`

![image-20241220203853219](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220203853219.png)


---

> 作者:   
> URL: http://localhost:1313/posts/e1fb916/  

