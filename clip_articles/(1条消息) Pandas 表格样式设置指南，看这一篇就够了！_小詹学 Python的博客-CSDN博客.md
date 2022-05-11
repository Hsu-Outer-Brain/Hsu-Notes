# (1条消息) Pandas 表格样式设置指南，看这一篇就够了！_小詹学 Python的博客-CSDN博客
````null
最近这些年，Python在数据分析以及人工智能领域是越来越火。```

这离不开[pandas](https://so.csdn.net/so/search?q=pandas&spm=1001.2101.3001.7020)、numpy、sklearn、TensorFlow、PyTorch等数据科学包，尤其是 Pandas，几乎是每一个从事Python数据科学相关的同学都绕不过去的。

Pandas是一种高效的数据处理库，它以 `dataframe` 和 `series` 为基本数据类型，呈现出类似excel的二维数据。

在 `Jupyter` 中(jupyter notebook 或者 jupyter lab)，可以对数据表格按照条件进行个性化的设置，方便形象的查看和使用数据。

Pandas提供了 `DataFrame.style` 属性，它会返回 `Styler对象`，用于数据样式的设置。

基于 Pandas提供的方法，本文主要内容概括如下：

![](https://img-blog.csdnimg.cn/img_convert/5144542630378f6f7e6c790f40b56b93.png)

内容目录

01 环境准备
-------

### 使用环境

本次使用的环境如下：

*   MacOS系统
    
*   Python 3.8
    
*   Jupyter Notebook
    

Pandas 和 [Numpy](https://so.csdn.net/so/search?q=Numpy&spm=1001.2101.3001.7020) 的版本为：

首先导入 pandas 和 numpy 库，这次咱们本次需要用到的两个 Python 库，如下：

```null
print(f'pandas version:{pd.__version__}')print(f'numpy version:{np.__version__}')```

### 数据准备

本次咱们使用的两份数据是关于主动基金以及消费行业指数基金的数据，本次演示用的数据仅为展示Pandas图表美化功能，对投资没有参考建议哈。

**数据1**

消费行业指数基金相关的数据，导入如下：

```null
df_consume = pd.read_csv('./data/fund_consume.csv',index_col=0,parse_dates=['上任日期','规模对应日期'])df_consume = df_consume.sort_values('基金规模(亿)',ascending=False).head(10)df_consume = df_consume.reset_index(drop=True)```

**数据2**

主动基金数据，导入如下：

```null
df_fund = pd.read_csv('./data/fund-analysis.csv',index_col=0,parse_dates=['上任日期','规模对应日期'])df_fund = df_fund.sort_values('基金规模(亿)',ascending=False).head(10)df_fund = df_fund.reset_index(drop=True)```

文章中主要使用第一份数据。

02 隐藏索引
-------

用 `hide_index()` 方法可以选择隐藏索引，代码如下：

```null
df_consume.style.hide_index()
````

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/39fa3e1a9118884bc9565f9465a0292b.png)

隐藏索引

## 03 隐藏列

用 `hide_columns()` 方法可以选择隐藏一列或者多列，代码如下：

```null
df_consume.style.hide_index().hide_columns(['性别','基金经理','上任日期','2021'])
```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/1a28636234c9d8093d51747d3ddae12b.png)

隐藏列

## 04 设置数据格式

在设置数据格式之前，需要注意下，所在列的数值的数据类型应该为数字格式，如果包含字符串、时间或者其他非数字格式，则会报错。

可以用 `DataFrame.dtypes` 属性来查看数据格式。

```null
df_consume.dtypes
```

格式如下：

从上面来看，数据格式主要包括字符串、数字和时间这三种常见的类型，此外，空值（NaN，NaT 等）也是我们需要处理的数据类型之一。

-   对于字符串类型，一般不要进行格式设置；
-   对于数字类型，是格式设置用的最多的，包括设置小数的位数、千分位、百分数形式、金额类型等；
-   对于时间类型，经常会需要转换为字符串类型进行显示；
-   对于空值，可以通过 `na_rep` 参数来设置显示内容；

Pandas 中可以通过 `style.format()` 函数来对数据格式进行设置。

