# （数据科学学习手札69）详解pandas中的map、apply、applymap、groupby、agg - 费弗里 - 博客园
[（数据科学学习手札 69）详解 pandas 中的 map、apply、applymap、groupby、agg - 费弗里 - 博客园](https://www.cnblogs.com/feffery/p/11468762.html) 

 \* 从本篇开始所有文章的数据和代码都已上传至我的 github 仓库：[https://github.com/CNFeffery/DataScienceStudyNotes](https://github.com/CNFeffery/DataScienceStudyNotes)

一、简介

　　pandas 提供了很多方便简洁的方法，用于对单列、多列数据进行批量运算或分组聚合运算，熟悉这些方法后可极大地提升数据分析的效率，也会使得你的代码更加地优雅简洁，本文就将针对 pandas 中的 map()、apply()、applymap()、groupby()、agg() 等方法展开详细介绍，并结合实际例子帮助大家更好地理解它们的使用技巧（本文使用到的所有代码及数据均保存在我的 github 仓库：[https://github.com/CNFeffery/DataScienceStudyNotes](https://github.com/CNFeffery/DataScienceStudyNotes)对应本文的文件夹下）。

二、非聚合类方法

　　这里的非聚合指的是数据处理前后没有进行分组操作，数据列的长度没有发生改变，因此本章节中不涉及 groupby()，首先读入数据，这里使用到的全美婴儿姓名数据，包含了 1880-2018 年全美每年对应每个姓名的新生儿数据，在 jupyterlab 中读入数据并打印数据集的一些基本信息以了解我们的数据集：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/a2a7423a-e5dd-403d-8ed8-41fbdfe5c2a3.gif?raw=true)

import pandas as pd #读入数据
data = pd.read_csv('data.csv')
data.head()

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/fb756c2b-3b3b-428e-8e48-a5be5d1bad34.gif?raw=true)

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/315cd220-c71e-47b1-9af3-88e91f28d82a.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905173349405-1499927407.png)

\#查看各列数据类型、数据框行列数
print(data.dtypes) print() print(data.shape)

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/4f94fb2e-2ee6-4cfc-bc6a-969f3a64419d.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905173441354-1288478834.png)

2.1 map()

　　类似 Python 内建的 map() 方法，pandas 中的 map() 方法将函数、字典索引或是一些需要接受单个输入值的特别的对象与对应的单个列的每一个元素建立联系并串行得到结果，譬如这里我们想要得到 gender 列的 F、M 转换为女性、男性的新列，可以有以下几种实现方式：

● 字典映射

　　这里我们编写 F、M 与女性、男性之间一一映射的字典，再利用 map() 方法来得到映射列：

\#定义 F-> 女性，M-> 男性的映射字典
gender2xb = {'F': '女性', 'M': '男性'} #利用 map() 方法得到对应 gender 列的映射列
data.gender.map(gender2xb)

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/8d5c9f4e-4167-42e9-bc2c-bec877b5f7c8.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905174316357-66412945.png)

 ● lambda 函数  

　　这里我们向 map() 中传入 lambda 函数来实现所需功能：

\#因为已经知道数据 gender 列性别中只有 F 和 M 所以编写如下 lambda 函数
data.gender.map(lambda x:'女性' if x is 'F' else '男性')

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/ce7c69c0-6fc0-4bb6-9024-40b907d40692.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905175151377-776995792.png)

  ● 常规函数

　　也可以传入 def 定义的常规函数：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/dee895bd-0bcb-4f03-9f39-8c2f3ca9d889.gif?raw=true)

def gender_to_xb(x): return '女性' if x is 'F' else '男性' data.gender.map(gender_to_xb) 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/43c85fd0-8e9c-4d77-8e62-e6a55436db6f.gif?raw=true)

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/1e4b34fa-c75a-46eb-a9cd-00cbdbc8f219.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905180415364-1467705360.png)

 　　map() 可以传入的内容有时候可以很特殊，如下面的例子：

● 特殊对象

　　一些接收单个输入值且有输出的对象也可以用 map() 方法来处理：

