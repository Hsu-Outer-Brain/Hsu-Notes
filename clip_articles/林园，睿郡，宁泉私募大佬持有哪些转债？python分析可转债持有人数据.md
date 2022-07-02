# 林园，睿郡，宁泉私募大佬持有哪些转债？python分析可转债持有人数据
在企业的年报，半年报里会有可转债的十大持有人变动。

在集思录的收费数据里面也有对应的数据，数据按半年或年收费。

不过很多财经的门户网站（东财，同花顺）免费提供对应的数据，用 python 循环一遍就可以很简单的获取到。

授人与鱼不如授人于渔。

![](https://article-images.zsxq.com/FiZM1MDwq3ng17QmId0FgsyGFkqm)

获取到的数据（存储到 MongoDB 数据库）：

![](https://article-images.zsxq.com/Fv4AU2km0cdIDpSeA1zqHpG6cS3J)

code 字段是可转债的代码，name 是持有人的名称，amount 是持有金额，holding_ratio 是持有比例

本身笔者用的存入 mongo 数据库，不过考虑到读者的方便，稍微修改了版本，最后结果是输出到 excel 文件中，本文附件也提供了获取到的所有数据。

数据获取不是本文的重点，所以就一笔带过。具体代码在附件里面提供

### 数据分析

有了上面的数据，我们接下来就可以慢慢折腾上面的数据。

-   林园私募基金

最近几个月可转债的不断有公告，林园的私募产品买入超过当前转债规模的 20%，所以分析下目前林园持有的转债了列表：

```
df\[df\['name'\].str.contains('林园')\]\[\['zz\_name','name','holding\_ratio'\]\]

```

![](https://article-images.zsxq.com/FnQGJzR3I1xDIWvAlsAPabC5Rck6)

![](https://article-images.zsxq.com/FrIdaEMakPVUoL74-Qq57XaJ6QMd)

![](https://article-images.zsxq.com/Fu4KLL2k3ixfaQ9lCNd1OELn5DXT)

林园的私募命名都是 XX 好私募证券投资基金，看着好像已经到了 200 多号了。感觉是摊大饼的策略，所以以后如果看到林园某个私募业绩很猛，千万不要惊讶，很可能是踩到像小康，百川这样的转债，如果他的每个私募产品的规模不大的话。

因为上面的转债的规模都是偏小居多，所以稍微买多以点就会上十大了。 这些资金如果买入 AAA 大盘债，可能也就溅不起一点水花了。这也让一些人误认为林园的私募只买入小规模转债。不过可以看出林园买入的可转债普遍价格都不高，并且没有出现在规模更小的横河，蓝盾这些妖债上，看来林园当前并不参与妖债的炒作。

把上面的同一个转债的数据合并一下，看看每一只转债的林园私募的占比

```
df\[df\['name'\].str.contains('林园')\]\[\['zz\_name','name','holding\_ratio'\]\].groupby('zz\_name').sum('holding\_ratio')

```

![](https://article-images.zsxq.com/Ftm65P9AorVvgEgciekhYwP0M7TL)

**林园私募持有的转债**

占比最高的是纵横转债 16.99%，华体转债 15.06%，最低的是花王转债，只有 0.97%。

把持有比例转换为持有金额：

![](https://article-images.zsxq.com/FjQxgum9CDDVvibSaqr9WusFaICx)

最多的还是纵横转债，为 45 万张，注意这里的单位是张， 按照当前的价格算，大概也是 4 千万多的市值。最小的正裕转债，1 万 8 千张，市值约 200 万。 而根据当前林园 260 多亿的私募规模（2020 年底的数据），这点持仓真的只能是毛毛雨。

-   其他私募数据

除了平时比较高调的林园，还有哪些私募介入到转债十大持有人里面呢？

继续用 python 分析

```
def get\_invest\_company(x):
    return x.split('-')\[0\]

institute\_counter ={}
for index,row in df\[df\['name'\].str.contains('私募')\].iterrows():
    institute\_counter.setdefault(get\_invest\_company(row\['name'\]),0)
    institute\_counter\[get\_invest\_company(row\['name'\])\]+=1

```

通过过滤名字里面带有私募字样的产品，得到：

![](https://article-images.zsxq.com/FryIAhRw2KdYO4jRTTU5IiwoeZOQ)

总共录得有 60 多只私募上榜十大持有人，取其产品持有个数大于 3 的私募如上。

林园还是遥遥领先。排名第二到第五的私募是 宁泉，明汯，对外信托，睿郡。宁泉是 18 年新晋私募（兴全总经理杨东总奔私后成立的私募），部分产品的业绩排名 Top10。

随意翻了一下其持有的转债 (就是文章开头那个图片)

![](https://article-images.zsxq.com/FgTL7v-HyLB1VRPJG219cFKgNncr)

这个转债的十大持有人被它包办了。持仓公告时间是 2021-06-30 的。

猜是哪个转债？

答：亚药转债。 市场价格吊车尾的转债，常年活跃在 80 元以下，最近反弹了一波，涨到了 85 了。

按照上面的方法，统计下宁泉持有的转债：

```
df\[df\['name'\].str.contains('宁泉')\]\[\['zz\_name','name','holding\_ratio'\]\].groupby('zz\_name').sum('holding\_ratio')

```

![](https://article-images.zsxq.com/Fh1Gl5FO6G55D1yjd3U2neSwUnVH)

**宁泉私募持有转债**

当前宁泉持有 25% 的寿仙转债，而亚药转债的比例则为 13%，共 120 万张，市值约 1 亿，比林园最大的还要大一倍。

同样，只要把上面的查询名字替换成其他名字，比如 睿郡 （也是前兴全可转债的大佬出走后创立的私募），就可以查到改私募最大的可转债仓位。睿郡最大持有的是大秦转债，约 800 万张，单个持有市值比前面两位还要大得多。

![](https://article-images.zsxq.com/Ft5908W6OiIgGGw9yayMFYzS8eY0)

下一篇会分析一些牛散的关联关系以及妖债的持有人特点。 
 [https://articles.zsxq.com/id_ipbvdd2goxrf.html](https://articles.zsxq.com/id_ipbvdd2goxrf.html)
