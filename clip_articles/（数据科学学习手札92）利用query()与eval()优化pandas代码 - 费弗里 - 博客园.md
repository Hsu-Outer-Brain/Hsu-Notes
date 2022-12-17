# （数据科学学习手札92）利用query()与eval()优化pandas代码 - 费弗里 - 博客园
[（数据科学学习手札 92）利用 query() 与 eval() 优化 pandas 代码 - 费弗里 - 博客园](https://www.cnblogs.com/feffery/p/13440148.html) 

> 本文示例代码已上传至我的`Github`仓库[https://github.com/CNFeffery/DataScienceStudyNotes](https://github.com/CNFeffery/DataScienceStudyNotes)

　　利用`pandas`进行数据分析的过程，不仅仅是计算出结果那么简单，很多初学者喜欢在计算过程中创建一堆命名**随心所欲**的中间变量，一方面使得代码读起来费劲，另一方面越多的不必要的中间变量意味着越高的内存占用，越多的计算资源消耗。

　　因此很多时候为了提升整个数据分析工作流的**执行效率**以及代码的**简洁性**，需要配合一些`pandas`中的高级特性。本文就将带大家学习如何在`pandas`中化繁为简，利用`query()`和`eval()`来实现高效简洁的数据查询与运算。

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-32-28/b5a7179b-af9a-42c4-b803-79772a3a679c.png?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202008/1344061-20200807145017141-394850960.png)

图 1

　　`query()`顾名思义，是`pandas`中专门执行数据查询的 API，其实早在 2014 年，`pandas`0.13 版本中这个特性就已经出现了，随着后续众多版本的迭代更新，目前`pandas`中的`query()`已经进化得非常好用（笔者目前使用的`pandas`版本为 1.1.0）。

　　首先从一个实际例子认识一下`query()`的用法，这里我们使用到**netflix**电影与剧集发行数据集，包含了 6234 个作品的基本属性信息，你可以在文章开头的`Github`仓库对应目录下找到它。

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-32-28/b520230e-10cc-49db-ba76-7348b4ecd7cf.jpeg?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202008/1344061-20200807145020142-420169105.jpg)

图 2

　　正常读入数据后，我们分别使用传统方法和`query()`来执行这样的组合条件查询，不同的条件之间用对应的`and or`或`& |`连接均可：

> 找出类型为**TV Show**且国家不含**美国**的**Kids' TV**

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-32-28/792321de-f80c-47de-b0ad-49175adc97c6.png?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202008/1344061-20200807145023219-1967626668.png)

图 3

　　通过比较可以发现在使用`query()`时我们在不需要重复书写`数据框名称 [字段名]`这样的内容，字段名也直接可以当作变量使用，而且不同条件之间不需要用括号隔开，在条件繁杂的时候简化代码的效果更为明显。

　　通过上面的小例子我们认识到`query()`的强大之处，下面我们就来学习`query()`的常用特性：

## 2.1 直接解析字段名[#](#21-直接解析字段名)

　　`query()`最核心的特性就是可以直接根据传入的查询表达式，将字段名解析为对应的列，其中对字段名的命名规范有一定要求：当字段名符合`Python`中对变量命名规范的要求时，即变量名完全由**字母**、**数字**、**下划线**构成且不以**数字**开头，这样的字段是可以直接写入`query()`表达式的。

　　但大家如果尝试过会发现一些不符合上述规范的变量名也不报错，譬如：

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-32-28/8077f12b-af79-4c37-b5e0-7a85e4891509.png?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202008/1344061-20200807145026264-628486180.png)

图 4

　　因此可以记住只要在`Python`里作为变量名不报错，就可以直接填入字段名，否则需要在字段名两边加上 \`，譬如下面的例子：

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-32-28/9905f4ab-b780-489d-8303-7ba408eebad0.png?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202008/1344061-20200807145029395-1653120587.png)

图 5

## 2.2 链式表达式[#](#22-链式表达式)

　　`query()`中还支持链式表达式（_chained expressions_），使得我们可以进一步简化多条件组合时的语法：

```null
demo = pd.DataFrame({
    'a': [5, 4, 3, 2, 1],
    'b': [1, 2, 3, 4, 5]
})

demo.query("a <= b != 4")

```

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-32-28/d16bd610-9d81-4925-b5cd-05528befe4ea.png?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202008/1344061-20200807145032741-99646993.png)

图 6

## 2.3 支持 in 与 not in 判断[#](#23-支持in与not-in判断)

　　`query()`支持`Python`原生的`in`判断以及`not in`判断，从而简化了多条件判断，比如我们针对**netflix**数据集想找出`release_year`等于 2018 或 2019 的作品：

```null
netflix.query("release_year in [2018, 2019]")

```

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-32-28/e348f490-2f18-4663-a218-a9fb7be994c0.png?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202008/1344061-20200807145036051-87731390.png)

图 7

## 2.4 对外部变量的支持[#](#24-对外部变量的支持)

　　`query()`表达式还支持使用外部变量，只需要在外部变量前加上`@`符号即可：

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-32-28/619733e6-cad7-482e-85ca-86c6ebd9b2ee.png?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202008/1344061-20200807145040623-1493330699.png)

图 8

## 2.5 对常规语句的支持[#](#25-对常规语句的支持)

　　`query()`我个人觉得最惊人的功能就是其可以直接解析`Python`语句，这赋予我们极大的自由度：

```null
def country_count(s):
    '''
    计算涉及国家数量
    '''
    
    return s.split(',').__len__()


netflix.query("release_year.isin([2018, 2019]) and country.apply(@country_count) > 5")

