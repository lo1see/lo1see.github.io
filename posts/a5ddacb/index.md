# LLVM学习——调试开发环境配置


&lt;!--more--&gt;



# LLVM调试环境配置

我这里使用了VScode作为调试开发的IDE，貌似用CLion可能会更轻松方便一些，不过都可以用。

参考Qfrost  Q神的博客Orzzzzzz：	[LLVM 调试环境配置 – - Qfrost -](http://www.qfrost.com/posts/llvm/llvm调试环境配置/)

须要先以Debug模式编译好LLVM

## 环境配置

这是文件结构，`settings.json`是`vscode`自己生成的，不用在意

![image-20241220224209551](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220224209551.png)

要在当前开发目录下创建`.vscode`文件夹，里面存储三个重要的配置文件，如下，打注释的是需要手动修改的

### `c_cpp_properties.json`

```json
// c_cpp_properties.json  用来配置要包含的头文件和宏
{
    &#34;configurations&#34;: [
        {
            &#34;name&#34;: &#34;Linux&#34;,
            &#34;includePath&#34;: [
                &#34;${workspaceFolder}/**&#34;,
                &#34;/home/loise/myDocu/LLVM/LLVM14.0.6/llvm-project-llvmorg-14.0.6/llvm/include&#34;,       // 这里设置为LLVM源码目录的include目录
                &#34;/home/loise/myDocu/LLVM/LLVM14.0.6/build/include&#34;                       // 这里设置为LLVM_HOME的include目录
            ],
            &#34;defines&#34;: [
                &#34;_GNU_SOURCE&#34;,
                &#34;__STDC_CONSTANT_MACROS&#34;,
                &#34;__STDC_FORMAT_MACROS&#34;,
                &#34;__STDC_LIMIT_MACROS&#34;
            ],
            &#34;compilerPath&#34;: &#34;/home/loise/myDocu/LLVM/LLVM14.0.6/build/bin/clang&#34;,        // 这里设置为clang的绝对路径（vscode不加载环境变量，必须要用绝对路径）
            &#34;cStandard&#34;: &#34;c17&#34;,
            &#34;cppStandard&#34;: &#34;c&#43;&#43;14&#34;,
            &#34;intelliSenseMode&#34;: &#34;linux-clang-x64&#34;,
            &#34;compilerArgs&#34;: [
                &#34;-fno-exceptions&#34;,
                &#34;-fno-rtti&#34;
            ]
        }
    ],
    &#34;version&#34;: 4
}
```

### `launch.json`

```json
// launch.json   配置调试设置
{
    &#34;version&#34;: &#34;0.2.0&#34;,
    &#34;configurations&#34;: [
        {
            &#34;name&#34;: &#34;Debug Pass&#34;,
            &#34;type&#34;: &#34;cppdbg&#34;,
            &#34;request&#34;: &#34;launch&#34;,
            &#34;program&#34;: &#34;/home/loise/myDocu/LLVM/LLVM14.0.6/build/bin/opt&#34;,       // 这里设置为opt的绝对路径
            &#34;args&#34;: [
                &#34;-load&#34;,
                &#34;${fileDirname}/build/${fileBasenameNoExtension}.so&#34;,
                &#34;--${fileBasenameNoExtension}&#34;,
                &#34;-enable-new-pm=0&#34;,
                &#34;${fileDirname}/build/1-target_before.bc&#34;,
                &#34;-o&#34;,
                &#34;${fileDirname}/build/2-target_after.bc&#34;,
            ],
            &#34;stopAtEntry&#34;: false,
            &#34;cwd&#34;: &#34;${fileDirname}&#34;,
            &#34;environment&#34;: [],
            &#34;externalConsole&#34;: false,
            &#34;MIMode&#34;: &#34;gdb&#34;,
            &#34;setupCommands&#34;: [
                {
                    &#34;description&#34;: &#34;Enable pretty-printing for gdb&#34;,
                    &#34;text&#34;: &#34;-enable-pretty-printing&#34;,
                    &#34;ignoreFailures&#34;: true
                }
            ],
            &#34;sourceFileMap&#34;:{
                // &#34;./&#34;: &#34;&#34;
            },
            &#34;preLaunchTask&#34;: &#34;build&#34;,
        }
    ]
}
```

### `tasks.json`

```json
// tasks.json
{
    &#34;version&#34;: &#34;2.0.0&#34;,
    &#34;tasks&#34;: [
        {
            &#34;label&#34;: &#34;build&#34;,       // 设置build任务 即 make build -j2
            &#34;type&#34;: &#34;shell&#34;,
            &#34;command&#34;: &#34;make&#34;,     
            &#34;args&#34;: [
                &#34;build&#34;,
                &#34;-j2&#34;,
            ],
            &#34;options&#34;: {
                &#34;cwd&#34;: &#34;${workspaceFolder}&#34;,
                &#34;env&#34;: {
                    &#34;FILE&#34;: &#34;${relativeFile}&#34;,
                    &#34;FILE_DIR_NAME&#34;: &#34;${fileDirname}&#34;,
                    &#34;FILE_EXT_NAME&#34;: &#34;${fileExtname}&#34;,
                    &#34;FILE_BASENAME_NO_EXTENSION&#34;: &#34;${fileBasenameNoExtension}&#34;
                }
            },
            &#34;isBackground&#34;: false,
            &#34;presentation&#34;: {
                &#34;echo&#34;: true,
                &#34;reveal&#34;: &#34;always&#34;,
                &#34;focus&#34;: false,
                &#34;panel&#34;: &#34;dedicated&#34;,
                &#34;showReuseMessage&#34;: true,
                &#34;clear&#34;: true
            },
            &#34;problemMatcher&#34;: {
                &#34;owner&#34;: &#34;cpp&#34;,
                &#34;fileLocation&#34;: [
                    &#34;absolute&#34;
                ],
                &#34;pattern&#34;: {
                    &#34;regexp&#34;: &#34;^(.*):(\\d&#43;):(\\d&#43;):\\s&#43;(warning|error):\\s&#43;(.*)$&#34;,
                    &#34;file&#34;: 1,
                    &#34;line&#34;: 2,
                    &#34;column&#34;: 3,
                    &#34;severity&#34;: 4,
                    &#34;message&#34;: 5
                }
            }
        },
        {
            &#34;label&#34;: &#34;run&#34;,     // 设置run任务
            &#34;type&#34;: &#34;shell&#34;,
            &#34;command&#34;: &#34;make&#34;,
            &#34;group&#34;: {
                &#34;kind&#34;: &#34;build&#34;,
                &#34;isDefault&#34;: true
            },
            &#34;args&#34;: [
                &#34;run&#34;,
                &#34;-j2&#34;
            ],
            &#34;options&#34;: {
                &#34;cwd&#34;: &#34;${workspaceFolder}&#34;,
                &#34;env&#34;: {
                    &#34;FILE&#34;: &#34;${relativeFile}&#34;,
                    &#34;FILE_DIR_NAME&#34;: &#34;${fileDirname}&#34;,
                    &#34;FILE_EXT_NAME&#34;: &#34;${fileExtname}&#34;,
                    &#34;FILE_BASENAME_NO_EXTENSION&#34;: &#34;${fileBasenameNoExtension}&#34;
                }
            },
            &#34;isBackground&#34;: false,
            &#34;presentation&#34;: {
                &#34;echo&#34;: true,
                &#34;reveal&#34;: &#34;always&#34;,
                &#34;focus&#34;: false,
                &#34;panel&#34;: &#34;dedicated&#34;,
                &#34;showReuseMessage&#34;: true,
                &#34;clear&#34;: true
            },
            &#34;problemMatcher&#34;: []
        },
        {
            &#34;label&#34;: &#34;clean&#34;,       // 设置clean任务
            &#34;type&#34;: &#34;shell&#34;,
            &#34;command&#34;: &#34;make&#34;,
            &#34;args&#34;: [
                &#34;clean&#34;
            ],
            &#34;options&#34;: {
                &#34;cwd&#34;: &#34;${workspaceFolder}&#34;,
                &#34;env&#34;: {
                    &#34;FILE&#34;: &#34;${relativeFile}&#34;,
                    &#34;FILE_DIR_NAME&#34;: &#34;${fileDirname}&#34;,
                    &#34;FILE_EXT_NAME&#34;: &#34;${fileExtname}&#34;,
                    &#34;FILE_BASENAME&#34;: &#34;${fileBasename}&#34;
                }
            },
            &#34;isBackground&#34;: true,
            &#34;presentation&#34;: {
                &#34;reveal&#34;: &#34;never&#34;,
            },
            &#34;problemMatcher&#34;: []
        }
    ]
}
```

返回开发代码的目录，创建一个`Makefile`

### `Makefile`

```makefile
TARGRT_FILE=./hello.cpp		# 指定目标源文件名

cxxflags=`llvm-config --cxxflags`
ldflags=`llvm-config --ldflags`

build-pass:
	@bash -c &#34;if [[ ! -d $(FILE_DIR_NAME)/build ]];then mkdir $(FILE_DIR_NAME)/build;fi&#34; 	
	@g&#43;&#43; $(cxxflags) -ggdb -shared $(ldflags) -fPIC $(FILE) -o $(FILE_DIR_NAME)/build/$(FILE_BASENAME_NO_EXTENSION).so
	@echo &#34;\033[32mBuild $(FILE) Successfully.\033[0m&#34;

build-target:
	@bash -c &#34;if [[ ! -d $(FILE_DIR_NAME)/build ]];then mkdir $(FILE_DIR_NAME)/build;fi&#34;
	@clang -emit-llvm -c $(TARGRT_FILE) -o $(FILE_DIR_NAME)/build/1-target_before.bc
	@llvm-dis $(FILE_DIR_NAME)/build/1-target_before.bc
	@echo &#34;\033[32mBuild target.cpp successfully.\033[0m&#34;

clean:
	rm -rf ./build/*.{o,so,bc}

build: build-pass build-target

run: build
	@clear
	@opt -load $(FILE_DIR_NAME)/build/$(FILE_BASENAME_NO_EXTENSION).so --$(FILE_BASENAME_NO_EXTENSION) -enable-new-pm=0 $(FILE_DIR_NAME)/build/1-target_before.bc -o $(FILE_DIR_NAME)/build/2-target_after.bc
	@llvm-dis $(FILE_DIR_NAME)/build/2-target_after.bc
```

Q神的配置我直接搬来不能直接使用，在调试和加载时我添加了`-enable-new-pm=0`参数后续才能正常使用了

### `MyPass.cpp`

在开发目录创建

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

和`hello.cpp`

```C&#43;&#43;
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



## 测试

VScode当前窗口在`MyPass.cpp`上按`Ctrl&#43;Shift&#43;B`编译运行当前`Pass`

当前窗口开的哪个文件会影响配置文件的参数，所以要确保在`MyPass.cpp`界面编译运行

![image-20241220224949933](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220224949933.png)

编译完成，按`F5`调试一下看看

![image-20241220225214216](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241220225214216.png)

配置成功


---

> 作者:   
> URL: http://localhost:1313/posts/a5ddacb/  

