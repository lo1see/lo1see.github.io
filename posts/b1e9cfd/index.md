# LLVM学习——控制流平坦化源码学习


&lt;!--more--&gt;



# OLLVM控制流平坦化学习

## 控制流平坦化结构

控制流平坦化`Control Flow Flattening`，主要结构就是如下所示

![image-20241221180841237](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221180841237.png)

通过主分发器控制程序真实代码的执行流程。删除了程序原先的跳转和执行流程，将所有真实代码作为基础块分布在控制流程图`CFG`的最底部，和名字一样`平坦化`。再由事先根据真实流程对每个块绑定的`switchVar`来判断分发流程，最终由主分发块进行分发。将原先清晰的程序执行流程转为复杂的难以直观分析的执行流程。

### 实际结构分析

分析程序示例源码

```C
#include &lt;stdio.h&gt;
#include &lt;stdlib.h&gt;

int main()
{
    int a = 2;
    int b = 4;
    int c = rand() &amp; 0xf;
    int d = 0;
    printf(&#34;%d\n&#34;, c);

    if(c - 6){
        d = a*b;
    }
    else{
        d = a;
    }

    printf(&#34;%d\n&#34;, d);

    return 0;
}
```

混淆后

![image-20241221183944055](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221183944055.png)

手动`Patch`一下这些随机生成的变量

![image-20241221184623444](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221184623444.png)

可以发现实际原理就是类似这样的代码结构：

```C
v5 = switchVar;
while(1){
    if(v5 == 1){}
    else if(v5 == 2){}
    else if(v5 == 3){}
    //......（以此类推）
    else{}
}
```

或者说是

```C
v5 = switchVar;
while(1){
    switch(v5){
        case 1:		//code......
            break;
        case 2:		//code......
            break;
        case 3:		//code......
            break;
        default:	//code......
            break;
    }
}
```

在IDA中的`控制流程图CFG`中

![image-20241221185439811](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221185439811.png)

在`控制流程图CFG`中其实感觉真实块还是很容易就能找到的，只是在没有流程图时，进行动态调试，反汇编分析，反编译分析都会十分困难。

不过根据这个特征，倒也足够反这种基础的控制流平坦化混淆了。

## 源码分析

### 核心函数分析

#### `runOnFunction(Function &amp;F)`

```c&#43;&#43;
bool Flattening::runOnFunction(Function &amp;F) {
  Function *tmp = &amp;F;
  if (toObfuscate(flag, tmp, &#34;fla&#34;)) {
    if (flatten(tmp)) {
      &#43;&#43;Flattened;
    }
  }
  return false;
}
```

`toObfuscate()`函数判断命令行中有没有`fla`，是否要执行`flatten()`平坦化函数

#### `flatten(Function *f)`

`flatten()`是最终真实对函数进行平坦化混淆的函数

```c&#43;&#43;
  char scrambling_key[16];
  llvm::cryptoutils-&gt;get_bytes(scrambling_key, 16);
```

用自定义的`cryptoutils`生成一段随机密钥，以供后续生成随机的`switchVar`值。也可以自己设定固定的密钥，若密钥固定，后续的`switchVar`则也是固定的。



```c&#43;&#43;
  // Save all original BB
  for (Function::iterator i = f-&gt;begin(); i != f-&gt;end(); &#43;&#43;i) {
    BasicBlock *tmp = &amp;*i;
    origBB.push_back(tmp);

    BasicBlock *bb = &amp;*i;
    if (isa&lt;InvokeInst&gt;(bb-&gt;getTerminator())) {
      return false;
    }
  }

  // Nothing to flatten
  if (origBB.size() &lt;= 1) {
    return false;
  }

  // Remove first BB
  origBB.erase(origBB.begin());
```

保存当前函数所有的基础块，判断如果函数太小了就不进行平坦化混淆，擦除第一个基础块，第一个基础块作为初始的序言块不能进行平坦化混淆。



