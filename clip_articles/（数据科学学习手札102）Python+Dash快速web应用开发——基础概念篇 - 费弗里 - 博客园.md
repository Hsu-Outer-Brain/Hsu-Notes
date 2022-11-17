# （数据科学学习手札102）Python+Dash快速web应用开发——基础概念篇 - 费弗里 - 博客园
[（数据科学学习手札 102）Python+Dash 快速 web 应用开发——基础概念篇 - 费弗里 - 博客园](https://www.cnblogs.com/feffery/p/14258438.html) 

> 本文示例代码与数据已上传至我的`Github`仓库[https://github.com/CNFeffery/DataScienceStudyNotes](https://github.com/CNFeffery/DataScienceStudyNotes)

-   😋由我开源的先进`Dash`组件库`feffery-antd-components`正处于早期测试版本阶段，欢迎前往官网`http://fac.feffery.tech/`了解更多

　　 这是我的新系列教程**Python+Dash 快速 web 应用开发**的第一期，我们都清楚学习一个新工具需要一定的动力，那么为什么我要专门为`Dash`制作一个系列教程呢？

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-17%2019-41-33/a3fcb6d3-9ab8-412e-83e1-e8fb6a19cf60.png?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202101/1344061-20210110191333418-2111010974.png)

图 1

　　`Dash`是一个高效简洁的`Python`框架，建立在`Flask`、`Poltly.js`以及`React.js`的基础上，设计之初是为了帮助**前端知识匮乏**的数据分析人员，以纯`Python`编程的方式快速开发出交互式的数据可视化 web 应用。

　　`Dash`已经过数年的迭代发展，早期的`Dash`我也体验过，但当时还比较简陋，很多问题亟待解决，因此并没有引起我的多大注意。

　　但随着近一两年的高速发展和积极更新迭代，现阶段的`Dash`已经是一个相当成熟的框架，且其功能已经丰富到不仅仅可以用来开发在线数据可视化作品，即使是轻量级的_数据仪表盘_、_BI 应用_，甚至是搭建_文档说明_、_博客_或常规的_网站_，都驾驭得住，配合丰富的第三方拓展，只会`Python`的你可以开发出相当精美正式的 web 应用。

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-17%2019-41-33/180ca7eb-6f91-4c25-8160-9b744c9cf9e9.png?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202101/1344061-20210110191336113-836462185.png)

图 2

　　而关于`Dash`的像样的中文教程几乎没有（其实英文教程也没多少😅），有的也大多只是在照搬官方文档，因此类似之前写作完成反响不错的_geopandas_教程那样，我来写一个看得过去的系列教程吧~ 下面开始我们的`Dash`之旅吧！

　　在学习`Dash`的一开始，我们需要对`Dash`的若干基础概念进行了解，首先我们来从头开始搭建`Dash`环境，因为主要是面向数据分析处理人员，所以我推荐使用`conda`进行环境管理，参考下列命令即可完成环境的初始化：

```null
conda create -n dash-dev python=3.7 -y
conda activate dash-dev
pip install dash -U

```

　　上述代码执行完成后，你就已经创建好最基本的`Dash`运行所需环境了，你可以创建代码如下的`py`脚本并执行（推荐使用`pycharm`、`vscode`等工具进行`Dash`开发）：

> app1.py

```null
import dash
import dash_html_components as html

app = dash.Dash(__name__)

app.layout = html.H1('第一个Dash应用！')

if __name__ == '__main__':
    app.run_server()

```

　　运行上述脚本之后，一切正常的话，按照提示点击进对应网址会看到如下内容：

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-17%2019-41-33/6d97f4e9-fd47-4caf-99cf-47158bae9bbf.png?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202101/1344061-20210110191338298-1298648402.png)

图 3

　　至此我们就完成了`Dash`环境的搭建，下面我们来了解`Dash`应用中的一些基础概念：

## 2.1 用 layout 设计页面内容[#](#21-用layout设计页面内容)

　　一个 web 应用的关键之一在于其前端所呈现的页面内容，在`Dash`中我们通过对其`layout`属性进行定义，从而自由设计页面内容。

