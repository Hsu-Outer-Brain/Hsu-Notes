# (4条消息) 可视化神器Plotly玩转多子图绘制_尤尔小屋的猫的博客-CSDN博客_plotly 多子图
> 公众号：尤而小屋  
> 作者：Peter  
> 编辑：Peter

## [可视化](https://so.csdn.net/so/search?q=%E5%8F%AF%E8%A7%86%E5%8C%96&spm=1001.2101.3001.7020)神器 Plotly 玩转多子图绘制

大家好，我是 Peter~

很长时间没有 Plotly 绘图的文章，之前已经介绍如何绘制[柱状图](https://so.csdn.net/so/search?q=%E6%9F%B1%E7%8A%B6%E5%9B%BE&spm=1001.2101.3001.7020)、饼图、小提琴图、桑基图等，今天给大家带来的是一篇关于 Plotly 如何绘制多子图的文章，在一个画布中如何实现多种类型图形的绘制。

![](https://img-blog.csdnimg.cn/img_convert/2b5557723255db85a8ec411fe10a332f.png)

## Plotly 连载文章

![](https://img-blog.csdnimg.cn/img_convert/06d83d66b406f42369fdb469be649478.png)

## 多子图绘制

首先看看实际的效果：

Plotly 中有两种方式来绘制子图，基于 plotly_express 和 graph_objects。

但是 plotly_express 只支持 facet_plots(切面图) 和 marginal distribution subplots（边际分布子图），只有 graph_objects 是基于 make_subplots 模块才能够绘制真正意义上的多子图。下面通过实际例子来讲解。

```python
import pandas as pd
import numpy as np

import plotly_express as px
import plotly.graph_objects as go


from plotly.subplots import make_subplots

```

最重要的是要导入 make_subplots 模块。

## 基于 plotly_express

plotly_express 绘制 “子图” 是通过参数`marginal_x` 和 `marginal_y` 来实现的，表示在边际上图形的类型，可以是 "histogram", “rug”, “box”, or “violin”。

### 基于 facet_plots

切面图是由具有相同轴集的多个子图组成的图形，其中每个子图显示数据的子集，也称之为：trellis（网格） plots or small multiples。直接上官方英文，感觉更合适，暂时找不到比较恰当的中文来翻译。

#### 不同图形切面展示

先导入内置的消费数据集：

![](https://img-blog.csdnimg.cn/img_convert/697eb22764dcfcb218d55e2befa6b6c8.png)

1、基于散点图的切面图形

```python
fig = px.scatter(tips,  
                 x="total_bill", 
                 y="tip", 
                 color="smoker", 
                 facet_col="day"  
                )
fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/a5ed97b9affd80f25f195c67f982c4fd.png)

2、基于柱状图的切面展示

```python


fig = px.bar(tips, 
             x="size", 
             y="total_bill", 
             color="day", 
             facet_row="smoker"  
            )
fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/8be2708144f33158141eaf549e162641.png)

#### 控制子图个数

3、wrapping column Facets：控制显示元素个数的切面图

当我们指定的某个切面的字段的取值有很多种不同的情况，我们可以通过 facet_col_wrap 参数来控制每行最多显示的图形个数，wrap 可以理解成：被限制的意思，就是每行限制显示多少。

使用内置的 GDP 数据集：

![](https://img-blog.csdnimg.cn/img_convert/459e39567ecb1423e18bf9ea53a389dc.png)

```python


fig = px.scatter(gdp,   
                 x='gdpPercap',  
                 y='lifeExp', 
                 color='continent', 
                 size='pop',
                 facet_col='continent', 
                 facet_col_wrap=3 
                )
fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/b64e3f9e17179c9e30a8f16450bd974a.png)

上面的切面图形的 col 字段是洲，最多只有 5 个洲。下面的图形选用时间 year：

```python
fig = px.scatter(gdp,   
                 x='gdpPercap',  
                 y='lifeExp', 
                 color='continent', 
                 size='pop',
                 facet_col='year', 
                 facet_col_wrap=3  
                )
fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/2bdc2bf1aabb73218643d0081c393893.png)

每行最多显示 4 个图形：

```python
fig = px.scatter(gdp, 
                 x='gdpPercap', 
                 y='lifeExp', 
                 color='continent', 
                 size='pop',
                 facet_col='year', 
                 facet_col_wrap=4  
                )
fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/463f766082b284856f9ca998c714a47f.png)

#### 子图坐标轴设置

默认情况下子图的 y 轴是相同的：

```python


fig = px.scatter(tips, 
                 x="total_bill", 
                 y="tip", 
                 color='day', 
                 facet_row="time"
                )
fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/f7580d9f66988bdd6c1213dab1c91ea8.png)

通过参数设置不共享 y 轴：

```python
fig = px.scatter(tips, 
                 x="total_bill", 
                 y="tip", 
                 color='day', 
                 facet_row="time"  
                )


fig.update_yaxes(matches=None)

fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/e63b6b4e911bb29f19862fdf6594abdd.png)

如果是 facet_col 在列方向上的切面，则可以设置不共享 x 轴

```python
fig = px.scatter(tips, 
                 x="total_bill", 
                 y="tip", 
                 color='day', 
                 facet_col="time"  
                )


fig.update_xaxes(matches=None)

fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/45b313331cd36c189f8e4cad1ddf706c.png)

#### 子图标题设置

```python
fig = px.scatter(tips,
                 x="total_bill", 
                 y="tip", 
                 color="time",
                 facet_col="smoker"
                )
fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/4d84c45cd2dfd28aad985f05856fdb16.png)

通过设置改变子图的标题：

```python
fig = px.scatter(tips, 
                 x="total_bill", 
                 y="tip", 
                 color="time",
                 facet_col="smoker"
                )


fig.for_each_annotation(lambda a: a.update(text=a.text.split("=")[-1]))  

fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/29a6b7f044157d4baee7bdbc0fb800f6.png)

