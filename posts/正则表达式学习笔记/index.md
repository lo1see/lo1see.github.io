# 正则表达式学习笔记




# 正则表达式

## 简介

利用表达式设定的规则，进行查找符合规则的数据。在多种语言中都可用，本笔记用于python的正则表达式

参考文献：[正则表达式 | 白月黑羽 (byhy.net)](https://www.byhy.net/tut/py/extra/regex/)

在线正则表达式验证网址：https://regex101.com/

## 常用语法

| 方法 | 描述                                                         |
| ---- | ------------------------------------------------------------ |
| `.`  | 匹配所有字符                                                 |
| `*`  | 重复匹配任意次（0 ~ n）                                      |
| `&#43;`  | 重复匹配多次（1 ~ n）                                        |
| `?`  | 匹配 0 ~ 1 次                                                |
| `{}` | 匹配指定次数                                                 |
| `\`  | 转义字符，对元字符转义                                       |
| `[]` | 匹配几个字符之一。方括号中`.`表示字符`.`而不是任意字符；方括号中`^`表示非 |
| `^`  | 单行模式，表示匹配整个文本的开始位置；多行模式，表示匹配文本每行的开始位置 |
| `$`  | 单行模式，表示匹配整个文本的结束位置；多行模式，表示匹配文本每行的结束位置 |
| `()` | 组选择符，分组，返回符合规则的组内成员                       |
| `|`  | 或                                                           |



### 匹配某种字符

| 方法 | 描述                                                         |
| ---- | ------------------------------------------------------------ |
| `\d` | 匹配0-9之间任意一个数字字符，等价于表达式 `[0-9]`            |
| `\D` | 匹配任意一个不是0-9之间的数字字符，等价于表达式 `[^0-9]`     |
| `\s` | 匹配任意一个空白字符，包括 空格、tab、换行符等，等价于表达式 `[\t\n\r\f\v]` |
| `\S` | 匹配任意一个非空白字符，等价于表达式 `\[^ \t\n\r\f\v\]`      |
| `\w` | 匹配任意一个文字字符，包括大小写字母、数字、下划线，等价于表达式 `[a-zA-Z0-9_]` |
| `\W` | 匹配任意一个非文字字符，等价于表达式 `[^a-zA-Z0-9_]`         |

`\w`和`\W`规则匹配文字字符时，缺省情况也包括 Unicode文字字符，如果指定 ASCII 码标记，则只包括ASCII字母

反斜杠也可以用在方括号里面，比如 `[\s,.]` 表示匹配 ： 任何空白字符， 或者逗号，或者点

## 贪婪模式和非贪婪模式

在正则表达式中，`* &#43; ?`都是贪婪的，使用他们时，都会尽可能的匹配更多内容

要就近匹配少的内容时，就要使用非贪婪模式，在`*`或`&#43;`后加上`?`即可

比如`abcabcabca`按`a.*a`规则匹配结果是`abcabcabca`，但`a.*?a`的匹配结果就是`abca`

## 单行模式和多行模式

| `^`  | 单行模式，表示匹配整个文本的开始位置；多行模式，表示匹配文本每行的开始位置 |
| ---- | ------------------------------------------------------------ |
| `$`  | **单行模式，表示匹配整个文本的结束位置；多行模式，表示匹配文本每行的结束位置** |

使用单行模式只要不使用参数即可

```python
content = &#34;&#34;&#34;文本内容&#34;&#34;&#34;
p = re.compile(r&#39;^\d&#43;&#39;)
for i in p.findall(content):
	print(i)
```

使用多行模式则需要使用参数`re.M`或者全称`re.MULTILINE`

```python
content = &#34;&#34;&#34;文本内容&#34;&#34;&#34;
p = re.compile(r&#39;\d&#43;$&#39;, re.M)
for i in p.findall(content):
	print(i)
```

## 分组——分组名

当有多个分组的时候，可以使用`(?P&lt;分组名&gt;...)`的格式，给每个分组命名

这样做的好处是，更方便后续的代码提取每个分组里面的内容，比如

```python
content = &#39;&#39;&#39;张三，手机号码15945678901
李四，手机号码13945677701
王二，手机号码13845666901&#39;&#39;&#39;

import re
p = re.compile(r&#39;^(?P&lt;name&gt;.&#43;)，.&#43;(?P&lt;phone&gt;\d{11})&#39;, re.MULTILINE)
for match in  p.finditer(content):
    print(match.group(&#39;name&#39;))
    print(match.group(&#39;phone&#39;))
```

## 让点匹配换行

使用`re.DOTALL`参数，让`.`可以匹配换行符`\n`来筛选特征，比如

```python
content = &#39;&#39;&#39;
&lt;div class=&#34;el&#34;&gt;
        &lt;p class=&#34;t1&#34;&gt;           
            &lt;span&gt;
                &lt;a&gt;Python开发工程师&lt;/a&gt;
            &lt;/span&gt;
        &lt;/p&gt;
&lt;/div&gt;
&#39;&#39;&#39;

import re
p = re.compile(r&#39;class=\&#34;t1\&#34;&gt;.*?&lt;a&gt;(.*?)&lt;/a&gt;&#39;, re.DOTALL)
for one in  p.findall(content):
    print(one)
```

## 作为`re库`函数参数之一

### 切割字符串——`re.split`

方法语法：`re.split(pattern, string, maxsplit=0, flags=0)`

`-pattern`：正则表达式

`-string`：要切割的字符串

```python
import re

names = &#39;关羽; 张飞, 赵云,   马超, 黄忠  李逵&#39;

namelist = re.split(r&#39;[;,\s]\s*&#39;, names)
print(namelist)
```

### 替换字符串——`re.sub`

方法语法：`re.sub(pattern, repl, string, count=0, flags=0)`

`-pattern`：正则表达式

`-repl`：表示用什么字符串来替换

`-string`：源字符串

```python
import re

names = &#39;&#39;&#39;
下面是这学期要学习的课程：
&lt;a href=&#39;https://www.bilibili.com/video/av66771949/?p=1&#39; target=&#39;_blank&#39;&gt;点击这里，边看视频讲解，边学习以下内容&lt;/a&gt;
这节讲的是牛顿第2运动定律
&#39;&#39;&#39;

newStr = re.sub(r&#39;/av\d&#43;/&#39;, &#39;/cn345677/&#39; , names)
print(newStr)
```

### 回调函数——`re.sub`

方法语法：`re.sub(pattern, repl, string, count=0, flags=0)`

`-pattern`：正则表达式

`-repl`：回调函数

`-string`：源字符串

```python
import re

names = &#39;&#39;&#39;
下面是这学期要学习的课程：
&lt;a href=&#39;https://www.bilibili.com/video/av66771949/?p=1&#39; target=&#39;_blank&#39;&gt;点击这里，边看视频讲解，边学习以下内容&lt;/a&gt;
这节讲的是牛顿第2运动定律
&lt;a href=&#39;https://www.bilibili.com/video/av46349552/?p=125&#39; target=&#39;_blank&#39;&gt;点击这里，边看视频讲解，边学习以下内容&lt;/a&gt;
这节讲的是毕达哥拉斯公式
&#39;&#39;&#39;

# 替换函数，参数是 Match对象
def subFunc(match:re.Match) -&gt; str:
    # Match对象 的 group(0) 返回的是整个匹配上的字符串， 
    src = match.group(0)
    # Match对象 的 group(1) 返回的是第一个group分组的内容
    number = int(match.group(1)) &#43; 6
    dest = f&#39;/av{number}/&#39;
    print(f&#39;{src} 替换为 {dest}&#39;)
    # 返回值就是最终替换的字符串
    return dest

newStr = re.sub(r&#39;/av(\d&#43;)/&#39;, subFunc , names)
print(newStr)
```

第二个参数可以为函数，执行`re.sub`时先执行`subFunc`，并自带参数`match`，`match`对象包含有匹配到的字符串

`subFunc`的参数`match`是`re.Match`类型对象，成员`group(0)`是匹配到的完整字符串，成员`group(1)`是匹配到的一个字符串分组，`group(2), group(3)`类推



---

> 作者: lo1see  
> URL: http://localhost:1313/posts/%E6%AD%A3%E5%88%99%E8%A1%A8%E8%BE%BE%E5%BC%8F%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/  

