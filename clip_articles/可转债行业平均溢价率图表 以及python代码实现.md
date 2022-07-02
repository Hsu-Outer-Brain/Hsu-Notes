# 可转债行业平均溢价率图表 以及python代码实现
首先展示下效果

![](https://article-images.zsxq.com/FrRwMw4bisUSc9CH-VarxOdYkvQC)

这个数据是实时爬取的，也就是数据是最新的。数据源是集思录。

从转债的行业平均溢价率大概可以看到，当前哪些板块是热门板块。

而在选择转债标的时候，应该优先选择热门板块中的转债，或者选择潜伏在里面还没有起涨的转债，或者是里面的低溢价率，双低转债，低价转债。

当前行业平均溢价率较低的板块依次是钢铁，化工，军工，电气，电器，有色，汽车。

垫底的是传媒，休闲服务，通信，纺织服装，商业贸易。

#### 实现流程

现在的集思录的数据需要登录才可以获取，可以参考文章

[python 自动登录集思录并获取完整数据](https://t.zsxq.com/BiIiuV3)

和

[python 自动登录集思录并获取完整数据 (二)](http://xn--4kq/)

#### 流程

流程是先登录集思录，然后获取行业的数据标号，然后逐个行业编号请求一次，就可以拿到分类数据。

![](https://article-images.zsxq.com/Fuim_Vig6rIZMug6QX00X7sAgixV)

编号数据在集思录数据页面的源码中。

![](https://article-images.zsxq.com/Fkim5-jQNAWTubRIi2PA2VF7pdMG)

data-level 是层级关系，一级行业的值为 1，子行业递增，为 2,3. 这里可以获取一级行业，比如农林渔牧，采掘，化工等。

content 为上面的文本，使用 xpath 进行提取。

```
resp = parsel.Selector(text=content)
nodes = resp.xpath('//option\[@data-level="1"\]')
result\_list=\[\]
for nod in nodes:
    value = nod.xpath('.//@value').extract\_first()
    name = nod.xpath('.//text()').extract\_first()
    name = name.split('(')\[0\]
    result\_list.append((name,int(value)))

```

最后得到 result_list 就是存放的行业名称与标号，用于后面请求集思录的时候作为参数使用。

拿到标号数据，比如采掘行业是 21，那么在请求时把 21 传入到 industry_id 即可。返回的就是某个板块的转债数据。

```
def get\_bond\_info(self, session, industry\_id):
    ts = int(time.time() \* 1000)
    url = 'https://www.jisilu.cn/data/cbnew/cb\_list/?\_\_\_jsl=LST\_\_\_t={}'.format(ts)
    data = {
    "fprice": None,
    "tprice": None,
    "curr\_iss\_amt": None,
    "volume": None,
    "svolume": None,
    "premium\_rt": None,
    "ytm\_rt": None,
    "rating\_cd": None,
    "is\_search": 'R',
    "btype": None,
    "listed": 'Y',
    "qflag": 'N',
    "sw\_cd": industry\_id,
    "bond\_ids": None,
    "rp": 50
    }

```

比如通信的是 73，请求的数据包把 sw_cd 复制为 73 即可得到通信的板块转债数据

得到的 8 个通信转债如下：

![](https://article-images.zsxq.com/FnhdaTzYBPmUy77PSnEju1xJ6raH)

拿到数据后做下求平均值即可使用数据绘制出文章开头的图片。

另外还可以做其他的选债动作。

比如我想找到低于行业平均溢价率，价格低于 130，溢价率低于 20% 的标的：

```
for k,v in industry\_dict.items():
    mean = v\['premium\_rt'\].mean()
    temp\_v = v\[(v\['premium\_rt'\]<mean) & (v\['premium\_rt'\]<20) & (v\['full\_price'\]<130)\]
    if len(temp\_v)>0:
        print(k,f'均值{mean:0.1f}    ',temp\_v\['bond\_nm'\].tolist())

```

筛选出来的结果：

![](https://article-images.zsxq.com/FknKdEiZNl1yLmPCdimpacSS0ePm)

附录数据与代码

读取数据：

```
import joblib
industry\_dict = joblib.dump('集思录行业数据.zip')
print(industry\_dict)

```

 [https://articles.zsxq.com/id_h3zx89e3eals.html](https://articles.zsxq.com/id_h3zx89e3eals.html)
