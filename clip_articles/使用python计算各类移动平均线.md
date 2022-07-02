# 使用python计算各类移动平均线
【此文适合之前没有做过类似股票分析的读者朋友，高手请忽略】

计算移动平均线是最常见的需求，下面这段代码将完成以下三件事情：

1. 从 csv 格式的文件中导入股票数据，数据例图如下：

2. 计算各类移动平均线，包括简单简单算术移动平均线 MA、指数平滑移动平均线 EMA；

3. 将计算好的数据输出到 csv 文件中。

_"""_

_@author: 量化自由之路_

@contact: wx： rockyzsu

_"""_

```
import pandas as pd

```

\# ========== 从原始 csv 文件中导入股票数据，以浦发银行 sh600000 为例

\# 导入数据 - 注意：这里请填写数据文件在您电脑中的路径， 数据可以查看附件

```
df= pd.readcsv('600000.csv',index\_col=0)

```

![](https://article-images.zsxq.com/FtsUIinfms4M7-_-YOZtxAljfMTG)

\# 将数据按照交易日期从远到近排序

```
df.sort\_values(by='tradeDate',inplace=True)

```

\# ========== 计算移动平均线

\# 分别计算 5 日、20 日、60 日的移动平均线

```
ma\_list = \[5, 20, 60\]

```

\# 计算简单算术移动平均线 MA - 注意：df\['closePrice']为股票每天的收盘价

```
for ma in ma\_list:
 df\['MA\_' + str(ma)\] = df\['closePrice'\].rolling(ma).mean()

```

\# 计算指数平滑移动平均线 EMA

```
for ma in ma\_list:

  df\['EMA\_' + str(ma)\]= pd.DataFrame.ewm(df\['closePrice'\], halflife=ma).mean()

```

\# 将数据按照交易日期从近到远排序

\# 得到的新数据: 用 df.tail() 查看最后 5 条

![](https://article-images.zsxq.com/Fm3WHOr2uzTmm405g2lelqkEx-fi)

\# 还可以简单的画画图，看看这几天均线的走势

```
import matplotlib.pyplot as plt
plt.figure(figsize=(10,16))
df\[\['closePrice','MA5','MA20','MA\_60'\]\].plot()
plt.show()

```

![](https://article-images.zsxq.com/Fkv9t7ew0oR5--ePDIkijAJ0QpFn) 
 [https://articles.zsxq.com/id_r8vqfb60gwrw.html](https://articles.zsxq.com/id_r8vqfb60gwrw.html)