```c&#43;&#43;
  // Get a pointer on the first BB
  Function::iterator tmp = f-&gt;begin(); //&#43;&#43;tmp;
  BasicBlock *insert = &amp;*tmp;

  // If main begin with an if
  BranchInst *br = NULL;
  if (isa&lt;BranchInst&gt;(insert-&gt;getTerminator())) {
    br = cast&lt;BranchInst&gt;(insert-&gt;getTerminator());
  }

  if ((br != NULL &amp;&amp; br-&gt;isConditional()) ||
      insert-&gt;getTerminator()-&gt;getNumSuccessors() &gt; 1) {
    BasicBlock::iterator i = insert-&gt;end();
    --i;

    if (insert-&gt;size() &gt; 1) {
      --i;
    }

    BasicBlock *tmpBB = insert-&gt;splitBasicBlock(i, &#34;first&#34;);
    origBB.insert(origBB.begin(), tmpBB);
  }

  // Remove jump
  insert-&gt;getTerminator()-&gt;eraseFromParent();
```

保存第一个基础块`insert`，判断第一个基础块结尾有没有分支跳转，如果有分支跳转就调用`splitBasicBlock`从第`i`个指令给分开，分成两个基础块，插在上一步保存的`origBB`全部基础块前面。



```c&#43;&#43;
  // Create switch variable and set as it
  switchVar =
      new AllocaInst(Type::getInt32Ty(f-&gt;getContext()), 0, &#34;switchVar&#34;, insert);
  new StoreInst(
      ConstantInt::get(Type::getInt32Ty(f-&gt;getContext()),
                       llvm::cryptoutils-&gt;scramble32(0, scrambling_key)),
      switchVar, insert);
```

在insert块后创建了两条指令，用于创建和初始化`switchVar`变量。调用`AllocaInst`在栈上创建变量，名称为`switchVar`；调用`StoreInst`将`ConstantInt::get`生成的随机值存入`switchVar`中，以便后续主分发器分发。



```c&#43;&#43;
  // Create main loop
  loopEntry = BasicBlock::Create(f-&gt;getContext(), &#34;loopEntry&#34;, f, insert);
  loopEnd = BasicBlock::Create(f-&gt;getContext(), &#34;loopEnd&#34;, f, insert);

  // Move first BB on top
  insert-&gt;moveBefore(loopEntry);
  BranchInst::Create(loopEntry, insert);

  // loopEnd jump to loopEntry
  BranchInst::Create(loopEntry, loopEnd);
```

先后在`insert`块前创建了两个基础块，`loopEntry`和`loopEnd`两个。此时逻辑是`loopEntry`--&gt;`loopEnd`--&gt;`insert`；再调用`insert-&gt;moveBefore`方法改变顺序为`insert`--&gt;`loopEntry`--&gt;`loopEnd`；最后调用两次`BranchInst::Create`创建了跳转指令，使得`insert`--跳转--&gt;`loopEntry`，`loopEntry`---跳转--&gt;`loopEnd`。



```c&#43;&#43;
  load = new LoadInst(switchVar-&gt;getType()-&gt;getElementType(), switchVar, &#34;switchVar&#34;, loopEntry);
```

在`loopEntry`块创建指令，读取`switchVar`中的数据



```c&#43;&#43;
  BasicBlock *swDefault =
      BasicBlock::Create(f-&gt;getContext(), &#34;switchDefault&#34;, f, loopEnd);
  BranchInst::Create(loopEnd, swDefault);

  // Create switch instruction itself and set condition
  switchI = SwitchInst::Create(&amp;*f-&gt;begin(), swDefault, 0, loopEntry);
  switchI-&gt;setCondition(load);

//  多余了
//  f-&gt;begin()-&gt;getTerminator()-&gt;eraseFromParent();
//  BranchInst::Create(loopEntry, &amp;*f-&gt;begin());
```

创建`switch`语句和`switchDefault`块。创建之后的两句语句分析后我感觉多余了，移除第一个块的跳转语句和创建第一个块`insert`和`loopEntry`的跳转已经做过了，这里又重复执行了一次，感觉没什么意义，注释掉之后也依旧可以编译运行。



```c&#43;&#43;
  // Put all BB in the switch
  for (std::vector&lt;BasicBlock *&gt;::iterator b = origBB.begin();
       b != origBB.end(); &#43;&#43;b) {
    BasicBlock *i = *b;
    ConstantInt *numCase = NULL;

    // Move the BB inside the switch (only visual, no code logic)
    i-&gt;moveBefore(loopEnd);

    // Add case to switch
    numCase = cast&lt;ConstantInt&gt;(ConstantInt::get(
        switchI-&gt;getCondition()-&gt;getType(),
        llvm::cryptoutils-&gt;scramble32(switchI-&gt;getNumCases(), scrambling_key)));
    switchI-&gt;addCase(numCase, i);
  }
```

