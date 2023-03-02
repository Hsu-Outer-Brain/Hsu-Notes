# Python的第三方库yaml介绍_python yaml库_我与懒惰作斗争的日子的博客-CSDN博客
[Python 的第三方库 yaml 介绍_python yaml 库_我与懒惰作斗争的日子的博客 - CSDN 博客](https://blog.csdn.net/qq_39437730/article/details/117934138?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0-117934138-blog-121892548.pc_relevant_landingrelevant&spm=1001.2101.3001.4242.1&utm_relevant_index=2) 

 yaml 是 Python 的[第三方库](https://so.csdn.net/so/search?q=%E7%AC%AC%E4%B8%89%E6%96%B9%E5%BA%93&spm=1001.2101.3001.7020)。YAML is a human friendly data serialization standard for all programming languages（YAML 是一个对所有编程语言都很友好的数据序列化标准）。  
但为了强调该语言以数据为中心，而不是以标记语言为重点，而用**返璞词**重新命名。它是一种直观的能够被电脑识别的**数据序列化格式**，是一种可读性高且容易被人类阅读、容易和脚本语言（不仅仅是 Python）交互，用于表达资料序列的编程语言。YAML 语言的**本质**是 一种通用的数据串行化格式。

-   在脚步语言中使用，实现简单，解析成本低；

-   序列化；

-   编程时写配置文件，比 xml 快，比 ini 文档功能更强。

-   YAML 是专门用于写配置文件的语言，非常简洁和强大，远比 JSON 格式方便。

-   大小写敏感；

-   使用缩进表示层级关系；

-   缩进时不允许使用 Tab 键，只允许使用空格；

-   缩进的空格数目不重要，只要相同层级的元素左侧对齐即可（一般 2 个或 4 个空格）；

-   \#表示注释当前行。

-   对象：即键值对的集合，又称为映射（mapping）/ 哈希（hashes）/ 字典（dictionary）；

-   数组：一组按次序排列的值，又称为序列（sequence）/ 列表（list）；

-   纯量：单个的、不可再分的值。

## 4.1 对象

使用冒号代表，格式为 key: value。冒号后须加一个空格。  
使用缩进表示层级关系，如下：

```yaml
key:
  child_key1: value1
  child-key2: value2


```

YAML 还支持流式（flow）语法表示对象，上例可写成：

```yaml
key: {child_key1: value1, child_key2: value2}

```

这在 Python 中是 字典嵌套字典，是这么写的：

```python
"key": {
        "child_key1":"value1",
        "child_key2":"value2"
       }

```

较为复杂的对象格式，可使用 一个问号 加一个空格代表一个复杂的 key，配合一个冒号加一个空格 代表一个 value：

```yaml
? 
  - complex_key1
  - complex_key2
: 
  - complex_value1
  - complex_value2

```

上述表示：对象的属性是一个数组\[complex_key1, complex_key2]，其对应的值也是一个数组\[complex_value1, complex_value2]。

## 4.2 数组

使用一个短横线 加一个空格代表一个数组项：

```yaml
hobby:
  - python
  - test

```

也可以这样说：

```yaml
-
  - python
  - test

```

可简单理解为：\[\[python, test]]  
再看一个相对复杂的例子：

```yaml
role:
- 
  id: 1
  name: developer
  auth: dev
- 
  id: 2
  name: tester
  auth: test 


```

可理解为：role 属性是一个数组，每个数组元素又是由 id、name、auth 3 个属性构成。  
用流式（flow）的方式表示如下：

```yaml
role: [{id: 1, name: developer, auth: dev}, {id: 2, name: tester, auth: test}]

```

### 4.2.1 对象和数组 可结合使用，形成复合结构

```yaml
languages:
 - Ruby
 - Perl
 - Python 
websites:
 YAML: yaml.org 
 Ruby: ruby-lang.org 
 Python: python.org 
 Perl: use.perl.org

```

## 4.3 纯量

纯量是最基本的、不可再分的值。YAML 提供了多种常量结构：整数、浮点数、字符串、NULL、日期、布尔值、时间。

```yaml
int: 
- 123
- 0b1010_0111_0100_1010_1110 
float:
- 3.14159
- 6.6e+5 
string:
- 'Hello world!' 
- newline
  newline2 
null: 
 nodeName: 'node'
 parent: ~ 
boolean: 
 - TRUE 
 - FALSE 
date:
- 2018-12-29 
datetime: 
- 2018-12-29T18:43:21+08:00 

```

## 4.4 还有一些特殊符号

### 4.4.1 — YAML 可在同一个文件中，使用—表示一个文档的开始

```yaml
server: 
  address: 192.168.1.100
---
spring: 
  profiles: development
  server: 
    address: 127.0.0.1
---
spring:
  profiles: production
  server: 
    address: 192.168.1.120

```

上述例子定义两个 profile，一个 development、一个 production。

也可以用 —来分割不同的内容，比如记录日志：

```yaml
---
Time: 2018-12-29T19:09:30+08:00
User: ed
Warning:
  This is an error message for the log file.
---
Time: 2018-12-29T19:11:45+08:00
User: ed
Warning:
  A slightly different error message.

```

### 4.4.2 … 和—配合使用，在一个配置文件中代表一个的结束

```yaml
---
time: 19:13:09
player: Tim
action: strike
...
---
time: 20:14:45
player: Lily
action: grand
...

```

此例相当于在一个 yaml 文件中连续写了两个 yaml 配置项。

### 4.4.3 YAML 中使用!! 做类型强行转换

```yaml
string:
  - !!str 123456
  - !!str true

```

相当于将数字和布尔类型强转为字符串（允许转换的类型还有很多）。

### 4.4.4 > 在字符串中表示折叠换行；| 保留换行。这两个符号是 YAML 中字符串经常使用的符号

```yaml
acomplistment: >
  Mark set a major league
  home run record in 1998.
status: |
  65 Home Runs
  0.278 Batting Average

```

accomplistment 的结果为：

```yaml
accomplistment=Mark set a major league home run record in 1998.

```

status 的结果为：

```yaml
status=65 Home Runs
 0.278 Batting Average

```

### 4.4.5 引用。重复的内容在 YAML 中可使用 & 来完成锚点定义，用 \* 来完成锚点引用

```yaml
hr: 
  - Mark McGwire
  - &SS Sammy Sosa
rbi: 
  - *SS
  - Ken Griffey

```

在 hr 中，使用 & SS 为 Sammy Sosa 设置了一个锚点（引用），名称为 SS；在 rbi 中，使用 \* SS 完成了锚点使用。结果是：

```yaml
{rbi=[Mark McGwire, Ken Griffey], hr=[Mark McGwire, Sammy Sosa]}

```

也可以这样定义：

```yaml
SS: &SS Sammy Sosa
hr:
 - Mark McGwire
 - *SS
rbi:
 - *SS 
 - Ken Griffey

```

还可以用锚点定义更复杂的内容：

```yaml
default: &default
    - Mark McGwire
    - Sammy Sosa
hr: *default

```

hr 相当于引用 default 数组。不过，hr: \*default 须写在同一行。

### 4.4.6 合并内容。主要是和锚点配合使用，可将一个锚点内容直接合并到一个对象中

```yaml
merge:
  - &CENTER { x: 1, y: 2 }
  - &LEFT { x: 0, y: 2 }
  - &BIG { r: 10 }
  - &SMALL { r: 1 }
  
sample1: 
    <<: *CENTER
    r: 10
    
sample2:
    << : [ *CENTER, *BIG ]
    other: haha
    
sample3:
    << : [ *CENTER, *BIG ]
    r: 100

```

在 merge 中，定义了四个锚点，分别在 sample 中使用。

sample1 中，&lt;&lt;: \*CENTER 意思是引用 {x: 1,y: 2}，并且合并到 sample1 中，那么合并的结果为：sample1={r=10, y=2, x=1}

sample2 中，&lt;&lt;: \[\*CENTER, \*BIG] 意思是联合引用 {x: 1,y: 2} 和{r: 10}，并且合并到 sample2 中，那么合并的结果为：sample2={other=haha, x=1, y=2, r=10}

sample3 中，引入了\*CENTER, \*BIG，还使用了 r: 100 覆盖了引入的 r: 10，所以 sample3 值为：sample3={r=100, y=2, x=1}

有了合并，我们就可以在配置中，把相同的基础配置抽取出来，在不同的子配置中合并引用即可。

## 5.1 安装 yaml

yaml 包名是 pyyaml，但导入是 yaml。

## 5.2 Python 使用 yaml

以 【用 Python 读取 yaml 文件（后缀可为 .yml 或 .yaml）】为例：先用 open 方法读取文件数据，再通过 load 方法转成字典（load 方法跟 json 的 load 是相似的）。

在同一个文件夹下，编写 yaml 文件，名为 cfg.yml，内容如下：

```yaml
nb:
  user: admin
  psw: 123456

```

编写读取 yaml 文件的. py 文件，名为 readyml.py，内容如下：

```yaml
import yaml
import os

curPath = os.path.dirname(os.path.realpath(__file__)) 
ymlPath = os.path.join(curPath, "cfg.yml") 


f = open(ymlPath, 'r')
cfg = f.read()
print(type(cfg)) 
print(cfg)

d = yaml.load(cfg) 
print(d)
print(type(d))

a = {'name': 'Tom',
	'race': 'cat',
	'traits': ['Two_Hand', 'Two_Eye']
}
ret = yaml.dump(a)
print(ret)
print(type(ret))

```

其中，最重要的两个方法：

-   load()，解析 yaml 文档，返回一个 Python 对象；
-   load_all()，如果是 string 或文件包含几块 yaml 文档，可用该方法来解析全部的文档，生成一个迭代器；
-   dump()，将一个 Python 对象生成为一个 yaml 文档；
-   dump_all()，将多个段输出到一个 yaml 文档中。