　　在前面的`app1.py`中，我们设置了`app.layout = html.H1('第一个 Dash 应用！')`，这里的`html`即开头导入的`dash_html_components`，它是`dash`的自带依赖库，用于在`Dash`应用中定义常见的`html`元素，就像前面用到的`H1`对应一级标题，即`<h1></h1>`标签。

　　而每个`html.XX`对象，其接收的第一个位置上的参数都是`children`，它用于表示对应`html`标签所包裹的内容，譬如上文的`'第一个 Dash 应用！'`，也可以通过传入多元素列表或进行多层嵌套，从而构建结构更复杂的页面内容，就像下面的例子一样：

> app2.py

```null
import dash
import dash_html_components as html

app = dash.Dash(__name__)

app.layout = html.Div(
    [
        html.H1('标题1'),
        html.H1('标题2'),
        html.P(['测试', html.Br(), '测试']),
        html.Table(
            html.Tr(
                [
                    html.Td('第一列'),
                    html.Td('第二列')
                ]
            )
        )
    ]
)

if __name__ == '__main__':
    app.run_server()

```

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-17%2019-41-33/82944393-9aa3-4e4d-9eaa-f760687cbff9.png?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202101/1344061-20210110191340097-410810973.png)

图 4

　　而除了常见的`html`元素之外，`Dash`还在其官方依赖库`dash_core_components`中内置了众多常见网页小部件，是我们实现交互式所依托的重要元素，就像下面的例子一样我们利用其`Dropdown`部件创建出一个下拉选择部件：

> app3.py

```null
import dash
import dash_html_components as html
import dash_core_components as dcc

app = dash.Dash(__name__)

app.layout = html.Div(
    [
        html.H1('下拉选择'),
        html.Br(),
        dcc.Dropdown(
            options=[
                {'label': '选项一', 'value': 1},
                {'label': '选项二', 'value': 2},
                {'label': '选项三', 'value': 3}
            ]
        )
    ]
)

if __name__ == '__main__':
    app.run_server()

```

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-17%2019-41-33/b57093d2-b994-4e42-a9af-35f52a8c3ea5.gif?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202101/1344061-20210110191342406-2129203987.gif)

图 5

　　`Dash`与`plotly`既然 “师出同门”，自然已经相互打通，我们同样可以非常轻松的在网页中插入数据可视化的内容，这里我们使用到`plotly.express`，它简化了诸多`plotly`图表的创建过程，将创建好的图表对象作为`figure`参数传入`dcc.Graph()`中即可：

> app4.py

```null
import dash
import dash_html_components as html
import dash_core_components as dcc
import plotly.express as px

app = dash.Dash(__name__)

fig = px.scatter(x=range(10), y=range(10))

app.layout = html.Div(
    [
        html.H1('嵌入plotly图表'),
        dcc.Graph(figure=fig)
    ]
)

if __name__ == '__main__':
    app.run_server()

```

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-17%2019-41-33/1718414f-5995-4a20-bb48-15f410b92050.gif?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202101/1344061-20210110191344694-1620017216.gif)

图 6

　　除了上述的几个官方`Dash`依赖库以外，还有很多优秀的第三方库都可以帮助我们快速创建出效果惊人的前端内容，关于这部分的详细内容我将会在本系列之后的文章中分主题详细介绍，敬请期待。

## 2.2 用 callback 实现交互[#](#22-用callback实现交互)

　　`Dash`最大的优点之一就是其高度封装了`React.js`，使得我们无需编写`js`代码即可实现前端与后端之间的异步交互，为了实现这一步，我们需要使用到`dash.dependencies`中的`Input`与`Output`，再配合自定义回调函数来实现所需交互功能。

　　举一个非常简单的例子：我们设计一个 web 页面，其中有一个**下拉选项**部件，当我们下拉选取到某个选项值对应的省份时，其下方打印出对应的省会城市：

> app5.py

