# 最近一个月有股票回购实施的可转债 【tushare使用教程一】
**股票回购**

股票回购是指上市公司利用现金等方式，从股票市场上购回本公司发行在外的一定数额的股票的行为。

通俗理解就是：上市公司花自己的钱买自己的股票

正常而言，有股票回购的上市公司，可偏向解读为利好。

**tushare**

本文借获取股票回购的例子，带大家如何入门使用 tushare 这个库。如果只想看结果的，可以跳过技术部分，直接跳转到文末查看结果即可。

简单来说，tushare 理解为一个股票金融数据获取的第三方库。可以理解为穷人版的 Wind，或者本地版的优矿，聚宽数据获取库。不过 tushare 并没有回测功能，只能提供数据获取。很多时候用 tushare 来获取分析数据也是很方便的。

提供的部分数据如下图示：

![](https://article-images.zsxq.com/FpwrgdHYQAye0dVFfEEfmGJCC6WY)

行情数据包括日 k，涨停跌统计等

![](https://article-images.zsxq.com/FiEHFeLTcvxppfh2SU6u4mqGebcP)

市场数据包括一些财务数据，比如财报，股东数，板块等等

甚至连数字货币的数据也支持：

![](https://article-images.zsxq.com/FqjMbeWqqpOYJvetLByBQUmhpJDH)

官方页面：

[http://tushare.org/index.html](http://tushare.org/index.html)

[https://tushare.pro/document/1?doc_id=129](https://tushare.pro/document/1?doc_id=129)

**安装**

pro 版本的部分数据需要一定的积分才可以获取，并且会限制获取的速度，比如一分钟不能超过 100 次等等，注册后就有会送积分。 笔者使用的是湘财版本的 tushare，它的 token 是没有限制速度和次数的。

首先需要安装，使用国内的源安装，速度会快很多

```
pip install tushare -i https://pypi.douban.com/simple

```

湘财版本的 tushare 安装

```
pip install xcsc\_tushare -i https://pypi.douban.com/simple 

```

初始化：

```
import tushare as ts
ts.set\_token('你的token')
pro = ts.pro\_api() # 初始化接口

```

湘财版本的初始化

```
xc\_server = 'http://116.128.206.39:7172'
xc\_token\_pro = '申请的token'

xc.set\_token(xc\_token\_pro)
pro = xc.pro\_api(env='prd', server=xc\_server)

```

这两个主要初始化的时候不一样。后面使用的值 进行操作 pro 代码就完全一致，不区分普通版本和湘财版本。

**代码操作**

官方的接口 API 文档

![](https://article-images.zsxq.com/FpLqG6dyZFOs_qzPCNn2LpC9pUd4)

![](https://article-images.zsxq.com/Fjy1wJneDzW1d4RCBfZmrcJyvMwz)

![](https://article-images.zsxq.com/FkHNtyJbGRBmTv3bfOeeZW9kyS2J)

![](https://article-images.zsxq.com/FufVD-jjePROLQRL5JuMAdwaPTSw)

基本官方每一个接口都会提供实例教你怎么使用，所以很多使用方法，多看看文档就容易上手。

获取回购数据，把股票代码如 600000.SH 后面的 .SH 去掉。因为后面和集思录的正股代码合并时，集思录的股票代码没有这个后缀。

```
df = pro.repurchase(ann\_date='', start\_date='20211001', end\_date='20211101')
df\['ts\_code'\]=df\['ts\_code'\].map(lambda x:x.split('.')\[0\])

```

获取的数据是 10 月有回购计划或进度的上市公司，表的数据如下。 需要不同月份的数据，可以自行替换开始于结束时间

![](https://article-images.zsxq.com/Fn1GrbDEVg27K7uLMjG01a2FFmrt)

读取集思录的转债数据, 下载集思录的数据文件（数据为 11 月 1 日的收盘数据）。 （文件在附件）

```
import pandas as pd
ROOT\_PATH = r'C:\\data'
excel\_file = os.path.join(ROOT\_PATH, 'tb\_bond\_jisilu.xlsx')
jsl\_data = pd.read\_excel(excel\_file,dtype={'正股代码':np.str})

```

表的数据如下：

![](https://article-images.zsxq.com/Fm6Hc6FmQuC-9iCXBonv9f9MDOV8)

现在有 2 个表，根据他们的相同字段，正股的股票代码来合并，做并集处理，因为并不每个股票都有转债，每个转债的正股未必有回购的操作。

合并操作：

```
merge\_table = pd.merge(df,jsl\_data,left\_on='ts\_code',right\_on='正股代码')

```

![](https://article-images.zsxq.com/FtHQN5ujgOTyopzQ3ahiE8TNIev8)

发现部分同一个转债有多个回购行，是因为它们不同时段有不同的状态，比如新北转债的正股新北洋在 10 月 14 日的进度是预案，10 月 30 日是股东大会通过。

![](https://article-images.zsxq.com/FnGMwLJ_kgBDBZV-d2HgbR17562k)

用 merge_table\['ts_code'].value_counts() 可以把重复的去掉，获得大概有 41 个转债。

笔者认为当前有钱回购公司股票的上市公司，一般不会突然暴雷而没钱还债的，至少在短期内还是会有一波不错的表现。有兴趣的读者可以把每月的回购股票作为回测标的进行收益率回测。 
 [https://articles.zsxq.com/id_34v7tj97xnzk.html](https://articles.zsxq.com/id_34v7tj97xnzk.html)
