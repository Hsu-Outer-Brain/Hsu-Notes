# python代码实现BIAS技术指标： 转债正股的乖离率
乖离率 (BIAS) 是描述股价与股价的移动平均线的相距的远近程度。BIAS 指的是相对距离。

1．BIAS 的计算公式及参数。

### N 日乖离率 =（当日收盘价 - N 日移动平均价）/N 日移动平均价

式中：分子为股价 (收盘价) 与移动平均价的绝对距离，可正可负，除以分母后，就是相对距离。

移动平均价为 1 元时相差 0.1 元，与移动平均价为 10 元时相差 0．1 元是很不相同的，所以在一定场合要用相对距离，不应考虑绝对距离。

BIAS 的公式中含有参数的项只有一个，即 MA。这样，MA 的参数就是 BIAS 的参数，即乖离率的参数就是移动平均价的参数，也就是天数。参数大小的选择首先影响 MA，其次影响 BIAS。一般说来，参数选得越大，则允许股价远离 MA 的程度就越大。换句话说，股价远离 MA 到了一定程度，我们就会认为该回头了，而这个远离的程度是随着参数的变大而变大的。例如，参数为 5 时，我们可能认为 BIAS 到了 4％股价就该回头了；而参数为 10 时，我们则必须等到 BIAS 超过 4％，比方说到了 7％才认为股价该回头。

2．BIAS 的应用法则。

BIAS 的原理是离得太远了就该回头，因为股价天生就有向心的趋向，这主要是由人们的心理因素造成的。

另外，经济学中价格与需求的关系也是产生这种向心作用的原因。股价低需求就大，需求一大，供不应求，股价就会上升；反之，股价高需求就小，供过于求，股价就会下降，最后达到平衡，平衡位置就是中心。

BIAS 的应用法则主要是从以下考虑：

从 BIAS 的取值大小方面考虑。这个方面是产生 BIAS 的最初的想法。找到一个正数或负数，只要 BIAS 一超过这个正数，我们就应该感到危险而考虑抛出，

只要 BIAS 低于这个负数，我们就感到机会可能来了而考虑买入。这样看来问题的关键就成了如何找到这个正数或负数，它是采取行动与沉默的分界线。

上面的是百度出来的 bias 的解释。 也看到少数人用这个指标来根据正股来追踪转债，获得不错的收益率。所以笔者也尝试下该参数的计算。有空再把回测贴上了。

python 代码实现：

```
#计算方法：
#bias指标
#N期BIAS=(当日收盘价-N期平均收盘价)/N期平均收盘价\*100%

df\['bias\_6'\] = (df\['close'\] - df\['close'\].rolling(6, min\_periods=1).mean())/ df\['close'\].rolling(6, min\_periods=1).mean()\*100
df\['bias\_12'\] = (df\['close'\] - df\['close'\].rolling(12, min\_periods=1).mean())/ df\['close'\].rolling(12, min\_periods=1).mean()\*100
df\['bias\_24'\] = (df\['close'\] - df\['close'\].rolling(24, min\_periods=1).mean())/ df\['close'\].rolling(24, min\_periods=1).mean()\*100
df\['bias\_6'\] = round(df\['bias\_6'\], 2)
df\['bias\_12'\] = round(df\['bias\_12'\], 2)
df\['bias\_24'\] = round(df\['bias\_24'\], 2)

```

核心就是上面的几行，压缩下，变成一个函数，df 为正股的日线数据。 N 为要计算的 N 期

```
def bias(df,N):
    label='bias\_{}'.format(N)
    
    df\[label\] = (df\['closePrice'\] - df\['closePrice'\].rolling(N, min\_periods=1).mean())/ df\['closePrice'\].rolling(N, min\_periods=1).mean()\*100
    
    df\[label\] = df\[label\].map(lambda x:round(x,2))
    
    return df.iloc\[-1\]\[label\]

```

当然，写成 map 函数也可以，一行都可以。不过这样追求简洁反而让看的人错过某些细节信息。 python 花式炫技要不得。

有了核心部分，剩下的边角料就一步一步填就好了。

### 获取转债对应正股的代码。

-   方法 1：

集思录和宁稳的数据，数据准确性与更新最快，有过滤强赎公告。 具体操作方式见文：

集思录：