```null
import dash
import dash_html_components as html
import dash_core_components as dcc
from dash.dependencies import Input, Output

app = dash.Dash(__name__)

app.layout = html.Div(
    [
        html.H1('根据省名查询省会城市：'),
        html.Br(),
        dcc.Dropdown(
            id='province',
            options=[
                {'label': '四川省', 'value': '四川省'},
                {'label': '陕西省', 'value': '陕西省'},
                {'label': '广东省', 'value': '广东省'}
            ],
            value='四川省'
        ),
        html.P(id='city')
    ]
)

province2city_dict = {
    '四川省': '成都市',
    '陕西省': '西安市',
    '广东省': '广州市'
}

@app.callback(Output('city', 'children'),
              Input('province', 'value'))
def province2city(province):

    return province2city_dict[province]

if __name__ == '__main__':
    app.run_server()

```

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-17%2019-41-33/39beeec1-ad37-4172-8fd1-37e88b9b2356.gif?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202101/1344061-20210110191346884-2138761051.gif)

图 7

　　在交互操作的时候查看后台可以看到，每一次点选都在进行与后台的**异步通信**，我们整个应用的页面并没有刷新，如果不用`Dash`，你就得书写相应的`js`语句，较为繁琐：

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-17%2019-41-33/27c11e23-8a5e-43b5-8b0b-562905956e06.gif?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202101/1344061-20210110191348956-1386805014.gif)

图 8

　　而`Dash`目前已经支持**多输入多输出**的回调函数书写方式，以及**阻止初次回调**、**基于表单提交状态的回调**等诸多特性，理论上你可以创建出任何形式的页面交互行为，这些内容我们都会在之后的系列文章中详细教授给大家。

## 2.3 监听图表交互式选择行为[#](#23-监听图表交互式选择行为)

　　`Dash`与`plotly`的高度耦合，还体现在其可以监听针对`plotly`图表的悬浮、选择、框选等行为，广泛适用于`plotly`中的大量常规图表与地图，这一点懂的朋友应该都明白，借助这个特性，我们可以创建出交互能力强大的 web 应用，就像我下面的这个典型的例子：

> app6.py

```null
import dash
import dash_core_components as dcc
import dash_html_components as html
from dash.dependencies import Input, Output
import plotly.express as px

app = dash.Dash(__name__)

fig = px.scatter(x=range(10), y=range(10), height=400)
fig.update_layout(clickmode='event+select')  

app.layout = html.Div(
    [
        dcc.Graph(figure=fig, id='scatter'),
        html.Hr(),
        html.Div([
            '悬浮事件：',
            html.P(id='hover')
        ]),
        html.Hr(),
        html.Div([
            '点击事件：',
            html.P(id='click')
        ]),
        html.Hr(),
        html.Div([
            '选择事件：',
            html.P(id='select')
        ]),
        html.Hr(),
        html.Div([
            '框选事件：',
            html.P(id='zoom')
        ])
    ]
)



@app.callback([Output('hover', 'children'),
               Output('click', 'children'),
               Output('select', 'children'),
               Output('zoom', 'children'),],
              [Input('scatter', 'hoverData'),
               Input('scatter', 'clickData'),
               Input('scatter', 'selectedData'),
               Input('scatter', 'relayoutData')])
def listen_to_hover(hoverData, clickData, selectedData, relayoutData):
    return str(hoverData), str(clickData), str(selectedData), str(relayoutData)


if __name__ == '__main__':
    app.run_server()

```

　　可以看到，我们监听到的悬浮、点击、选择以及框选四种行为对应传回的特征数据：

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-17%2019-41-33/dbbe0a6c-3781-496c-bb78-767fb69352b2.gif?raw=true)
](https://img2020.cnblogs.com/blog/1344061/202101/1344061-20210110191351888-1714032672.gif)

图 9

　　而这方面内容，我也会在之后的系列文章中进行非常详实的介绍😇～

　　我们接下来的系列文章就会围绕上述基础概念，以及**多页面应用**、**外部 css、js 的引入**、**Dash 应用的部署发布**等还未提及的重要内容进行详细介绍，以帮助广大使用`Python`的读者朋友使用最少的前端知识，创建出优秀的 web 应用，方便日常的工作学习生产生活，敬请期待！

* * *

　　以上就是本文的全部内容，欢迎在评论区与我进行讨论~