### 基于边际图 Marginal

该方法主要是通过 marginal_x 和 marginal_y 来实现的。首先导入数据集：

![](https://img-blog.csdnimg.cn/img_convert/3dd3097f88472db1f5acdf1a7c335754.png)

#### 基于不同图形边际图

1、基于散点图：

```python
fig = px.scatter(iris,  
                 x="sepal_length",  
                 y="sepal_width",
                 marginal_x="rug",  
                 marginal_y="histogram"  
                )

fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/145e74f0deec3fb1a7be070dc03fb1b8.png)

2、基于密度热图的边际图形设置：

```python
fig = px.density_heatmap(
    iris,  
    x="sepal_width",  
    y="sepal_length",  
    marginal_x="violin",  
    marginal_y="box")

fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/8e7e1013a26fe93ce8572c0af2eb0eb2.png)

3、边际图颜色设置

```python
fig = px.scatter(iris,
                 x="sepal_length", 
                 y="sepal_width", 
                 color="species",   
                 marginal_x="violin", 
                 marginal_y="box",
                 title="边际图颜色设置")
fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/856b6591a409143dc13c59c124baa3e2.png)

### 边际图和切面图连用

```python
fig = px.scatter(
    tips, 
    x="total_bill",
    y="tip", 
    color="sex", 
    facet_col="day",  
    marginal_x="violin"  
)

fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/417cf17436515467d801df816a2f0c0d.png)

## 基于 graph_objects

graph_objects 方式其实是通过 make_subplots 函数来实现的。一定要先导入：

```python


from plotly.subplots import make_subplots
import plotly.graph_objects as go

```

### 基础子图