````null
format_dict = {'基金规模(亿)': '￥{0:.1f}', '规模对应日期':lambda x: "{}".format(x.strftime('%Y%m%d')),df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期','2021'])\```

![](https://img-blog.csdnimg.cn/img_convert/0db1fad2b15341a56d502401e794d096.png)

数据格式设置

### 空值设置

使用 `na_rep` 设置空值的显示，一般可以用 `-`、`/`、`MISSING` 等来表示:

```null
df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期','2021'])\                .format(format_dict,na_rep='-')```

![](https://img-blog.csdnimg.cn/img_convert/73bb2eae5156f87451f4cf44c6881411.png)

空值设置

05 颜色高亮设置
---------

对于最大值、最小值、NaN等各类值的颜色高亮设置，pandas 已经有专门的函数来处理，配合 `axis` 参数可以对行或者列进行应用：

*   highlight\_max()
    
*   highlight\_min()
    
*   highlight\_null()
    
*   highlight\_between()
    

### highlight\_max

通过 `highlight_max()`来高亮最大值，其中 `axis=0` 是按列进行统计：

```null
df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期',])\                .highlight_max(axis=0,subset=['2018','2019','2020'])```

![](https://img-blog.csdnimg.cn/img_convert/fd81a09e2fd8c9c678d052f9b444990b.png)

高亮最大值

### highlight\_min

通过 `highlight_min()`来高亮最小值，其中 `axis=1` 是按行进行统计：

```null
df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期',])\                .highlight_min(axis=1,subset=['2018','2019','2020'])```

![](https://img-blog.csdnimg.cn/img_convert/574abb064baedb48e7d256d87d2889b6.png)

高亮最小值

### highlight\_null

通过 `highlight_null()`来高亮空值（NaN值）

```null
df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期',])\```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/5caf1ca3df7411d21aacfdfbf59a573b.png)

高亮空值

### highlight\_between

`highlight_between()` 函数，对处于范围内的数据进行高亮显示。

`highlight_between()` 函数的使用参数如下：

`Styler.highlight_between(subset=None, color='yellow', axis=0, left=None, right=None, inclusive='both', props=None)`

`highlight_between()` 函数,对处于范围内的数据进行高亮显示，通过 `left` 和 `right` 参数来设置两边的范围。

需要注意下，`highlight_between()` 函数从 pandas 1.3.0版本开始才有，旧的版本可能不能使用哦。

下面示例中 对2018年至2020年的年度涨跌幅度 `-20%~+20%` 范围内的数据进行高亮标注.

```null
df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期',])\                .highlight_between(left=-0.2,right=0.2,subset=['2018','2019','2020'])```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/06376a004f10cfdbf41f503c2401fb21.png)

也可以分别对不同年度的不同涨跌范围进行设置，比如下面示例中:

*   2018年的年度涨跌幅度 `-15%~+0%` 范围；
    
*   2019年的年度涨跌幅度 `0%~20%%` 范围；
    
*   2020年的年度涨跌幅度 `0%~40%` 范围。
    

```null
df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期',])\                .highlight_between(left=[-0.15,0,0],right=[0,0.2,0.4],subset=['2018','2019','2020'],axis=1)```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/dd11009688f474ef715c6426f5952694.png)

### 个性化设置

`highlight_max()`、`highlight_min()`、`highlight_null()` 等函数的默认颜色设置，我们不一定满意，可以通过 `props` 参数来进行修改。

**字体颜色和背景颜色设置**

```null
df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期',])\                .highlight_min(axis=1,subset=['2018','2019','2020','2021'],props='color:black;background-color:#99ff66')\                .highlight_max(axis=1,subset=['2018','2019','2020','2021'],props='color:black;background-color:#ee7621')\                .highlight_null(props='color:white;background-color:darkblue')```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/3c5146ad52121949f9a571c4a36d4ee5.png)

**字体加粗以及字体颜色设置**

```null
df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期',])\                .highlight_between(left=-0.2,right=0.2,subset=['2018','2019','2020'],props='font-weight:bold;color:#ee7621')```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/4df22f26e0ab11e025f44a8f1cce3445.png)

类似的个性化设置，在本文后续内容中也是适用的。

06 色阶颜色设置
---------

### 背景色阶颜色设置

使用 `background_gradient()` 函数可以对背景颜色进行设置。

该函数的参数如下：

`Styler.background_gradient(cmap='PuBu', low=0, high=0, axis=0, subset=None, text_color_threshold=0.408, vmin=None, vmax=None, gmap=None)`

使用如下：

```null
df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期'])\                .background_gradient(cmap='Blues')```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/e3205b0d4860d597f5304c79d0550baf.png)

如果不对 `subset` 进行设置，`background_gradient` 函数将默认对所有数值类型的列进行背景颜色标注。

对 `subset` 进行设置后，可以选择特定的列或特定的范围进行背景颜色的设置。

```null
df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期'])\                .background_gradient(subset=['基金规模(亿)'],cmap='Blues')```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/7248659c3e052a74822518e29c01e918.png)