data.gender.map("This kid's gender is {}".format)

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/42502fe5-b196-4210-91b2-7d94c2b39c22.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905180749394-80819952.png)

 　　map() 还有一个参数 na_action，类似 R 中的 na.action，取值为'None'或'ingore'，用于控制遇到缺失值的处理方式，设置为'ingore'时串行运算过程中将忽略 Nan 值原样返回。

2.2 apply()

　　apply() 堪称 pandas 中最好用的方法，其使用方式跟 map() 很像，主要传入的主要参数都是接受输入返回输出，但相较于 map() 针对单列 Series 进行处理，一条 apply() 语句可以对单列或多列进行运算，覆盖非常多的使用场景，下面我们来分别介绍：

● 单列数据

　　这里我们参照 2.1 向 apply() 中传入 lambda 函数：

data.gender.apply(lambda x:'女性' if x is 'F' else '男性')

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/2135e455-d41e-4940-81f8-dadd61cb767d.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905183251758-840566831.png)

 　　可以看到这里实现了跟 map() 一样的功能。

● 输入多列数据

　　apply() 最特别的地方在于其可以同时处理多列数据，我们先来了解一下如何处理多列数据输入单列数据输出的情况，譬如这里我们编写一个使用到多列数据的函数用于拼成对于每一行描述性的话，并在 apply() 用 lambda 函数传递多个值进编写好的函数中（当调用 DataFrame.apply() 时，apply() 在串行过程中实际处理的是每一行数据而不是 Series.apply() 那样每次处理单个值），注意在处理多个值时要给 apply() 添加参数 axis=1：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/b2f065b0-d418-4dca-815d-829a52f6b3dd.gif?raw=true)

def generate_descriptive_statement(year, name, gender, count):
    year, count = str(year), str(count)
    gender = '女性' if gender is 'F' else '男性'

    return '在{}年，叫做{}性别为{}的新生儿有{}个。'.format(year, name, gender, count)

data.apply(lambda row:generate_descriptive_statement(row\['year'],
                                                      row\['name'],
                                                      row\['gender'],
                                                      row\['count']),
           axis = 1)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/1f8b7113-b0fa-4cd1-9a62-c6b5e3487710.gif?raw=true)

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/d530aabe-bc7e-4561-96f9-4e108916ed57.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905191318060-1161117274.png)

 ● 输出多列数据

　　有些时候我们利用 apply() 会遇到希望同时输出多列数据的情况，在 apply() 中同时输出多列时实际上返回的是一个 Series，这个 Series 中每个元素是与 apply() 中传入函数的返回值顺序对应的元组，比如下面我们利用 apply() 来提取 name 列中的首字母和剩余部分字母：

data.apply(lambda row: (row\['name']\[0], row\['name']\[1:]), axis=1)

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/2f5047ce-79c8-4a00-8fca-be7b6ebd94a8.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201910/1344061-20191028102205089-1737731928.png)

 　　可以看到，这里返回的是单列结果，每个元素是返回值组成的元组，这时若想直接得到各列分开的结果，需要用到 zip(\*zipped) 来解开元组序列，从而得到分离的多列返回值：

a, b = zip(\*data.apply(lambda row: (row\['name']\[0], row\['name']\[1:]), axis=1)) print(a\[:10]) print(b\[:10])

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/c082035c-4eb2-47be-be10-5b175986fc4f.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201910/1344061-20191028102850616-593865569.png)

● 结合 tqdm 给 apply() 过程添加进度条

　　我们知道 apply() 在运算时实际上仍然是一行一行遍历的方式，因此在计算量很大时如果有一个进度条来监视运行进度就很舒服，在[**（数据科学学习手札 53）Python 中 tqdm 模块的用法**](https://www.cnblogs.com/feffery/p/10343544.html)中，我对基于 tqdm 为程序添加进度条做了介绍，而 tqdm 对 pandas 也是有着很好的支持，我们可以使用 progress_apply() 代替 apply()，并在运行 progress_apply() 之前添加 tqdm.tqdm.pandas(desc='') 来启动对 apply 过程的监视，其中 desc 参数传入对进度进行说明的字符串，下面我们在上一小部分示例的基础上进行改造来添加进度条功能：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/74f956dd-63a3-4407-8a11-af9ca4dcccd3.gif?raw=true)

from tqdm import tqdm def generate_descriptive_statement(year, name, gender, count):
    year, count = str(year), str(count)
    gender = '女性' if gender is 'F' else '男性'

    return '在{}年，叫做{}性别为{}的新生儿有{}个。'.format(year, name, gender, count) #启动对紧跟着的apply过程的监视

tqdm.pandas(desc='apply')
data.progress_apply(lambda row:generate_descriptive_statement(row\['year'],
                                                      row\['name'],
                                                      row\['gender'],
                                                      row\['count']),
          axis = 1)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/aeaeb464-9ce2-4108-a113-46e88f6b7abe.gif?raw=true)

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/49e1c198-9694-4039-bb45-6407a56f0820.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905192215358-1070515151.png)

 　　可以看到在 jupyter lab 中运行程序的过程中，下方出现了监视过程的进度条，这样就可以实时了解 apply 过程跑到什么地方了。