```python

fig = make_subplots(rows=1, cols=2)  


fig.add_trace(
    go.Scatter(x=[1, 2, 3], y=[5, 10, 15]),
    row=1, col=1  
)

fig.add_trace(
    go.Scatter(x=[20, 30, 40], y=[60, 70, 80]),
    row=1, col=2  
)


fig.update_layout(height=600, 
                  width=800, 
                  title_text="子图制作")
fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/6a5a6723105c419ce7f211549fc870f3.png)

```python
fig = make_subplots(rows=3, cols=1)  


fig.add_trace(
    go.Scatter(x=[1, 2, 3], y=[5, 10, 15]),
    row=1, col=1  
)

fig.add_trace(
    go.Scatter(x=[20, 30, 40], y=[60, 70, 80]),
    row=2, col=1  
)

fig.add_trace(
    go.Scatter(x=[50, 60, 70], y=[110, 120, 130]),
    row=3, col=1  
)

fig.update_layout(height=600, 
                  width=800, 
                  title_text="子图制作")
fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/8705857060f8ce2f44ab383362e3baeb.png)

### 多行多列子图

多行多列的时候，我们可以指定第一个图形的位置，表示图形从哪里开始。默认左上角是第一个图形的位置。

```python
\fig = make_subplots(rows=2, cols=2,
                    start_cell="bottom-left"
                   )  


fig.add_trace(
    go.Bar(x=[1, 2, 3], y=[5, 10, 15]),
    row=1, col=1  
)

fig.add_trace(
    go.Scatter(x=[20, 30, 40], y=[60, 70, 80]),
    row=1, col=2  
)

fig.add_trace(
    go.Scatter(x=[50, 60, 70], y=[110, 120, 130]),
    row=2, col=1  
)
fig.add_trace(
    go.Bar(x=[50, 60, 70], y=[110, 120, 130]),
    row=2, col=2  
)

fig.update_layout(height=600, 
                  width=800, 
                  title_text="子图制作")
fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/72b6448d06c5dd390c5fd1792f4bc50d.png)

### 多子图标题设置

很多时候我们要给每个子图取名，用 subplot_titles 来实现

```python
fig = make_subplots(rows=2, cols=2,
                    start_cell="bottom-left", 
                    subplot_titles=["子图1","子图2","子图3","子图4"]  
                   )  


fig.add_trace(
    go.Bar(x=[1, 2, 3], y=[5, 10, 15]),
    row=1, col=1  
)

fig.add_trace(
    go.Scatter(x=[20, 30, 40], y=[60, 70, 80]),
    row=1, col=2  
)

fig.add_trace(
    go.Scatter(x=[50, 60, 70], y=[110, 120, 130]),
    row=2, col=1  
)
fig.add_trace(
    go.Bar(x=[50, 60, 70], y=[110, 120, 130]),
    row=2, col=2  
)

fig.update_layout(height=600, 
                  width=800, 
                  title_text="多行多列子图制作")
fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/6d1a2eb18b6a6a1b408391e40ad6d64e.png)

### 多子图标注 Annotations

在子图中我们还可以给数据添加标注：

```python
fig = make_subplots(rows=1, cols=2,
                    subplot_titles=["子图1","子图2"]  
                   )  


fig.add_trace(
    go.Bar(x=[1, 2, 3], 
           y=[5, 10, 15],
           text=["文字1", "文字2", "文字3"], 
           textposition="inside"    
          ),
    row=1, col=1  
)


fig.add_trace(
    go.Scatter(x=[1, 2, 3], 
           y=[5, 10, 15],
           mode="markers+text",  
           text=["文字4", "文字5", "文字6"],  
           textposition="bottom center"    
          ),
    row=1, col=2  
)


fig.update_layout(height=600, 
                  width=800, 
                  title_text="多子图添加标注")
fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/d4501a0c369100bf38f844cae1e8d6cd.png)

### 子图宽度设置

上面绘制的多子图都是大小相同的，我们可以通过参数来进行设置显示不同的大小：column_widths

```python
fig = make_subplots(rows=1, 
                    cols=2,
                    column_widths=[0.35,0.65],  
                    subplot_titles=["子图1","子图2"]  
                   )  