此外，可以通过对 low 和 high 值的设置，可以来调节背景颜色的范围，low 和 high 分别是压缩 低端和高端的颜色范围，其数值范围一般是 0~1 ，各位可以调试下。

```null
# 通过对 low 和 high 值的设置，可以来调节背景颜色的范围# low 和 high 分别是压缩 低端和高端的颜色范围，其数值范围一般是 0~1 ，各位可以调试下df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期'])\                .background_gradient(subset=['基金规模(亿)'],cmap='Blues',low=0.3,high=0.9)```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/386dcf5d678e77cfdd87a64edc732d04.png)

当数据范围比较大时，可以通过设置 vmin 和 vmax 来设置最小和最大的颜色的设置起始点。

比如下面，基金规模在20亿以下的，颜色最浅，规模70亿以上的，颜色最深，20~70亿之间的，颜色渐变。

```null
df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期'])\                .background_gradient(subset=['基金规模(亿)'],cmap='Blues',vmin=20,vmax=70)```

![](https://img-blog.csdnimg.cn/img_convert/9b79452d597cb12c415d1e4c2868f7ee.png)

通过 `gmap` 的设置，可以方便的按照某列的值，对行进行全部的背景设置

```null
df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期'])\                .background_gradient(cmap='Blues',gmap=df_consume['基金规模(亿)'])```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/4ae2639c1d06c567a020f6f053215a3b.png)

`gmap` 还可以以矩阵的形式对数据进行样式设置，如下：

```null
df_gmap = df_consume.loc[:2,['基金名称','管理费','基金规模(亿)','2020']]gmap = np.array([[1,2,3], [2,3,4], [3,4,5]])  # 3*3 矩阵，后面需要进行颜色设置的形状也需要是 3*3，需要保持一致df_gmap.style.background_gradient(axis=None, gmap=gmap,    cmap='Blues', subset=['管理费','基金规模(亿)','2020']```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/1f89ca3882bd981ecacf65ac7aa58040.png)

上面 gmap 是 `3*3 矩阵`，后面需要进行颜色设置的形状也需要是 `3*3`，需要保持一致。

需要注意的是 颜色设置是根据 `gmap`中的值来设置颜色深浅的，而不是根据 `DataFrame` 中的数值来的。

这个在某些特定的情况下可能会用到。

### 文本色阶颜色设置

类似于背景色阶颜色设置，文本也是可以进行颜色设置的。

使用 `text_gradient()` 函数可以实现这个功能，其参数如下：

`Styler.text_gradient(cmap='PuBu', low=0, high=0, axis=0, subset=None, vmin=None, vmax=None, gmap=None)`

`text_gradient()` 函数的用法跟 `background_gradient()` 函数的用法基本是一样的。

下面演示两个使用案例，其他的用法参考 `background_gradient()` 函数。

**某列的文本色阶显示**

```null
df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期'])\                .text_gradient(subset=['基金规模(亿)'],cmap='RdYlGn')```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/1f0574f04863e336d4a7f76fef605353.png)

**全部表格的文本色阶显示**

```null
# 通过 `gmap` 的设置，可以方便的按照某列的值，对行进行全部的文本颜色设置df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期'])\                .text_gradient(cmap='RdYlGn',gmap=df_consume['基金规模(亿)'])```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/3a035b731b8a02ab93f75473a6bbe48d.png)

07 数据条显示
--------

数据条的显示方式，可以同时在数据表格里对数据进行可视化显示，这个功能咱们在 Excel 里也是经常用到的。

在 pandas 中，可以使用 `DataFrame.style.bar()` 函数来实现这个功能，其参数如下：

`Styler.bar(subset=None, axis=0, color='#d65f5f', width=100, align='left', vmin=None, vmax=None)`

示例代码如下：

```null
df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期'])\                .bar(subset=['基金规模(亿)','2018','2021'],color=['#99ff66','#ee7621'])```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/2bf9a0edba253b6a0b482a580a43fca5.png)

### 设置对其方式

上面这个可视化效果，对于正负数值的区别，看起来总是有点别扭。

可以通过设置 `aligh` 参数的值来控制显示方式：

*   left: 最小值从单元格的左侧开始。
    
*   zero: 零值位于单元格的中心。
    
*   mid: 单元格的中心在（max-min）/ 2，或者如果值全为负（正），则零对齐于单元格的右（左）。
    

将显示设置为 `mid` 后，符合大部分人的视觉审美观，代码如下：

```null
df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期'])\                .bar(subset=['基金规模(亿)','2018','2021'],color=['#99ff66','#ee7621'],align='mid')```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/667f4b642d113f13175f3d00d4fa618c.png)