将所有原先程序的基本块添加到`switch`语句中，移动到`loopEnd`块前，并对每个块分配不同的随机`switchVar`值。建立了各个真实基本块与`loopEntry, loopEnd, switchVar`之间的联系，还没有进行细致的处理。



```c&#43;&#43;
  for (std::vector&lt;BasicBlock *&gt;::iterator b = origBB.begin();
       b != origBB.end(); &#43;&#43;b) {
    BasicBlock *i = *b;
    ConstantInt *numCase = NULL;
  
    //code......细致处理各个分支      
  }
```

接着就是对真实基本块的遍历，细致处理各个真实基本块与`loopEntry, loopEnd, switchVar`之间的跳转关系



```c&#43;&#43;
    // Ret BB
    if (i-&gt;getTerminator()-&gt;getNumSuccessors() == 0) {
      continue;
    }
```

如果没有后继块，则就是返回块，不做处理直接`continue`



```c&#43;&#43;
    if (i-&gt;getTerminator()-&gt;getNumSuccessors() == 1) 
      // Get successor and delete terminator
      BasicBlock *succ = i-&gt;getTerminator()-&gt;getSuccessor(0);
      i-&gt;getTerminator()-&gt;eraseFromParent();

      // Get next case
      numCase = switchI-&gt;findCaseDest(succ);

      // If next case == default case (switchDefault)
      if (numCase == NULL) {
        numCase = cast&lt;ConstantInt&gt;(
            ConstantInt::get(switchI-&gt;getCondition()-&gt;getType(),
                             llvm::cryptoutils-&gt;scramble32(
                                 switchI-&gt;getNumCases() - 1, scrambling_key)));
      }

      // Update switchVar and jump to the end of loop
      new StoreInst(numCase, load-&gt;getPointerOperand(), i);
      BranchInst::Create(loopEnd, i);
      continue;
    }
```

如果有一个后继块，那么就是原先顺序执行的真实块。获取唯一的后继块，移除本块结尾的跳转指令，从`switchI`中读取后继块所属的`numCase`，创建将`numCase`存储到`switchVar`中的指令，创建本块与`loopEnd`的跳转，完成处理。



```c&#43;&#43;
    if (i-&gt;getTerminator()-&gt;getNumSuccessors() == 2) {
          // Get next cases
          ConstantInt *numCaseTrue =
                  switchI-&gt;findCaseDest(i-&gt;getTerminator()-&gt;getSuccessor(0));
          ConstantInt *numCaseFalse =
                  switchI-&gt;findCaseDest(i-&gt;getTerminator()-&gt;getSuccessor(1));

          // Check if next case == default case (switchDefault)
          if (numCaseTrue == NULL) {
              numCaseTrue = cast&lt;ConstantInt&gt;(
                      ConstantInt::get(switchI-&gt;getCondition()-&gt;getType(),
                                       llvm::cryptoutils-&gt;scramble32(
                                               switchI-&gt;getNumCases() - 1, scrambling_key)));
          }

          if (numCaseFalse == NULL) {
              numCaseFalse = cast&lt;ConstantInt&gt;(
                      ConstantInt::get(switchI-&gt;getCondition()-&gt;getType(),
                                       llvm::cryptoutils-&gt;scramble32(
                                               switchI-&gt;getNumCases() - 1, scrambling_key)));
          }

          // Create a SelectInst
          BranchInst *br = cast&lt;BranchInst&gt;(i-&gt;getTerminator());
          SelectInst *sel =
                  SelectInst::Create(br-&gt;getCondition(), numCaseTrue, numCaseFalse, &#34;&#34;,
                                     i-&gt;getTerminator());

          // Erase terminator
          i-&gt;getTerminator()-&gt;eraseFromParent();

          // Update switchVar and jump to the end of loop
          new StoreInst(sel, load-&gt;getPointerOperand(), i);
          BranchInst::Create(loopEnd, i);
          continue;
    }
```

如果有两个后继块，那么就要处理分支情况。创建两个后继块的`numCase`分别处理，将结尾的条件跳转指令改为条件赋值，如果为真则把`numCaseTrue`赋值给`switchVar`，如果为假则把`numCaseFalse`赋值给`switchVar`，最后清除本块结尾的跳转指令，创建与`loopEnd`的跳转指令。