fig.add_trace(
    go.Bar(x=[1, 2, 3], 
           y=[5, 10, 15],
           text=["文字1", "文字2", "文字3"], 
           textposition="inside"    
          ),
    row=1, col=1  
)


fig.add_trace(
    go.Scatter(x=[1, 2, 3], 
           y=[5, 10, 15],
           mode="markers+text",  
           text=["文字4", "文字5", "文字6"],  
           textposition="bottom center"    
          ),
    row=1, col=2  
)

fig.update_layout(height=600, 
                  width=800, 
                  title_text="多子图添加标注")
fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/1dc02b1a44419761c309ed3fe53f15b4.png)

### 共享 x 轴

共享 x 轴指的是针对在同列行中的多个图形共享 x 轴：

```python
fig = make_subplots(rows=3, 
                    cols=1,
                    
                    shared_xaxes=True,  
                    vertical_spacing=0.03,  
                    subplot_titles=["子图1","子图2","子图3"]  
                   )  

fig.add_trace(
    go.Bar(x=[1, 2, 3], 
           y=[5, 10, 15],
           text=["文字1", "文字2", "文字3"], 
           textposition="inside"    
          ),
    row=1, col=1  
)


fig.add_trace(
    go.Scatter(x=[4, 5, 6], 
           y=[5, 10, 15],
           mode="markers+text",  
           text=["文字4", "文字5", "文字6"],  
           textposition="bottom center"    
          ),
    row=2, col=1  
)



fig.add_trace(
    go.Scatter(x=[10, 20, 30], 
           y=[25, 30, 45],
           mode="markers+text",  
           text=["文字7", "文字8", "文字9"],  
           textposition="bottom center"    
          ),
    row=3, col=1  
)


fig.update_layout(height=600, 
                  width=1000, 
                  title_text="多子图添加标注")
fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/c669ea2756e35f80d23640ac5c59409a.png)

### 共享 y 轴

共享 y 轴指的是针对在同一行中的多个图形共享 y 轴：

```python
fig = make_subplots(rows=2, cols=2,   
                    subplot_titles=["子图1","子图2","子图3","子图4"],
                    shared_yaxes=True  
                   )

fig.add_trace(go.Scatter(x=[1, 2, 3], y=[2, 3, 4]),  
              row=1, col=1)  

fig.add_trace(go.Scatter(x=[20, 30, 40], y=[5, 6, 7]),
              row=1, col=2)

fig.add_trace(go.Bar(x=[20, 30, 40], y=[20, 40, 10]),
              row=2, col=1)

fig.add_trace(go.Scatter(x=[40, 50, 60], y=[70, 80, 90]),
              row=2, col=2)

fig.update_layout(height=600, width=600,
                  title_text="多子图共享y轴")
fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/e8628c0e5343695a3082a360f101bbf5.png)

### 坐标轴自定义

```python
from plotly.subplots import make_subplots
import plotly.graph_objects as go


fig = make_subplots(
    rows=2, 
    cols=2, 
    subplot_titles=("Plot 1", "Plot 2", "Plot 3", "Plot 4")
)


fig.add_trace(go.Scatter(x=[1, 2, 3], 
                         y=[4, 5, 6]), 
              row=1, col=1)

fig.add_trace(go.Scatter(x=[20, 30, 40], 
                         y=[50, 60, 70]), 
              row=1, col=2)
fig.add_trace(go.Scatter(x=[300, 400, 500], 
                         y=[600, 700, 800]), 
              row=2, col=1)

fig.add_trace(go.Scatter(x=[4000, 5000, 6000], 
                         y=[7000, 8000, 9000]), 
              row=2, col=2)


fig.update_xaxes(title_text="xaxis—1 标题", row=1, col=1)  
fig.update_xaxes(title_text="xaxis-2 标题", range=[10, 50], row=1, col=2)  
fig.update_xaxes(title_text="xaxis-3 标题", showgrid=False, row=2, col=1)  
fig.update_xaxes(title_text="xaxis-4 标题", type="log", row=2, col=2)  


fig.update_yaxes(title_text="yaxis 1 标题", row=1, col=1)
fig.update_yaxes(title_text="yaxis 2 标题", range=[40, 80], row=1, col=2)
fig.update_yaxes(title_text="yaxis 3 标题", showgrid=False, row=2, col=1)
fig.update_yaxes(title_text="yaxis 4 标题", row=2, col=2)


fig.update_layout(title_text="自定义子图轴坐标", height=700)

fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/e6c54b97f4b9420bc36ef552577a8bb5.png)

### 共享颜色轴

使用的参数是 coloraxis

```python
fig = make_subplots(rows=1, cols=2, 
                    shared_yaxes=True)   