● 结合 tqdm_notebook() 给 apply() 过程添加美观进度条

　　熟悉 tqdm 的朋友都知道其针对 jupyter notebook 开发了 ui 更加美观的 tqdm_notebook()，而要想在 jupyter notebook/jupyter lab 平台上为 pandas 的 apply 过程添加美观进度条，可以参照如下示例：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/73e34717-d7e0-4fd0-8ab6-4f9f46903a63.gif?raw=true)

from tqdm.\_tqdm_notebook import tqdm_notebook

tqdm_notebook.pandas(desc='apply')
data.progress_apply(lambda row:generate_descriptive_statement(row\['year'],
                                                      row\['name'],
                                                      row\['gender'],
                                                      row\['count']),
          axis = 1)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/348bc108-ce63-45cd-8870-850597d05799.gif?raw=true)

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/6d746079-2934-405f-b5b3-cc729a0bbe21.png?raw=true)
](https://img2018.cnblogs.com/i-beta/1344061/201911/1344061-20191126185714420-2123945270.png)

 　　这时所添加的进度条就美观了不少。

2.3  applymap()

　　applymap() 是与 map() 方法相对应的专属于 DataFrame 对象的方法，类似 map() 方法传入函数、字典等，传入对应的输出结果，不同的是 applymap() 将传入的函数等作用于整个数据框中每一个位置的元素，因此其返回结果的形状与原数据框一致，譬如下面的简单示例，我们把婴儿姓名数据中所有的字符型数据消息小写化处理，对其他类型则原样返回：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/ad1cfdcd-7c49-4ec2-ada5-9bd256a73b70.gif?raw=true)

def lower_all_string(x): if isinstance(x, str): return x.lower() else: return x

data.applymap(lower_all_string)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/b83be90d-c223-4c30-bec0-8520daed976d.gif?raw=true)

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/b5e10da9-ab6f-4b33-820f-c0049d9aab60.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905193541292-1404558833.png)

 　　其形状没有变化:

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/bf69f5bf-7f21-44e4-ac47-b63f99a56c59.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905193651929-210913839.png) 

 　　配合 applymap()，可以简洁地完成很多数据处理操作。

三、聚合类方法

　　有些时候我们需要像 SQL 里的聚合操作那样将原始数据按照某个或某些离散型的列进行分组再求和、平均数等聚合之后的值，在 pandas 中分组运算是一件非常优雅的事。

3.1 利用 groupby() 进行分组

　　要进行分组运算第一步当然就是分组，在 pandas 中对数据框进行分组使用到 groupby() 方法，其主要使用到的参数为 by，这个参数用于传入分组依据的变量名称，当变量为 1 个时传入名称字符串即可，当为多个时传入这些变量名称列表，DataFrame 对象通过 groupby() 之后返回一个生成器，需要将其列表化才能得到需要的分组后的子集，如下面的示例：

\#按照年份和性别对婴儿姓名数据进行分组
groups = data.groupby(by=\['year','gender']) #查看 groups 类型
type(groups)

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/fe8682d8-0a93-4bed-ace9-bf90f39b432d.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905195153351-774675930.png)

 　　可以看到它此时是生成器，下面我们用列表解析的方式提取出所有分组后的结果：

\#利用列表解析提取分组结果
groups = \[group for group in groups]