以上就是控制流平坦化的核心代码逻辑了



## 魔改思路

根据源码中的思路

```c&#43;&#43;
i-&gt;moveBefore(loopEnd);

BranchInst::Create(loopEnd, i);
```

这样的逻辑导致了所有的真实块一定在`loopEnd`分发块之前，而在这些真实块以上的代码块几乎全为`switch`的功能而服务。

![image-20241221201634753](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221201634753.png)

在反OLLVM控制流平坦化混淆的思路中，也是先对代码划分基本块操作，找到特征明显的预分发块，对预分发块交叉引用，预分发块的前继块就是真实块了。

那么要反反OLLVM控制流平坦化，就需要失去这个特征，也就是使得`loopEnd`预分发块的前置块的前继块并非就是全部的真实块。

### 方案一

我想到的方法是实现部分真实块的后继块并非`loopEnd`预分发块，而是真实块。真实块连接真实块再连接预分发块。

实现起来很简单，比如在细致处理真实块时，随机不进行处理直接返回。

```c&#43;&#43;
    if (i-&gt;getTerminator()-&gt;getNumSuccessors() == 1) {
     //************************\\
        if(rand()&amp;0xf &gt; 0xa){
            continue;
        }
     //************************\\
        
      // Get successor and delete terminator
      BasicBlock *succ = i-&gt;getTerminator()-&gt;getSuccessor(0);
      i-&gt;getTerminator()-&gt;eraseFromParent();
      //code......
    }
```

这个随机值可以这样比较简单的设置，也可以修改下虚假控制流中的这段获取命令行参数的代码，借用`CryptoUtils`库来计算随机概率和命令行设置每次混淆的随机概率。

```c&#43;&#43;
static cl::opt&lt;int&gt;
    ObfProbRate(&#34;bcf_prob&#34;,
                cl::desc(&#34;Choose the probability [%] each basic blocks will be &#34;
                         &#34;obfuscated by the -bcf pass&#34;),
                cl::value_desc(&#34;probability rate&#34;), cl::init(defaultObfRate),
                cl::Optional);
```



和

```c&#43;&#43;
    // If it&#39;s a conditional jump
    if (i-&gt;getTerminator()-&gt;getNumSuccessors() == 2) {
      //*************\\
          continue;
      //*************\\
        
          // Get next cases
          ConstantInt *numCaseTrue =
                  switchI-&gt;findCaseDest(i-&gt;getTerminator()-&gt;getSuccessor(0));
          ConstantInt *numCaseFalse =
                  switchI-&gt;findCaseDest(i-&gt;getTerminator()-&gt;getSuccessor(1));
    }
```



测试程序源代码：

