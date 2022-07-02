# python量化回测 - 可转债转股套利是一门亏钱手艺吗 附源码与数据
可转债转股套利，一个月经贴经常出现在集思录上话题。

![](https://article-images.zsxq.com/Fl8NSEsHYz5VOVe4DCJ7bm5aZEaH)

![](https://article-images.zsxq.com/Fj60eObLRr3mGPLwcVFDABdtLVk1)

老韭菜会说，亏钱手艺。

没尝试过的新韭菜蠢蠢欲动，哇，-5% 的折价，这套利真香。

昨天的小康转债溢价 - 4.5%，如果转股后今天的小康股份是这样的

![](https://article-images.zsxq.com/FrN0gRrz72RgeOufEBqMgEKJyM34)

一字板跌停开盘，如果套利开盘卖出，一碗 - 5.5% 的大面。

最近的今飞转债也有人套利

![](https://article-images.zsxq.com/FjZuJMt7mz6xuSlrrytA-wAJ4Ojs)

几乎都是面。

今天的福莱特也差不多。

![](https://article-images.zsxq.com/Fv7TQzjYTtosf-BLvPRxQhktceoJ)

接下来，笔者利用 python 与量化平台的数据进行一个回测，可转债转股套利到底能否赚钱，期望收益是多少。

抛转引玉顺便教大家如何使用 python 做量化分析，也希望有经验的朋友可以互相交流讨论。

#### 获取数据

因为量化平台的函数都大同小异（其实都是互相抄袭国外的量化平台吧），笔者挑一个优矿来回测。

**可转债代码**

首先要获取市场上所有可转债的代码

```
df = DataAPI.MktConsBondPremiumGet(SecID=u"",tickerBond=u"",
                                   beginDate=u"20170101",
                                   endDate=u"20201203",
                                   field=u"",
                                   pandas="1")

```

beginDate=u"20170101",endDate=u"20201203", 这两个参数分别是在这段时间内的所有可转债数据。

得到的结果如下：

![](https://article-images.zsxq.com/FhR1AJfjo3W-QSYkNH-OldRTu0b1)

因为里面包含了一些可交换债，国债，那么先对数据进行清洗，只保留可转债。 根据证券代码的前 2 位就可以过滤掉无用的数据。

深市可转债是 12 开头，沪市可转债是 11 开头。

那么过滤语句如下：

```
cb\_df = df.tickerBond.str.startswith(('12','11'))

```

上面是根据**tickerBond**这个字段（证券代码）进行过滤

过滤后的数据使用布尔选择取出来：

```
cb\_df\_ = df\[cb\_df\]

```

这样就得到了所有的可转债的数据了。

有一个字段**bondPremDisc** ，官方文档提示是 溢价率

```
bondPremDisc	float	可转债折溢价，可转债收盘价-(转股价/正股价)\*100

```

但笔者用回测平台返回的数据跟笔者日常抓取的历史数据有很大的出入：

![](https://article-images.zsxq.com/Fmc9RppsJo1I78OQyI_D8iqkJx5l)

量化平台返回的数据

像上面这个奇正转债，溢价率是 25.5631%，日期是 12 月 2 日，而笔者自己获取的历史数据是 31.2%，或者到集思录平台上查历史数据也是显示 31.2%。

![](https://article-images.zsxq.com/FiLUadE8WOtsWnUtK5DLxb2dwQhg)

 个人自有数据

![](https://article-images.zsxq.com/FsyTHY1GLtDohKMzeGJNJOBLFNn8)

集思录数据

因为刚开始笔者也是用这个数据进去回测的，得到最后拿到结果后发现收益率是一个高的离谱的数据，所以在一步一步调试过程中发现它的这个溢价率数据是有问题的，所以量化分析的第一步是先要确保数据的准确性，要在中间阶段就进行数据核对，一旦发现数据不对劲，就需要及时调整，不然后面的流程即使是对的，得到的结论也可能是错的。虽然过程很繁琐，但是却是很有必要的，拼的就是耐心。

所以不能使用上面的数据进行获取溢价率，但上面的数据可以获取所有的转债代码。

```
ticker\_list = cb\_df\_\['tickerBond'\].unique()

```

**转股起始日**

接下来要获取每个转债的转股起始日，因为在所有交易日中，不一定所有的交易日都是可以转股的，发行后的半年内是不能转股的。

```
def filter\_day(ticker):
    # 转股日
    convert\_date = DataAPI.BondConvStockItemGet(secID=u"",ticker=ticker,bondID=u"",convID=u"",field=u"convStarttime",pandas="1")
    convert\_date = convert\_date\['convStarttime'\].values\[0\]
    return convert\_date

```

上面接口提供的是转债基本数据，可以传入一个证券代码，然后就可以获取到转股时间

![](https://article-images.zsxq.com/Fs7uUPPPLJDhBVJBydttOlYmykvJ)

**可转债溢价率**

因为前面验证了之前那个溢价率数据是错误的，那么要想办法获取到证券的溢价率。

找到一个日线数据，里面居然也有一个溢价率的字段

```
bond\_df = DataAPI.MktConsBondPerfGet(beginDate=beginDate,endDate=endDate,secID=u"",tickerBond=ticker,tickerEqu=u"",field=\['bondPremRatio','closePriceBond','tradeDate','tickerEqu','secShortNameBond'\],pandas="1")

```

得到数据与真实数据对比一下，结果是吻合的，那么就可以用上面的函数获取每一个转债的溢价率。

![](https://article-images.zsxq.com/Fk2PWM9zQ9O00Dd_Gj8Gq0MozFGS)

然后就把溢价率小于 0 的过滤出来：

```
percent=0
df = df\[df\['bondPremRatio'\]<percent\]

```

笔者把 percent 提取出来，因为后面可以方便的进行其他试验，比如笔者后续要改成溢价率小于 - 1 的时候才进行套利，只需要在这里 percent=-1 ，很方便就把溢价率小于 - 1 的转债日线过滤出来。

然后就拿到了所有的负溢价的转债了吧。 No，这里有个坑。 因为得到的数据里面，有一些脏数据需要处理。 这个坑也是自己在查看结果是发现的。

![](https://article-images.zsxq.com/FtXwEQnxf1jwswM2JyCfeF7d7zW2)

看到上面的图，发现什么问题了吗？

和尔转债在 2 月 5 日停止了交易，但是后面可以转股。但是接口返回的数据里面可转债的价格不变，但是溢价率确实一直在变，最后是到 2 月 27 才退市的，而这些日子里，和而泰的股价是一直涨的，所以它的溢价率是不断的折价，最后是到了 - 30%，所以需要把停止交易后的这段数据去掉。

```
    df\_shift = df.shift(1)
    df = df\[df\['closePriceBond'\]-df\_shift\['closePriceBond'\]!=0\]

```

笔者使用的方式是先按收盘价移动一位，然后相减，如果得到的数据是 0，那么认为这两个后面的数据是停止交易的了。取反就是拿到正常交易数据。

这样就得到所有转债的溢价率数据了。

**正股第二天的开盘价**

这里的回测，笔者使用的折价转股，第二天以开盘价卖掉，当然你可以以其他价格卖也行，比如收盘价。

那么接下来需要获取当前日期的第二天的日期。

```
import datetime
def next\_day(s):
      next\_date = datetime.datetime.strptime(s,'%Y-%m-%d') + datetime.timedelta(days=1)
      next\_date\_weeks = datetime.datetime.strptime(s,'%Y-%m-%d') + datetime.timedelta(days=14)
      return next\_date.strftime('%Y-%m-%d'),next\_date\_weeks.strftime('%Y-%m-%d')

```

这里先获取当前日期的下一天，然后再往后取 2 周，为什么这么做呢？ 因为如果当然日期是国庆前最后一天，那么它的下一个交易日并不是在当然日期直接加 1 天，所以用了一个区间范围去取值，比如当前日期是 9 月 29 日，那么下一天是 9 月 30 日，然后在取接下来的 2 周，到 10 月 13 日，然后取获取区间【9 月 30 日，10 月 13 日】这个时候正股的数据，然后取它的第一条（这里是 10 月 8 日），也就是 9 月 29 日的下一个交易日数据了。

获取正股日线数据：

```
zg\_df = DataAPI.MktEqudGet(secID=u"",ticker=zg\_ticker,tradeDate=u"",beginDate=\_next\_day,endDate=\_next\_week,isOpen="",field=u"",pandas="1")

```

![](https://article-images.zsxq.com/FsRABlU7H1S7qaJUBfvcHFfOxzbB)

正如上面说的，这里获取的数据是一个区间，所以拿第一条数据

```
open\_date\_series = zg\_df.iloc\[0\]

```

然后只需要今天的开盘价，与昨天的收盘价。

```
def open\_raise(opened,last\_closed):
    return (opened - last\_closed)/last\_closed\*100.0

```

那么就得到了开盘卖出的收益率。

但是，这里就又有一个坑，因为从结果看到，部分套利日最高的收益达到了 99%，显然不太靠谱，经过查找数据所在的行，发现部分脏数据在里面，需要额外清洗的，哎，坑真多呀。

![](https://article-images.zsxq.com/Fp8zKXV1RjXGHgxu_tw_l9SLFaF3)

上面这个东音股份，开盘价是 0，最高最低也是 0，说明这个票当天可能停牌了，所以这个接口数据返回的日线数据居然停牌的也返回？ 去。。。 所以价格过滤条件吧，吧开盘，收盘价格是 0 的一律返回 0，不做除法运算。

**最终收益率**

最后的收益率是转股溢价的收益 + 第一天开盘涨幅

```
\-1\*convert\_profit + open\_profit

```

PS：如果你持有的转债数目较少的话，因为转股不一定全部金额会转为正股，剩余的会变成现金返回你账户。

主要核心就是上面的几个流程，然后把所有代码拼接起来，因为笔者都写成模块化的函数，所以直接调用就可以了。

```
\# main
beginDate='2017-01-01'
endDate='2020-12-01'
percent =0
ticker\_list = bonds()
convert\_date\_dict ={}
for ticker in ticker\_list:
    convertdate = filter\_day(ticker)
    convert\_date\_dict\[ticker\]=convertdate

profit\_result =\[\]
for ticker in ticker\_list:
    bond\_df = bond\_trade\_data(ticker,beginDate,endDate,percent)
    bond\_df = is\_convertable(bond\_df,ticker)
    for ids,row in bond\_df.iterrows():
        bondPremRatio=row\['bondPremRatio'\]
        tradeDate=row\['tradeDate'\]
        tickerEqu=row\['tickerEqu'\]
        secShortNameBond=row\['secShortNameBond'\]
        p = profit(zg\_ticker=tickerEqu,start\_date=tradeDate,convert\_profit=bondPremRatio)
        d={'ticker':ticker,'tradeDate':tradeDate,'profit':p}
    profit\_result.append(d)

```

最后数据保留在 profit_result 里面。

上面代码获取的是 17 年 1 月到 20 年 12 月 1 日的数据。

**结果分析**

你看，其实数据分析大部分时间都是花在数据收集，清洗上面。到最后分析阶段，倒是最简单的了

```
import pandas as pd
profit\_df = pd.DataFrame(profit\_result)
profit\_df\['profit'\].min()
# 最大亏损
-9.85428

profit\_df\['profit'\].max()
# 最大收益
.7600433

profit\_df\['profit'\].mean()
# 平均收益率
0.28580

profit\_df\['profit'\].argmin()
# 最大亏损所在的行
1722

profit\_df.iloc\[1722\]

profit                -9.854%
secShortNameBond          寒锐转债
ticker                  123017
tradeDate           2020-01-23


profit\_df\['profit'\].argmax()
# 最大收益所在的行
2236

profit                   13.76%
secShortNameBond          模塑转债
ticker                  127004
tradeDate           2020-02-03

```

上面的分析结论是： 如果出现负溢价就转股套利，最终的平均收益率是**0.28580%**，如果扣除手续费，估计也剩多少的了。

套利亏损最大的是在今年 1 月 23 日的寒锐转债，因为正股寒锐钴业第二天几乎是跌停开盘，所以亏损值为 **-9.854%**。

**套利盈利最大的是今年 2 月 3 日的模塑转债**，因为当时疫情原因，当天几乎很多个股接近跌停开盘，而模塑技术居然是涨停开盘的，所以溢价 - 3% 加上开盘涨停的 10%，所以收益达到了 13%。

因为之前很早前做过类似的回测，只是没有写文章分享。 笔者一般的做法就是持有转债不要转股，等待溢价收敛，如果是高价转债（价格大于 145 以上），就不要去参与了。 持有转债，第二天如果正股跌停，转债的依然可以从容跑路。

完整 python 代码见附件

结果数据是 csv 格式，用 excel 打开如果中文乱码选择 gbk 或者 utf8 编码即可。 
 [https://articles.zsxq.com/id_nk7t7w7wetwl.html](https://articles.zsxq.com/id_nk7t7w7wetwl.html)