关于颜色设置，`color=['#99ff66','#ee7621']`， `color`可以设置为单个颜色，所有的数据只显示同一个颜色，也可以设置为包含两个元素的list或tuple形式，左边的颜色标注负数值，右边的颜色标注正数值。

08 自定义函数的使用
-----------

通过 `apply` 和 `applymap` 函数，用户可以使用自定义函数来进行样式设置。

其中：

*   `apply` 通过axis参数，每一次将一列或一行或整个表传递到DataFrame中。对于按列使用 axis=0, 按行使用 axis=1, 整个表使用 axis=None。
    
*   `applymap` 作用于范围内的每个元素。
    

### apply

先自定义了函数`max_value()`，用来找到符合条件的最大值，`apply` 使用的示例代码如下：

**按列设置样式**

```null
def max_value(x, color='red'):return np.where(x == np.nanmax(x.to_numpy()), f"color: {color};background-color:yellow", None)df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期',])\                .apply(max_value,axis=0,subset=['2018','2019','2020','2021'])```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/0419e58d0a8ba683bc91ec328c29b355.png)

**按行设置样式**

```null
df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期',])\                .apply(max_value,axis=1,subset=['2018','2019','2020','2021'])```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/e3378b69531297c656745664eb97c1ff.png)

**按整个表格设置样式**

按整个表格设置样式时，需要注意的是，整个表格的数据类型需要是一样的，不然会报错。

示例代码如下：

```null
# 注意，整个表格的数据类型需要是一样的，不然会报错df_consume_1 = df_consume[['2018','2019','2020','2021']]df_consume_1.style.hide_index().apply(max_value,axis=None)```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/4ee2de793434397cb1f2109d3400f1e5.png)

### applymap

继续上面的数据表格，我们来自定义一个函数，对于基金的年度涨跌幅情况，年度上涨以橙色背景标注，下跌以绿色背景标注，NaN值以灰色背景标注。

由于 `applymap` 是作用于每个元素的，因此该函数不需要 `axis` 这个参数来进行设置，示例代码如下：

```null
        color = '#EE7621'  # light red        color =  '#99ff66' # light green  '#99ff66'        color = '#FFFAFA'  # ligth grayreturn f'background-color: {color}'format_dict = {'基金规模(亿)': '￥{0:.1f}', '规模对应日期':lambda x: "{}".format(x.strftime('%Y%m%d')),df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期',])\                .applymap(color_returns,subset=['2018','2019','2020','2021'])```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/78e4257c12f1b98cd56899995017005d.png)

09 颜色设置范围选择
-----------

在使用 `Style` 中的函数对表格数据进行样式设置时，对于有 `subset` 参数的函数，可以通过设置 行和列的范围来控制需要进行样式设置的区域。

### 对行(row)进行范围设置

```null
df_consume_1.style.applymap(color_returns,subset=pd.IndexSlice[1:5,])
````

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/b42fe46f26555773d056ff35efc624af.png)

### 对列 (column) 进行范围设置

```null
df_consume_1.style.applymap(color_returns,subset=['2019','2020'])
```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/99f8e001a8a07c37cc3458041d4ced04.png)

### 对行和列同时进行范围设置

````null
df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期',])\                .applymap(color_returns,subset=pd.IndexSlice[1:5,['2018','2019','2020']])```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/39d4a3ace5db1319e008639987b0bad6.png)

10 共享样式
-------

对于pandas 中样式设置后的共享复用，目前支持通过 `Styler.export()` 导出样式，然后通过 `Styler.use()` 来使用导出的样式。

不过经过阳哥的测试，简单的样式导出与使用是可以的。但稍微复杂一些的情况，目前的pandas版本是不太好用的。

### 简单样式

示例如下，先保存当前样式：

```null
df_consume_1 = df_consume[['2018','2019','2020','2021']]style1 = df_consume_1.style.hide_index()\                .highlight_min(axis=1,subset=['2018','2019','2020','2021'],props='color:black;background-color:#99ff66')\                .highlight_max(axis=1,subset=['2018','2019','2020','2021'],props='color:black;background-color:#ee7621')\                .highlight_null(props='color:white;background-color:darkblue')```

保存的样式的效果如下：

![](https://img-blog.csdnimg.cn/img_convert/2bf871ea99e6432aab1a9966848d9795.png)

使用保存的样式：

```null
df_fund_1 = df_fund[['2018','2019','2020','2021']]df_fund_1.style.use(style1.export())```

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/48dc4bf41c55d10584b276cb3ac33a04.png)

由于后面的数据表格是没有空值的，所以两者的样式实际是一样的。

### 复杂样式