```

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-32-28/ff8bee92-0a61-4ff7-b224-1dc9dd1fa917.png?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202008/1344061-20200807145045261-1795949456.png)

图 9

## 2.6 对 Index 与 MultiIndex 的支持[#](#26-对index与multiindex的支持)

　　除了对常规字段进行条件筛选，`query()`还支持对数据框自身的`index`进行条件筛选，具体可分为三种情况：

-   **常规 index**

　　对于只具有单列`Index`的数据框，直接在表达式中使用`index`：

```null

netflix.set_index('title').query("index.str.contains('king', case=False)")

```

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-32-28/058e4fe0-6ec0-4784-8adc-be2cb6bd4dc6.png?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202008/1344061-20200807145048364-1245602925.png)

图 10

-   **names 为空的 MultiIndex**

　　对于`MultiIndex`的情况，可分为两种，首先我们来看看`MultiIndex`的`names`为空的情况，按照顺序，用`ilevel_n`表示`MultiIndex`中的第 n 列 index：

```null

temp = netflix.set_index(['title', 'type']);temp.index.names = (None, None)


temp.query("ilevel_0.str.contains('king', case=False) and ilevel_1 == 'Movie'")

```

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-32-28/803eac53-77e3-41ce-8b4c-3304c0720bf1.png?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202008/1344061-20200807145051762-1782318058.png)

图 11

-   **names 不为空的 MultiIndex**

　　而对于`MultiIndex`的`names`有内容的情况，直接用对应的名称传入表达式即可：

```null

temp = netflix.set_index(['title', 'type'])


temp.query("title.str.contains('king', case=False) and type == 'Movie'")

```

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-32-28/9ca4b4ee-077d-4234-9bb4-3e57901af564.png?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202008/1344061-20200807145054648-1255259245.png)

图 12

　　而`eval()`类似`Python`的`eval()`函数，可以将字符串形式的命令直接解析并执行。

　　而`pandas`中的`eval()`有两种，一种是`top-level`级别的`eval()`函数，而另一种是针对数据框的`DataFrame.eval()`，我们接下来要介绍的是后者，其与`query()`有很多相同之处，下面只介绍其独有特点。

　　同样从实际例子出发，同样针对**netflix**数据，我们按照一定的计算方法为其新增两列数据，对基于`assign()`的方式和基于`eval()`的方式进行比较，其中最后一列是 False 是因为日期转换使用`coerce`策略之后无法被解析的日期会填充 pd.NAT，而缺失值之间是无法进行相等比较的：

```null

result1 = netflix.assign(years_to_now=2020 - netflix['release_year'],
                         new_date_added=pd.to_datetime(netflix['date_added'].str.strip(), 
                                                       format='%B %d, %Y', errors='coerce'))


result2 = netflix.eval('''
                       years_to_now = 2020 - release_year
                       new_date_added = @pd.to_datetime(date_added.str.strip(), format='%B %d, %Y', errors='coerce')''')

(result1 == result2).all()

```

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-32-28/6bc5c36b-e4d1-4f51-bfc9-6652d665b6d5.png?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202008/1344061-20200807145058244-1242298537.png)

图 13

　　虽然`assign()`已经算是`pandas`中简化代码的很好用的 API 了，但面对`eval()`，还是逊色不少

　　`DataFrame.eval()`通过传入多行表达式，每行作为独立的赋值语句，其中对应前面数据框中数据字段可以像`query()`一样直接书写字段名，亦可像`query()`那样直接执行`Python`语句。

　　但要注意的是`eval()`中每个新字段的赋值必须写在同一行，否则会出错：

```null
netflix.eval('''
             years_to_now = 2020 - release_year
             new_date_added = @pd.to_datetime(date_added.str.strip(), 
                                              format='%B %d, %Y', 
                                              errors='coerce')''')

```

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-32-28/77e995fc-0e63-486c-8762-a3a7ada2f9a8.png?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202008/1344061-20200807145101460-1237851264.png)

图 14

　　因此如果你要使用到的函数参数很多，可以利用`functools`中的`partial`将一些参数固化并保存，从而达到简化`eval()`表达式的目的：

```null
from functools import partial


func = partial(pd.to_datetime, format='%B %d, %Y', errors='coerce')

netflix.eval('''
             years_to_now = 2020 - release_year
             new_date_added = @func(date_added.str.strip())''')

```

　　而我最喜欢`DataFrame.eval()`的地方在于配合他，我可以在很多数据分析场景中实现 0 中间变量，一直链式下去，延续上面的例子，当我们新增了这两列数据之后，接下来我们按顺序进行按月统计影片数量、字段重命名、新增当月数量在全部记录排名字段、排序，其中关键的是**新增当月数量在全部记录排名字段**，如果不用`eval()`，你是无法在**不创建中间变量**的前提下如此简洁地完成需求的：

```null
netflix.eval('''
             years_to_now = 2020 - release_year
             new_date_added = @func(date_added.str.strip())''') \
       .resample('M', on='new_date_added') \
       .agg({'new_date_added': 'count'}) \
       .rename(columns={'new_date_added': '月度发行数量'}) \
       .eval('''月度发行数量排名 = 月度发行数量.rank(ascending=False).astype('int')''') \
       .sort_values('月度发行数量排名')

```

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-17%2023-32-28/ac70dc0a-1f8d-41d5-ac70-e130cab8da00.png?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202008/1344061-20200807145104568-369471147.png)

图 15

　　使用`query()+eval()`，升华`pandas`数据分析操作。

* * *

　　以上就是本文的全部内容，欢迎在评论区与我讨论~