[https://t.zsxq.com/zJM7qVJ](https://t.zsxq.com/zJM7qVJ)

宁稳：

[https://t.zsxq.com/jemEI6y](https://t.zsxq.com/jemEI6y)

日线数据获取：tushare，但需要积分（￥￥￥￥），过滤出来的数据需要通过时间过滤，因为出来的很多历史数据。 之前共享了一个 pro 版本的 token，也可以直接使用，无限调用。

![](https://article-images.zsxq.com/FrZm0cotF-lPjbof4KB42dEiDr5b)

用这个方法本地既可以跑，也可以上 QMT 实盘，用于每天自动选债。

-   方法 2：

量化平台，优矿、聚宽等。

方便，不需要本地安装 python 环境。不过没法每次做成自动化操作，只能每次都要打开网页，点一下运行。无法每天自动运行通知到自己的微信。

下面用方法 2 来实现：

获取所有 转债与正股代码

```
def get\_bonds\_list(beginDate=u"20170101", endDate=u"20201215",EB\_ENABLE=False):
    df = DataAPI.MktConsBondPremiumGet(SecID=u"",
                                       tickerBond=u"",
                                       beginDate=beginDate,
                                       endDate=endDate,
                                       field=u"",
                                       pandas="1")

    cb\_df = df.tickerBond.str.startswith(('12', '11'))
    df = df\[cb\_df\]
    cb\_df = df.tickerBond.str.startswith('117')
    df = df\[~cb\_df\]
    if not EB\_ENABLE:
        eb = df.secShortNameBond.str.match('\\d\\d.\*?E\[123B\]')  # TODO 判断EB是否过滤
        df = df\[~eb\]

    ticker\_list = \[\]
    remove\_duplicated = set()
    for \_, row in df\[\['tickerBond', 'secShortNameBond', 'tickerEqu'\]\].iterrows():
        if row\['tickerBond'\] not in remove\_duplicated:
            ticker\_list.append((row\['tickerBond'\], row\['secShortNameBond'\], row\['tickerEqu'\]))
            remove\_duplicated.add(row\['tickerBond'\])
    ticker\_list = force\_redemption(ticker\_list, beginDate)
    return ticker\_list

# 强赎部分处理下
import pandas as pd
import copy
from datetime import datetime,timedelta

start\_date = '2022-01-14'  # 回测起始时间
end\_date = '2022-01-14'  # 回测结束时间

def get\_last\_trading\_day():
    df = DataAPI.TradeCalGet(exchangeCD=u"XSHG,XSHE",beginDate=start\_date,endDate=end\_date,isOpen=u"1",field=u"calendarDate,prevTradeDate",pandas="1")
    df\['calendarDate'\]=df\['calendarDate'\].map(lambda x:x.replace('-',''))
    df\['prevTradeDate'\]=df\['prevTradeDate'\].map(lambda x:x.replace('-',''))
    return dict(zip(df.loc\[:,'calendarDate'\],df.loc\[:,'prevTradeDate'\]))

canlender\_dict = get\_last\_trading\_day()
force\_redemption\_dict = {}
def force\_redemption(ticker\_list, today):
    global force\_redemption\_dict,canlender\_dict
    ticker\_list\_code = \[\]
    for item in ticker\_list:
        if item\[0\] not in force\_redemption\_dict:
            ticker\_list\_code.append(item\[0\])

    df = DataAPI.BondConvStockItemGet(secID=u"",
                                      ticker=ticker\_list\_code,
                                      bondID=u"",
                                      convID=u"",
                                      field=u"ticker,traStoptime",
                                      pandas="1")

    df = df\[~df\['traStoptime'\].isnull()\]
    if len(df) > 0:
        df\['traStoptime'\]=df\['traStoptime'\].map(lambda x:x.replace('-',''))
        new\_force\_redemption\_notice = dict(zip(df\['ticker'\].tolist(), df\['traStoptime'\].tolist()))
        force\_redemption\_dict.update(new\_force\_redemption\_notice)
    new\_list\_data = copy.deepcopy(ticker\_list)

    for \_ticker in new\_list\_data:
        force\_redemption\_date = force\_redemption\_dict.get(\_ticker\[0\],None)         
        if force\_redemption\_date is not None:
            last\_day\_force\_ = canlender\_dict.get(force\_redemption\_date,None)
            if last\_day\_force\_ is not None and last\_day\_force\_ <= today:
                # log.info('强赎日 {}移除强赎债 {}, {}今日日期{},------强赎前一天的日期{}'.format(force\_redemption\_date,\_ticker\[0\],\_ticker\[1\],today,last\_day\_force\_))
                ticker\_list.remove(\_ticker)

    return ticker\_list

```

```
start\_date = '2022-01-14'
ticker\_list=get\_bonds\_list(start\_date,start\_date) 

```

开始时间与结束时间同一天，start_date='2022-01-14', 因为我们需要计算当前的 bias 的值，所以我们不取历史强赎的转债数据。

取的正股日期，以当前日期往前 N×2 天，比如要求 N=6 的周期，那么就取个 6\*2=12 天的历史日线。 实际正常取 N+3 即可，为了以防节假日的时候数据过少导致 k 线不够。

```
start\_date\_dt = datetime.strptime(start\_date,'%Y-%m-%d') + timedelta(days=-N\*2)
start\_date\_str = start\_date\_dt.strftime('%Y-%m-%d')

```

所以使用一个正股代码求 bias 值的代码为：

```
def get\_zg\_bias(code):
    df=DataAPI.MktEqudAdjGet(secID=u"",ticker=code,tradeDate=u"",beginDate=start\_date\_str,endDate=u"",isOpen="1",field=u"closePrice,tradeDate",pandas="1")
    return bias(df,N)

```

批量取所有转债正股的 bias 只需要一个循环：

```
ticker\_list=get\_bonds\_list(start\_date,start\_date) 
bias\_list =\[\]
N = 6

for bond in ticker\_list:
    \_bias = get\_zg\_bias(bond\[2\])
    bias\_list.append({'zg\_name':bond\[1\],'code':bond\[0\],'bias':\_bias})

```

把结果转为 dataframe

```
bias\_df = pd.DataFrame(bias\_list)

```

得到的是 6 日均线的 bias 值排序，看看结果：

当前 bias 最大的 10 值。

![](https://article-images.zsxq.com/Frn4w5raUenWI5V22Xl5zeLcbSHP)

最大的居然是亚药转债的正股，bias 值为 25.14. 上表中最大的在最底下。 接着依次是万孚转债，润达转债的正股。

6 日均线的 bias 更多应用在短线上，选出来的都是多头均线排列。当然这时更加需要溢价率的配合，结合溢价率，把高溢价率的去除。

不然 100% 的溢价率，即使正股继续涨停，也得要涨 100% 多才能消化掉这个溢价率。

![](https://article-images.zsxq.com/FrGG1ABMtQyqKQETPCBrgZ_FpjaM)

计算转债溢价率，然后把高溢价率的去除，这里阈值随意给一个，比如过滤掉 30%

```
def getBondPremRatio(codelist):
    return DataAPI.MktConsBondPerfGet(beginDate=start\_date,endDate=start\_date,secID=u"",tickerBond=codelist,tickerEqu=u"",field=u"tickerBond,bondPremRatio",pandas="1")

```

溢价率与 bias 合并

```
join\_df = pd.merge(bias\_df,bondPremRatio\_list,left\_on='code',right\_on='tickerBond',how='inner')

```

过滤溢价率大于 30% 的，bias 值最大的 10 值

```
join\_df\[join\_df\['bondPremRatio'\]<30\].sort\_values('bias',ascending=False).head(10)

```

![](https://article-images.zsxq.com/FqCcc9Zd9U6bZjd45KkOvQZ8sCC-)

然后正股下降趋势的是 bias 值倒数 10 名：

![](https://article-images.zsxq.com/FrybM4RrCKqYyBnAwDEw_lbzPaKf)

随便找一只对应的正股看看，小康股份

![](https://article-images.zsxq.com/FokqJcuKp1gXhOh1AzdmQG1gldAU)

目前均线是空头排列。但是价格和溢价率都是相对较高，这种下降走势就不取抄底转债了。 要博反弹的话，可以把价格控制一下，最好在 160 以下的，越靠近债底安全较高。

如果需要计算更多均线的乖离值，可以尝试修改 N 的值即可。 
 [https://articles.zsxq.com/id_33dqnll0mwqc.html](https://articles.zsxq.com/id_33dqnll0mwqc.html)