　　查看其中的一个元素：

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/d49c6457-6379-499e-9e86-b96736376fcf.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905195406376-2143145934.png) 

 　　可以看到每一个结果都是一个二元组，元组的第一个元素是对应这个分组结果的分组组合方式，第二个元素是分组出的子集数据框，而对于 DataFrame.groupby() 得到的结果，主要可以进行以下几种操作：

● 直接调用聚合函数

　　譬如这里我们提取 count 列后直接调用 max() 方法：

\#求每个分组中最高频次
data.groupby(by=\['year','gender'])\['count'].max()

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/09ebabb5-9359-4d9e-b064-44aaffe1dcc3.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905205813363-1272771599.png) 

 　　注意这里的 year、gender 列是以索引的形式存在的，想要把它们还原回数据框，使用 reset_index(drop=False) 即可：

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/3030e583-da53-409a-ae26-f066df1a0ce6.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905210005100-612991963.png) 

 ● 结合 apply()  

　　分组后的结果也可以直接调用 apply()，这样可以编写更加自由的函数来完成需求，譬如下面我们通过自编函数来求得每年每种性别出现频次最高的名字及对应频次，要注意的是，这里的 apply 传入的对象是每个分组之后的子数据框，所以下面的自编函数中直接接收的 df 参数即为每个分组的子数据框：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/17e65c49-1d85-4915-ab64-3fa6adccf8c3.gif?raw=true)

import numpy as np def find_most_name(df): return str(np.max(df\['count']))+'-'+df\['name']\[df\['count'].idxmax()] data.groupby(\['year','gender']).apply(find_most_name).reset_index(drop=False)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/970d9dd6-814a-4256-8747-228dc61c2fe5.gif?raw=true)

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/310083b3-d3c3-42ba-972f-048db42091cf.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905210556702-678633955.png)

3.2 利用 agg() 进行更灵活的聚合

　　agg 即 aggregate，聚合，在 pandas 中可以利用 agg() 对 Series、DataFrame 以及 groupby() 后的结果进行聚合，其传入的参数为字典，键为变量名，值为对应的聚合函数字符串，譬如 {'v1':\['sum','mean'], 'v2':\['median','max','min]} 就代表对数据框中的 v1 列进行求和、均值操作，对 v2 列进行中位数、最大值、最小值操作，下面用几个简单的例子演示其具体使用方式：

 ● 聚合 Series

　　在对 Series 进行聚合时，因为只有 1 列，所以可以不使用字典的形式传递参数，直接传入函数名列表即可：

\#求 count 列的最小值、最大值以及中位数
data\['count'].agg(\['min','max','median'])

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/ce0b1211-e476-4adb-84c3-7c73c5417231.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905212159931-1598988143.png)

  ● 聚合数据框

　　对数据框进行聚合时因为有多列，所以要使用字典的方式传入聚合方案：

data.agg({'year': \['max','min'], 'count': \['mean','std']})

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/47683098-2097-4b28-8f80-b475abccd943.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905212526417-125108134.png)

 　　值得注意的是，因为上例中对于不同变量的聚合方案不统一，所以会出现 NaN 的情况。

● 聚合 groupby() 结果

data.groupby(\['year','gender']).agg({'count':\['min','max','median']}).reset_index(drop=False)

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/41defb9d-51ed-422c-98f3-b7986428f3d1.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905213345859-1829366518.png)

 　　可以注意到虽然我们使用 reset_index() 将索引列还原回变量，但聚合结果的列名变成红色框中奇怪的样子，而在 pandas 0.25.0 以及之后的版本中，可以使用 pd.NamedAgg() 来为聚合后的每一列赋予新的名字：

data.groupby(\['year','gender']).agg(min_count=pd.NamedAgg(column='count', aggfunc='min'),
    max_count=pd.NamedAgg(column='count', aggfunc='max'),
    median=pd.NamedAgg(column='count', aggfunc='median')).reset_index(drop=False)

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-46-24/402481b7-df6b-4f4f-a93d-9b34a3f62624.png?raw=true)
](https://img2018.cnblogs.com/blog/1344061/201909/1344061-20190905214424006-70825188.png)

　　以上就是本文全部内容，如有笔误望指出！ 

\* 2019.10.28 更新：添加了使用 apply 同时返回分离的多列数据的方法

\* 2019.11.26 更新：添加了 tqdm_notebook() 版 apply() 进度条的使用方法
