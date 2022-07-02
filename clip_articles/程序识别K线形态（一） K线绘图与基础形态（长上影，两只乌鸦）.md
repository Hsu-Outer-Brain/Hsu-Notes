# 程序识别K线形态（一） K线绘图与基础形态（长上影，两只乌鸦）
**python 识别股票 K 线形态，准确率回测（一）**

对于一些做股票技术分析的投资者来说，对常见的 k 线形态应该都不陌生，比如十字星，红三兵，头肩顶 (底)，岛型反转，吊颈线，两只乌鸦，三只乌鸦，四只乌鸦（哈）

![](https://article-images.zsxq.com/FuRP7yLY1kfbjUojJz9zbw-NjL32)

做价投的投资者可能会对这些划线的嗤之以鼻，而短线操盘者却有可能把它奉为圭臬，以之作为买卖标准。

海乃百川，兼听则明，笔者认为多吸收不同的观点与技术，可以更加全面的加深投资认知。你画的圆圈越大，圆圈外的未知空间也越大。

本文介绍使用 python 对 A 股市场的股票 (转债也可以) 的 K 线进行识别，然后回测某个形态出现后接下来的涨跌幅，还可以通过设定的形态进行选股。

举个例子，我们可以统计所有个股出现了 “早晨之星 “这一形态后，一周后甚至一个月后，个股是涨了还是跌了，从大量结果中统计出一些有意义的结果。

为了便于验证查看结果图形，我们先介绍如何使用 python 来画股票 K 线图。

有数据之后，剩下画图是很简单的，很多第三方的库已经封装好了各种蜡烛图绘制函数。我们要做只是传入每天开，收盘价，以及最高最低价。

**获取股票的数据**

市面有不少第三方的库可以获取股票和可转债的日线数据，比如 tushare 和 akshare。本文以股票数据为例。

以 akshare 为例，代码如下所示：

\# pip install akshare

import akshare as ak

def get_k_line(code\\="sz002241"): #

  df = ak.stock_zh_a_daily(symbol\\=code, start_date\\="20220101", end_date\\="20220515",

                             adjust\\="qfq")

  # 这个函数只适合取股票数据，转债会报错，转债的日线数据可以使用其他函数，这里选择股价前复权

  df.to_excel("同和药业 k.xlsx")

如上面代码所示，我们使用 stock_zh_a_daily 方法获取一段时间的数据，这里的日期可以根据你的需求进行调整，比如设置 start_date=“20200101”，end_date=“20220515”，就可以获取一年时间段的股票数据。

而股票的代码有 sh 与 sz 之分，sz 代表深市的股票，所有深市的股票前面都必须带上 sz，比如这里的同和药业，就是 sz002241。

同样的，sh 是上证交易所的股票，所有上证的股票前面必须带上 sh。

这里，我们来看看获取的同和药业股票数据格式，如下图所示：

![](https://article-images.zsxq.com/Fta4Pw4bjo3gccZ4-PcS8pf3IfOy)

date：交易日期

open：代表开盘价

high：当天最高价

low：当天最低价

close：当天收盘价

volume：当天成交量（元）

outstanding_share：流动股本 (股)

turnover：换手率

既然已经拿到了数据，下面我们来绘制 K 线图。

**绘制 K 线图**

在 python 中，绘制 K 线图需要用到 mpl_finance 库，而这个库提供了 4 种方式绘制 K 线图，这里我们介绍其中的一种，代码如下所示：

import mpl_finance as mpf

import matplotlib.pyplot as plt

import pandas as pd

​

\#创建绘图的基本参数

fig\\=plt.figure(figsize\\=(12, 8))

ax\\=fig.add_subplot(111)

​

\#获取刚才的股票数据

df = pd.read_excel("同和药业 k.xlsx")

mpf.candlestick2_ochl(ax, df\["open"], df\["close"], df\["high"], df\["low"], width\\=0.6, colorup\\='r',colordown\\='green',alpha\\=1.0)

\#显示出来

plt.show()

运行此段代码后，会显示如下效果图：

![](https://article-images.zsxq.com/Ft6qJi27u0B56vr3tk1VvqRIFj3_)

和东财上的同和药业的 k 线图基本一致的。

不过，这个 K 线图有一个问题，就是 X 坐标轴并不是显示的时间，而是数字，这是因为我们绘制 K 线图的方法，并没有提供 X 轴的参数，那怎么让下面的数字替换为时间呢？我们先来看一段代码：

import matplotlib.ticker as ticker

\#将股票时间转换为标准时间，不带时分秒的数据

def date_to_num(dates):

  num_time = \[]

  for date in dates:

    date_time = datetime.strptime(date, '%Y-%m-%d')

    num_date = date2num(date_time)

    num_time.append(num_date)

  return num_time

​

\#创建绘图的基本参数

fig\\=plt.figure(figsize\\=(12, 8))

ax\\=fig.add_subplot(111)

​

\#获取刚才的股票数据

df = pd.read_excel("同和药业 k.xlsx")

mpf.candlestick2_ochl(ax, df\["open"], df\["close"], df\["high"], df\["low"], width\\=0.6, colorup\\='r',colordown\\='green',alpha\\=1.0)

df\['date'] = pd.to_datetime(df\['date'])

df\['date'] = df\['date'].apply(lambda x: x.strftime('%Y-%m-%d'))

def format_date(x, pos\\=None):

  if x &lt; 0 or x > len(df\['date']) - 1:

    return ''

  return df\['date']\[int(x)]

ax.xaxis.set_major_formatter(ticker.FuncFormatter(format_date))

plt.setp(plt.gca().get_xticklabels(), rotation\\=45, horizontalalignment\\='right')

\#显示出来

plt.show()

这里，我们定义了 2 个方法：第 1 个方法 date_to_num 主要的作用就是将获取的时间数据转换为标准的日期数据；第 2 个方法，就是根据 X 的数值替换时间值。

其中，set_major_formatter 方法是将数值替换为时间值的操作库，而 plt.setup 的功能就是设置 X 轴的标签倾斜 45 度以及右对齐。运行之后，我们的 K 线图就显示的非常完美了，如下图所示：

![](https://article-images.zsxq.com/FpgOVlpzKwAm1AfarRPh9A8Gyv0_)

**均线图**

虽然我们实现了 K 线图，但是有没有发现，其实大多数的股票交易软件 K 线图上面其实还标记有均线，比如常用的有 5 日均线，10 日均线，30 均线。所以，我们需要将均线一起添加到我们的 K 线图之上。计算均线的方式如下：

df\["SMA5"] = df\["close"].rolling(5).mean()

df\["SMA10"] = df\["close"].rolling(10).mean()

df\["SMA30"] = df\["close"].rolling(30).mean()

3 行代码就可以获取到股票的均线值。接着，我们可以使用上面的 ax 进行绘制均线了，添加的代码如下所示：

ax.plot(np.arange(0, len(df)), df\['SMA5']) # 绘制 5 日均线

ax.plot(np.arange(0, len(df)), df\['SMA10']) # 绘制 10 日均线

ax.plot(np.arange(0, len(df)), df\['SMA30']) # 绘制 30 日均线

这里，同样将 X 轴设置为数字，将这两段代码添加到上面的 K 线图代码中，自然也会将 X 轴替换为时间。运行之后，显示效果如下图所示：

![](https://article-images.zsxq.com/Fjb4lFMKBLnk1gvft6sZzwsRihNm)

这里 5 日均线为蓝色，10 日均线为橙色，30 日均线为绿色，如果需要自己设置颜色，可以在每条均线的的绘制方法中加入 color 参数。

细心的读者肯定发现 30 日均线只有最后一小段，这是因为前 29 日不够 30 日是算不出均线的，同样的 5，10 日均线也是如此。

**成交量**

最后，我们还需要绘制成交量。在多数的股票交易软件中，上面是 K 线图，一般下面对应的就是成交量，这样对比起来看，往往能直观的看到数据的变化。

但是，因为上面有个 K 线图，那么同一个画布中就有了 2 个图片，且它们共用一个 X 轴，那么我们需要更改一下基本的参数，代码如下所示：

import mpl_finance as mpf

import matplotlib.pyplot as plt

import pandas as pd

import matplotlib.ticker as ticker

import numpy as np

\#创建绘图的基本参数

fig, axes = plt.subplots(2, 1, sharex\\=True, figsize\\=(15, 10))

ax1, ax2 = axes.flatten()

​

\#获取刚才的股票数据

df = pd.read_excel("同和药业 k.xlsx")

mpf.candlestick2_ochl(ax1, df\["open"], df\["close"], df\["high"], df\["low"], width\\=0.6, colorup\\='r',colordown\\='green',alpha\\=1.0)

df\['date'] = pd.to_datetime(df\['date'])

df\['date'] = df\['date'].apply(lambda x: x.strftime('%Y-%m-%d'))

def format_date(x, pos\\=None):

  if x &lt; 0 or x > len(df\['date']) - 1:

    return ''

  return df\['date']\[int(x)]

df\["SMA5"] = df\["close"].rolling(5).mean()

df\["SMA10"] = df\["close"].rolling(10).mean()

df\["SMA30"] = df\["close"].rolling(30).mean()

ax1.plot(np.arange(0, len(df)), df\['SMA5']) # 绘制 5 日均线

ax1.plot(np.arange(0, len(df)), df\['SMA10']) # 绘制 10 日均线

ax1.plot(np.arange(0, len(df)), df\['SMA30']) # 绘制 30 日均线

ax1.xaxis.set_major_formatter(ticker.FuncFormatter(format_date))

plt.setp(plt.gca().get_xticklabels(), rotation\\=45, horizontalalignment\\='right')

\#显示出来

plt.show()

这里，我们将绘图的画布设置为 2 行 1 列。同时，将上面绘制的所有数据都更改为 ax1，这样均线与 K 线图会绘制到第一个子图中。

接着，我们需要绘制成交量，但是我们知道，一般股票交易软件都将上涨的那天成交量设置为红色，而将下跌的成交量绘制成绿色。所以，首先我们要做的是将获取的股票数据根据当天的涨跌情况进行分类。具体代码如下：

red_pred = np.where(df\["close"] > df\["open"], df\["volume"], 0)

blue_pred = np.where(df\["close"] &lt; df\["open"], df\["volume"], 0)

如上面代码所示，我们通过 np.where(condition, x, y) 筛选数据。这里满足条件 condition，输出 X，不满足条件输出 Y，这样我们就将涨跌的成交量数据进行了分类。

最后，我们直接通过柱状图方法绘制成交量，具体代码如下所示：

ax2.bar(np.arange(0, len(df)), red_pred, facecolor\\="red")

ax2.bar(np.arange(0, len(df)), blue_pred, facecolor\\="blue")

将这 4 行代码，全部加入到 plt.show() 代码的前面即可。

运行之后，输出的效果图如下：

![](https://article-images.zsxq.com/Fj1tRpHR5kyVRLmXTUSFFNmIngfv)

**根据定义编写形态**

接下来，我们根据定义来构造一个简单的形态函数，比如长上影线。

**上影线**

根据网络上的定义：

> ⑤上影阳线：开盘后价格冲高回落，涨势受阻，虽然收盘价仍高于开盘价，但上方有阻力，可视为弱势。

> ⑥上影阴线：开盘后价格冲高受阻，涨势受阻，收盘价低于开盘价，上方有阻力，可视为弱势。

![](https://article-images.zsxq.com/FmpvhHDKlXnu55vjPW_yEEjmXUSN)

![](https://article-images.zsxq.com/FkD7MN7SS4VRdRFv832uDmz1PCci)

那么简单定义为：(high - close)/close > 7%，(high - open)/open > 7% ，可能不是太精确，这里只是为了演示，忽略一些具体细节。

简单理解就是，当天最高价比收盘价，开盘价高得多。

长上影匹配函数，简单，只有一行就搞定。

def long_up_shadow(o,c,h,l):

  return True if (h-c)/c >\\=0.07 and (h-o)/o>\\=0.07 else False

然后把这个长上影函数应用到上面的同和药业。得到的效果：

df = pd.read_excel("同和药业 k.xlsx")

count_num = \[]

for row,item in df.iterrows():

  if long_up_shadow(item\['open'],item\['close'],item\['high'],item\['low']):

    count_num.append(row)

​

plot_image(df,count_num)

![](https://article-images.zsxq.com/Fi9-LVHk3ksa3Hv4XVRhLQ2SZNJZ)

图里面的 4 个箭头长上影是 python 自动标上去的，然后对着数据到同花顺核对一下，都是满足长度大于 7% 的。

可以看到在 3 月 18 日和 28 日的两根长上影之后，同和药业的股价走了一波趋势下跌。【PS：选这个股票并没有刻意去挑选，在集思录默认排名，找了一个跌幅第一个转债的正股就把代码拷贝过来测试】

那么，有这么多的形态，难道每一个都要自己写代码筛选形态吗？ 不用哈，因为已经有人写了第三方的库，你只需要调用就可以了。

截取的部分函数名与说明

![](https://article-images.zsxq.com/FrBZkoLyxp5xztq3U-sWS0hEp3rW)

比如，像两只乌鸦，描述起来都感觉有点复杂，自己手撸代码都会搞出一堆 bug。而用 taLib 这个库，你只需要一行调用代码，就可以完成这些复杂的形态筛选工作。

示例代码：

df\['tow_crows'] = talib.CDL2CROWS(df\['open'].values, df\['high'].values, df\['low'].values, df\['close'].values)

​

pattern = df\[(df\['tow_crows'] == 100) | (df\['tow_crows'] == -100)]

上面得到的 pattern 就是满足要求的形态，如 CDL2CROWS 两只乌鸦。测试的个股是歌尔股份 2020 的日 K 线。

![](https://article-images.zsxq.com/FkHbXFYYLKG2YvDeuMymdzqffUB8)

虽然我不知道两只乌鸦是什么玩意，但是可以通过遍历某个长周期（5-10 年）下的全部 A 股数据，通过数据证伪，统计这个形态过后的一周或一个月（个人随意设定）涨跌幅，来证伪这个形态的有效性。

如果得到的是一个 50% 概率涨跌概况，那说明这个指标形态只是个游走在随机概率的指标，并不能作为一个有效指标参考。

(未完待续） 
 [https://articles.zsxq.com/id_htqhtwu6ppff.html](https://articles.zsxq.com/id_htqhtwu6ppff.html)
