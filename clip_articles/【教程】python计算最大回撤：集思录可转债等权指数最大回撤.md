# 【教程】python计算最大回撤：集思录可转债等权指数最大回撤
在计算收益率指标时，经常会提到最大回撤这个指标。部分新人或者接触的不多的人可能对着认识不深或者不知道怎么计算的，本文就做个小教程，介绍怎么计算，以集思录的等权指数为例。其实不少人应该也不知道 18 年到现在，集思录的等权指数的最大回撤有多大。

假设下面是一个收益率曲线图，请问最大回撤是哪段哪点到哪点呢？

![](https://article-images.zsxq.com/Frdvc-pG_ceori5kEz6Qwd6U9Wt_)

第一步，找到图中的：有 A、C、E、G、I 五个点；

第二步，找到局部高点对应的后续最低点，分别是 A→F、C→F、E→F、G→H、I→J。注意 A 对应的后续最低点是 F，而不是 B，因为 F 比 B 点更低。同样，C 点的后续低点也是 F，而不是 D 点，因为 F 比 D 点更低。

第三步，找出局部高点到后续最低点的最大跌幅，比较之后，显然是 C→F 的跌幅是最大的，按照定义，最大回撤就是 C 到 F 的下跌幅度。

使用 python 编写的话：

```
def get\_max\_withdraw(indexs):
    max\_withdraw = 0
    max\_date\_index =0
    last\_high = indexs\[0\]
    
    for index,current in enumerate(indexs):
        # 遍历所有数据
        if current>last\_high:
            last\_high=current
            continue

        if (last\_high-current)/last\_high>max\_withdraw:
            # 找到一个最大值时，保存其位置
            max\_withdraw = (last\_high-current)/last\_high
            max\_date\_index=index

    return max\_withdraw\*100,max\_date\_index # 变成百分比


```

数据源是集思录的指数页面：

![](https://article-images.zsxq.com/FkHMnmsTWDzI1UnvpCwnKsVD126K)

把数据复制下来保存到 excel，不是高频使用的话，手工操作即可。

```
df = pd.read\_excel('指数.xlsx',encoding='utf8')

```

如果使用代码的话，可以用一行语句：

```
import pandas as pd
df = pd.read\_hmtl(xxxx) # xxxx 为等权指数的链接

```

或者用鼠标复制所有的行，然后使用命令

```
df = pd.read\_clipboard()

```

也把所有数据弄下来了。

然后转为列表：

```
indexs=df\['指数'\].values
indexs=indexs\[::-1\] # 把数据反转下，时间最早的放在最前面

```

调用上面的求最大回测的代码里面：

```
get\_max\_withdraw(indexs)

```

得到集思录的可转债等权指数的最大回撤是：

**12.01%**

这是集思录等权指数从 2018 年到 2022 年 1 月 20 日的最大回撤值，索引位置在 755，通过

```
iloc\[len(df)-755\]

```

得到回撤最大的位置在 2021-02-04，

也就去年年初的那一轮暴跌的时候发生的。

![](https://article-images.zsxq.com/FlcqikVyJA7_bc32oie5c4QRHV8M) 
 [https://articles.zsxq.com/id_fw88eiigwd2i.html](https://articles.zsxq.com/id_fw88eiigwd2i.html)
