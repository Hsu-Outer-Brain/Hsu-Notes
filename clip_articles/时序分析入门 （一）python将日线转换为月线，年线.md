# 时序分析入门 （一）python将日线转换为月线，年线
时序数据，也就是时间序列的数据。

像股票价格、每日天气、体重变化这一类，都是时序数据，这类数据相当常见.

本文件就使用 python 的 pandas 对时间序列做个入门级的教程。也是股票分析里面经常用的, 把日线数据变成月线，年线，或者任意周期。 同理也可以把 tick 级别的分时数据合成为日线数据。

载入日线数据，随意找了一个可转债的日线数据，包含了很多的字段，有开盘价，收盘价，成交量，换手率，YTM 等。 csv 文件在附件里可以找到。

```
import pandas as pd
df = pd.read\_csv('113525.csv')

```

![](https://article-images.zsxq.com/FiHaBdqdmLnyJKxtkFX79j0GsE_r)

日线数据从 19 年到今年 21 年 9 月。

默认情况下，pandas 导入的列数据的格式都是靠运气识别的。可以用 df.info() 来看看它们的数据格式。

一般情况下识别是没有问题的。不过时间格式默认是被识别为字符串的。所以需要对日期列进行下转换。

比如上面的 tradeDate 字段，为交易日字段。

![](https://article-images.zsxq.com/FtnM5f1tdCgODSHIYwhyNNgQUU9n)

被识别为了 object 类型。在 pandas 里面，object 类型是通用类型。先对该字段做类型的转换后再运算。

把该列转换为时间类型。

```
df\['tradeDate'\]=pd.to\_datetime(df\['tradeDate'\])

```

这个语句在时序分析里面会经常用到的。

然后在使用 df.info() 查看类型

![](https://article-images.zsxq.com/FqqlmDGDPCJEBn-qYq8w5YXKOsZq)

该列的类型已经变为了 datetime64 格式了。

pandas 的时序操作需要把时间设置在索引 index 上。

```
df.set\_index('tradeDate',drop=True,inplace=True)

```

上面语句把 tradeDate 设为索引，drop 为 True 把原来那个默认索引删除，inplace 为 True，则是在原来的 df 上原地修改，不产生新的变量。

![](https://article-images.zsxq.com/Fl0VT9_hBqcMKc7haMlN2cALr0Al)

上图可以看到 tradeDate 交易日已经在前面，成为了索引

如果我们需要把数据合成月线级别。需要用 resample 重采样。

```
resample\_data = df.resample('M')

```

后面的 M 代表 Month，按月采样。 如果要换成年数据，则把 M 改为 A-DEC （DEC 是 12 月的缩写）

此时，resample_data 已经安装月分组好数据了。

需要进一步对不同字段进行运算。

```
resample\_data\['turnoverVol'\].sum()

```

比如上面的是统计月的成交额的总和。

![](https://article-images.zsxq.com/FmmSdRyv3-tKT4w-EMVy9TI9B5wf)

这里它把每个月的成交量全部累加起来，前面的日期是当月的月底最后一天。

如果想找出哪个月的月总成交量最大和最小

```
resample\_data\['turnoverVol'\].sum().argmax()

```

得到

![](https://article-images.zsxq.com/FnRnRXceCJQ4Xa8-j0M3luDb1Lou)

结果显示是当前 9 月的成交量最大，虽然这个月还没有结束 (数据还是截止到 9 月 15 日的)，但是统计的时候也是按照一个月来算的。然后可以在原数据中可以查到 21 年 9 月份(截止到 15 日) 的成交量达到了 500 多万手。

同理，成交量最小的月是

```
resample\_data\['turnoverVol'\].sum().argmin()

```

![](https://article-images.zsxq.com/Ftoqzfg6oGnF9eQTcM3Rf0IDd69S)

查询到最小的月份是 19 年 11 月，当时成交量只有 3 万手左右。

上述函数的 sum 叫聚合函数，resample 也可以理解为分组。分组后，使用聚合函数，将每一组聚合起来。

所以除了 sum 求和，还有很多聚合函数可以使用，比如 mean 求平均，median 求中位数，min/max 最小 / 大值。

比如：

```
resample\_data\['closePrice'\].mean()

```

得到的是 每个月的收盘价的平均值。

![](https://article-images.zsxq.com/Fgx8gOoS3BCo3x82kT-8agXeO3h8)

可以看到的这个台华转债一直在 120 以下排行，然后一下子就爆发了，9 月份均价就上到 200 了。

dataframe 有了这个时间索引后，查找区间的数据也就是很方便了。

比如我想找 8 月 15 到 9 月 15 日之间的所有成交量：

```
df\['2021-08-15':'2021-09-15'\]\['turnoverVol'\].sum()

```

使用索引值：'2021-08-15':'2021-09-15' 就可以方便查找任意时间的数据。

```
df\['2021-05-15':'2021-09-15'\]\['dealAmount'\].argmax()

```

上面语句查询 5 月 15 日到 9 月 15 日中，哪天的成交笔数最大。

得到的是 '2021-09-07’， 说明 9 月 7 日当天成交笔数多，也就是交易极其活跃，散户参与度高。

所以你可以尝试计算下，平均每笔的金额，可以大概算出大资金在哪天进入某个转债。如果平均每笔的金额越多，所以大资金的可能性越大，然后排下序，找出最大的那几天，看看后续的股价变化。

另外如果默认的聚合函数不能满足你的操作，可以自定义分组和聚合方法，比如统计每周一的涨跌幅的平均值，与 “黑色星期四” 的平均涨跌幅。通过历史数据大概就知道，是否真的有黑色星期四。

下一篇待续 
 [https://articles.zsxq.com/id_pq2w5ujfbk7h.html](https://articles.zsxq.com/id_pq2w5ujfbk7h.html)