```c&#43;&#43;
#include &lt;stdio.h&gt;
#include &lt;stdlib.h&gt;
#include &lt;string.h&gt;

const char * const base64char = &#34;ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789&#43;/&#34;;

int base64_decode( const char * base64, unsigned char * bindata )
{
    int i, j;
    unsigned char k;
    unsigned char temp[4];

    for ( i = 0, j = 0; base64[i] != &#39;\0&#39; ; i &#43;= 4 )
    {
        memset( temp, 0xFF, sizeof(temp) );
        for ( k = 0 ; k &lt; 64 ; k &#43;&#43; )
        {
            if ( base64char[k] == base64[i] )
                temp[0] = k;
        }
        for ( k = 0 ; k &lt; 64 ; k &#43;&#43; )
        {
            if ( base64char[k] == base64[i &#43; 1] )
                temp[1] = k;
        }
        for ( k = 0 ; k &lt; 64 ; k &#43;&#43; )
        {
            if ( base64char[k] == base64[i &#43; 2] )
                temp[2] = k;
        }
        for ( k = 0 ; k &lt; 64 ; k &#43;&#43; )
        {
            if ( base64char[k] == base64[i &#43; 3] )
                temp[3] = k;
        }

        bindata[j&#43;&#43;] = ((unsigned char)(((unsigned char)(temp[0] &lt;&lt; 2)) &amp; 0xFC)) |
                       ((unsigned char)((unsigned char)(temp[1] &gt;&gt; 4) &amp; 0x03));
        if ( base64[i &#43; 2] == &#39;=&#39; )
            break;

        bindata[j&#43;&#43;] = ((unsigned char)(((unsigned char)(temp[1] &lt;&lt; 4)) &amp; 0xF0)) |
                       ((unsigned char)((unsigned char)(temp[2] &gt;&gt; 2) &amp; 0x0F));
        if ( base64[i &#43; 3] == &#39;=&#39; )
            break;

        bindata[j&#43;&#43;] = ((unsigned char)(((unsigned char)(temp[2] &lt;&lt; 6)) &amp; 0xF0)) |
                       ((unsigned char)(temp[3] &amp; 0x3F));
    }
    return j;
}

char * base64_encode( const unsigned char * bindata, char * base64, int binlength ) {
    int i, j;
    unsigned char current;

    for (i = 0, j = 0; i &lt; binlength; i &#43;= 3) {
        current = (bindata[i] &gt;&gt; 2);
        current &amp;= (unsigned char) 0x3F;
        base64[j&#43;&#43;] = base64char[(int) current];

        current = ((unsigned char) (bindata[i] &lt;&lt; 4)) &amp; ((unsigned char) 0x30);
        if (i &#43; 1 &gt;= binlength) {
            base64[j&#43;&#43;] = base64char[(int) current];
            base64[j&#43;&#43;] = &#39;=&#39;;
            base64[j&#43;&#43;] = &#39;=&#39;;
            break;
        }
        current |= ((unsigned char) (bindata[i &#43; 1] &gt;&gt; 4)) &amp; ((unsigned char) 0x0F);
        base64[j&#43;&#43;] = base64char[(int) current];

        current = ((unsigned char) (bindata[i &#43; 1] &lt;&lt; 2)) &amp; ((unsigned char) 0x3C);
        if (i &#43; 2 &gt;= binlength) {
            base64[j&#43;&#43;] = base64char[(int) current];
            base64[j&#43;&#43;] = &#39;=&#39;;
            break;
        }
        current |= ((unsigned char) (bindata[i &#43; 2] &gt;&gt; 6)) &amp; ((unsigned char) 0x03);
        base64[j&#43;&#43;] = base64char[(int) current];

        current = ((unsigned char) bindata[i &#43; 2]) &amp; ((unsigned char) 0x3F);
        base64[j&#43;&#43;] = base64char[(int) current];
    }
    base64[j] = &#39;\0&#39;;
    return base64;
}

int main()
{
    printf(&#34;%d\n&#34;, d);
    base64_encode((const unsigned char*)s, in, strlen(s));
    printf(&#34;%s\n%s\n&#34;, in, s);

    return 0;
}
```

`-mllvm -fla`混淆之后的`base64_decode`函数

![image-20241221204053522](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221204053522.png)

`deflat.py`反混淆工具去混淆之后，非常好看

![image-20241221204954810](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221204954810.png)

魔改混淆后`base64_decode`函数

![image-20241221205124067](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221205232215.png)

`deflat.py`反混淆工具去混淆之后

![image-20241221205232215](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221205232215.png)

函数都去没掉了，看来这个思路还是可行的



### 方案二

也是实现真实块连接真实块的思路，因为LLVM这个库的机制，在IR阶段可以进行代码优化，所以可以将一些真实块中重复执行的代码进行优化，合并为同一个基本代码块，将前面的代码块都指向这同一个块上，再由这个优化后的真实代码块跳转到`loopEnd`。

与第一个方案不同，这个方案中优化合并的代码块不仅是真实块，也可以充当作为混淆的预处理块。若工具将这个快当成预处理块无视的话是无法正确分析出代码流程的，更加提高了反混淆难度。

如我在某次比赛中遇到的的这道题，图中框住的两个代码块。当时比赛的时候做这道题我是把他当成混淆的很好的OLLVM题目来做的，无法分辨出来是真实代码块还是其他的预处理块。（不过所幸最后也是做出来了，赛后还发现其实不是OLLVM，而是VM题目，但这个控制流程图真的有点像OLLVM的吧！）

![image-20241221205545731](https://cdn.jsdelivr.net/gh/lo1see/Picturebed@main/img/image-20241221205545731.png)

貌似这样的思路类似于编译器优化.....？



---

> 作者:   
> URL: http://localhost:1313/posts/b1e9cfd/  