fig.add_trace(go.Bar(x=[1, 2, 3], y=[4, 5, 6],
                    marker=dict(color=[4, 5, 6],
                                coloraxis="coloraxis")),
              1, 1)  

fig.add_trace(go.Bar(x=[1, 2, 3], y=[2, 3, 5],
                    marker=dict(color=[2, 3, 5],
                                coloraxis="coloraxis")),
              1, 2) 

fig.update_layout(coloraxis=dict(colorscale='Bluered'),   
                  showlegend=False)  

fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/80acd6fe62272ac3ce6ad1930cf8f5c6.png)

## 子图位置自定义

子图位置的自定义是通过参数 spec 来实现的，spec 是一个 2 维的列表集合，列表中包含 rows 和 cols 两个参数。

比如我们想绘制 2\*2 的图形，但是只有 3 个图形，那么肯定有个图形会占据 2 行 1 列或者是 1 行 2 列的位置，通过例子来解释。

```python
fig = make_subplots(
    rows=2, cols=2,
    specs=[[{}, {}],  
           [{"colspan": 2}, None]],  
    subplot_titles=("子图1","子图2", "子图3"))

fig.add_trace(go.Scatter(x=[1, 2], y=[5, 6]),
                 row=1, col=1) 

fig.add_trace(go.Scatter(x=[4, 6], y=[8, 9]),
                 row=1, col=2)  

fig.add_trace(go.Scatter(x=[1, 2, 3], y=[2, 5, 2]),
                 row=2, col=1)  

fig.update_layout(showlegend=False,   
                  title_text="子图位置自定义")
fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/ae9007476b1b707b48338b4c3324c1ac.png)

```python
fig = make_subplots(
    rows=2, cols=2,
    specs=[[{"rowspan":2}, {}],  
           [None,{}]],  
    subplot_titles=("子图1","子图2", "子图3"))

fig.add_trace(go.Scatter(x=[1, 2], y=[5, 6]),
                 row=1, col=1) 

fig.add_trace(go.Scatter(x=[4, 6], y=[8, 9]),
                 row=1, col=2)  

fig.add_trace(go.Scatter(x=[1, 2, 3], y=[2, 5, 2]),
                 row=2, col=2)  

fig.update_layout(showlegend=False,   
                  title_text="子图位置自定义")
fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/b6637d9cc75e169f3c2cfa447f5d904f.png)

让我们看一个更为复杂的例子：