当样式设置较多时，比如同时隐藏索引、隐藏列、设置数据格式、高亮特定值等，这个时候有些操作在导出后使用时并没有效果。

测试如下，先保存样式：

```null
style3 = df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期',])\                .highlight_min(axis=1,subset=['2018','2019','2020','2021'],props='color:black;background-color:#99ff66')\                .highlight_max(axis=1,subset=['2018','2019','2020','2021'],props='color:black;background-color:#ee7621')\                .highlight_null(props='color:white;background-color:darkblue')```

保存样式的效果如下：

![](https://img-blog.csdnimg.cn/img_convert/ef2d37cfe5c73d6fb74ec11e4aa6293a.png)

使用保存的样式：

```null
df_fund.style.use(style3.export())
````

效果如下：

![](https://img-blog.csdnimg.cn/img_convert/ee9512485a9c5ccb534d5041496d9ca7.png)

从上面来看，我们希望的样式效果，并没有很好的实现。

所以，针对较为复杂的样式，还是乖乖的复制代码使用吧。

## 11 导出样式到 Excel

导出样式到 Excel 中，这个功能还是比较实用的。

DataFrames 使用 `OpenPyXL` 或`XlsxWriter` 引擎可以将样式导出到 Excel 工作表。

不过，这个功能目前也还是处于不断完善过程中，估计有时候有些内容会没有效果。

大家可以在使用过程中来发现其中的一些问题。

来看一个案例：

````null
df_consume.style.hide_index()\                .hide_columns(['性别','基金经理','上任日期',])\                .highlight_min(axis=1,subset=['2018','2019','2020','2021'],props='color:black;background-color:#99ff66')\                .highlight_max(axis=1,subset=['2018','2019','2020','2021'],props='color:black;background-color:#ee7621')\                .highlight_null(props='color:white;background-color:darkblue')\                .to_excel('style_export.xlsx',engine = 'openpyxl')```

上面的案例内容导出到 excel 后，我从 excel 中打开查看了下效果如下：

![](https://img-blog.csdnimg.cn/img_convert/8ff9b760cb0b6aeeb2048c56c2f97959.png)

可以看出，跟共享样式里有些相同的问题，比如隐藏索引、隐藏列、设置数据格式等效果并没有实现。

**推荐阅读**

[Pandas处理数据太慢，来试试Polars吧！](http://mp.weixin.qq.com/s?__biz=MzU0OTU5OTI4MA%3D%3D&chksm=fbafdcf3ccd855e54c5d600e58cbdabb5cdafc52a341ab8c90a6761ab9c4a37803f28eac0e79&idx=2&mid=2247499180&scene=21&sn=2c27b1ed4a61a73aed1b43325d5ed7ce#wechat_redirect)  

[懒人必备！只需一行代码，就能导入所有的Python库](http://mp.weixin.qq.com/s?__biz=MzU0OTU5OTI4MA%3D%3D&chksm=fbafdceaccd855fc14b63812771c760fbe5b37e959e78d615b8661adf1b1f137bbac0fbb0da6&idx=1&mid=2247499189&scene=21&sn=95bd7a4a63939ff9b7d3e8976d99595e#wechat_redirect)  

[绝！关于pip的15个使用小技巧](http://mp.weixin.qq.com/s?__biz=MzU0OTU5OTI4MA%3D%3D&chksm=fbafdcfdccd855eb06aef0e37afb74c4df271db8e4ead9e7f540edf5aa7ed4e5170084ccaa22&idx=1&mid=2247499170&scene=21&sn=d4108c9371035653011fa1542252b1ca#wechat_redirect)  

[介绍10个常用的Python内置函数，99.99%的人都在用！](http://mp.weixin.qq.com/s?__biz=MzU0OTU5OTI4MA%3D%3D&chksm=fbafdcc1ccd855d71e0716812127b6ca54d97f191a98f1e203bc3d744813c55cbcd942ffac62&idx=2&mid=2247499166&scene=21&sn=8bde34934a74d5649152250113b168fa#wechat_redirect)  

[可能是全网最完整的 Python 操作 Excel库总结！](http://mp.weixin.qq.com/s?__biz=MzU0OTU5OTI4MA%3D%3D&chksm=fbafdcceccd855d8bd78759043b09edc497a9423c526a8761b0d0794666e650e182a3d2d66cf&idx=1&mid=2247499153&scene=21&sn=83d0c4aa533cabbf78a7880308a448a5#wechat_redirect) 
 [https://blog.csdn.net/weixin_40787712/article/details/118917799](https://blog.csdn.net/weixin_40787712/article/details/118917799)
````