```python
fig = make_subplots(
    rows=5, cols=2,  
    specs=[[{}, {"rowspan": 2}], 
           [{}, None],  
           [{"rowspan": 2, "colspan": 2}, None],  
           [None, None],
           [{}, {}]]  
)

fig.add_trace(go.Scatter(x=[1, 2], y=[1, 2], name="(1,1)"), row=1, col=1)

fig.add_trace(go.Scatter(x=[1, 2], y=[1, 2], name="(1,2)"), row=1, col=2)

fig.add_trace(go.Scatter(x=[1, 2], y=[1, 2], name="(2,1)"), row=2, col=1)

fig.add_trace(go.Scatter(x=[1, 2], y=[1, 2], name="(3,1)"), row=3, col=1)

fig.add_trace(go.Scatter(x=[1, 2], y=[1, 2], name="(5,1)"), 5,1) 
fig.add_trace(go.Scatter(x=[1, 2], y=[1, 2], name="(5,2)"), 5, 2)

fig.update_layout(height=600, 
                  width=600, 
                  title_text="多子图位置自定义")

fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/d31beeba56e5195107523ad736f8a1dd.png)

## 子图类型自定义

子图可供选择的图形类型：

-   “xy”: 二维的散点 scatter、柱状图 bar 等
-   “scene”: 3 维的 scatter3d、球体 cone
-   “polar”: 极坐标图形如 scatterpolar, barpolar 等
-   “ternary”: 三元图如 scatterternary
-   “mapbox”: 地图如 scattermapbox
-   “domain”: . 针对有一定域的图形，如饼图 pie, parcoords, parcats,

```python
from plotly.subplots import make_subplots
import plotly.graph_objects as go

fig = make_subplots(
    rows=2, cols=2,
    specs=[[{"type": "xy"}, {"type": "polar"}],
           [{"type": "domain"}, {"type": "scene"}]],  
)

fig.add_trace(go.Bar(y=[2, 3, 1]),
              1, 1)

fig.add_trace(go.Barpolar(theta=[0, 45, 90], r=[2, 3, 1]),
              1, 2)

fig.add_trace(go.Pie(values=[2, 3, 1]),
              2, 1)

fig.add_trace(go.Scatter3d(x=[2, 3, 1], y=[0, 0, 0],
                           z=[0.5, 1, 2], mode="lines"),
              2, 2)

fig.update_layout(height=700, showlegend=False)

fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/724c2e7a72bf79da531cc3fc3148b47f.png)

多子图类型和位置的连用：

```python
from plotly.subplots import make_subplots
import plotly.graph_objects as go

fig = make_subplots(
    rows=2, cols=2,
    specs=[[{"type": "xy"}, {"type": "polar"}],  
           [{"colspan":2,"type": "domain"},None]], 
)

fig.add_trace(go.Bar(y=[2, 3, 1]),
              1, 1)

fig.add_trace(go.Barpolar(theta=[0, 45, 90], r=[2, 3, 1]),
              1, 2)

fig.add_trace(go.Pie(values=[2, 3, 1]),
              2, 1)  

fig.update_layout(height=700, showlegend=False)

fig.show()

```

![](https://img-blog.csdnimg.cn/img_convert/c5c420542d409057069d0a25353e5f3b.png)

## 参数解释

最后附上官网地址，多多学习：[https://plotly.com/python/subplots/](https://plotly.com/python/subplots/)

```python
plotly.subplots.make_subplots(rows=1,   
                              cols=1, 
                              shared_xaxes=False,  
                              shared_yaxes=False, 
                              start_cell='top-left',   
                              print_grid=False,  
                              horizontal_spacing=None,   
                              vertical_spacing=None, 
                              subplot_titles=None,   
                              column_widths=None,  
                              row_heights=None, 
                              specs=None,  
                              insets=None, 
                              column_titles=None, 
                              row_titles=None, 
                              x_title=None,  
                              y_title=None, 
                              figure=None, 
                              **kwargs)

```

![](https://img-blog.csdnimg.cn/img_convert/4929e5b19faaef5a88466642764de44c.png)

通过下面的方式查看帮助文档：

```python
from plotly.subplots import make_subplots
help(make_subplots)

```

 [https://blog.csdn.net/qq_25443541/article/details/118884275?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-118884275-blog-120353275.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-118884275-blog-120353275.pc_relevant_default&utm_relevant_index=2](https://blog.csdn.net/qq_25443541/article/details/118884275?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-118884275-blog-120353275.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-118884275-blog-120353275.pc_relevant_default&utm_relevant_index=2)
