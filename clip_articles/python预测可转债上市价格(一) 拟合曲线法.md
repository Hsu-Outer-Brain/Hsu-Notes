# python预测可转债上市价格(一) 拟合曲线法
平时看到大 v 比较热衷于预测上市债的价格. 本文使用 python 对历史数据进行曲线拟合, 然后通过拟合的曲线来预测上市转债的价格.

首先需要转债的上市历史数据, 包括开盘价格, 收盘价格, 转股价值 (可以通过转股价与正股价格来计算, 也可以通过 溢价率来计算)

用 python 从 excel 读取数据

![](https://article-images.zsxq.com/FtJL3CAAF_SAAqQkVoBawucYUnjy)

```
file = 'new\_bond.xlsx'
data = pd.read\_excel(file)

```

抽查一行数据看看样本 :

![](https://article-images.zsxq.com/FjtxTjaYT_igk7RknPD6TTE1WPPI)

多了一行 Unnanamed: 0 没必要的数据, 可以删除掉.

```
data = data.drop('Unnamed: 0',axis=1)

```

计算转股价值

```
data\['innerValue'\] = data\['closePriceBond'\]/(1+data\['bondPremRatio'\]/100)

```

提取数据

```
X = data\['innerValue'\]
Y=data\['closePriceBond'\]

```

有了 X,Y 的数据就可以先绘制一个散点图:

![](https://article-images.zsxq.com/FhhyLC6c-NQHGtGfWNXyZ7qo6UT6)

拟合曲线有很多中方法, 有多项式, 有指数方式.

一般多项式用 3 次方的方程就够用了. 指数式一般可以采用自然指数为底.

这里采用指数方式.

```
Y = a\*e\*\*(b/x)

```

输入 X,Y, 拟合出 a 和 b 的值

```
\# 使用指数拟合
from scipy.optimize import curve\_fit
def func(x,a,b):
      return a\*np.exp(b/x)


popt,pcov=curve\_fit(func,X,Y)

a=popt\[0\] #popt里面是拟合系数，读者可以自己help其用法

b=popt\[1\]

zs\_yvals=func(X,a,b)

```

```
a=203.99004114640422
b=-58.72563272488835

```

得到的 zs_yvals 就是预测值了.

![](https://article-images.zsxq.com/Fn41-CD4bCgTDQcCbOylNCfbF2RR)

绿色线为溢价率为 0 时的直线, 如果红点落在直线上, 说明该转债溢价率平价收盘, 在绿线上方, 则是溢价收盘, 在下方, 则是折价收盘. 实际大部分位于绿色直线上方溢价收盘的, 这有转债这一品种决定的, 毕竟在可知的时间内下有底上涨无限.

中间那根黄色偏白色的线是拟合的曲线, 图中看到很多红点都是偏出黄线, 走势上围绕着黄线. 如果用高次多项式, 也只能过拟合, 不也只能穿过部分点, 这类纯线性的回归注定了大误差, 毕竟很多明显的因子没用上.

**实战**

拉个转债试试效果:

刚上市的银轮转债, 其转股价值为 93.78, 放进去试下:

得到 Y=109.0563598795809

预测结果是 109 元.

在 jisilu 上随便找一个预测贴, 比如盛唐风物 的 [https://www.jisilu.cn/question/428645](https://www.jisilu.cn/question/428645)

![](https://article-images.zsxq.com/Fln8EmZPd3yIK0cFm3wufTL9OGp6)

他预测的才 107.

结果银轮转债的收盘价 122. 和实际预测的都差挺远的.

因为只靠一个转股价值因子来预测, 肯定是不够的, 因为还有评级, 股东配售率, 中签率等因素影响呢.

更为靠谱的方法是使用机器学习或者深度学习的方式来预测, 因为在那些框架和模型下, 会自动调整因子的重要性与权重.

所以本文的标题标了一个 "一", 在后续的文章中, 会采用随机森林 (机器学习) 和 RNN 神经网络 (深度学习) 进行预测. 本文纯粹抛砖引玉, 有兴趣的可以后台交流.

对原来预测的数据做个误差分析:

x = 预测值 - 真实值, 然后取绝对值, 累加后取平均.

```
def abs\_error():
    count = len(Y)
    sum=0
    min\_,min\_index,max\_,max\_index=999999,0,0,0
    index=0
    for real,predict in zip(Y,zs\_yvals):
        v=abs(real-predict)
        if v>max\_:
            max\_=v
            max\_index=index
        if v<min\_:
            min\_=v
            min\_index=index
        index+=1
        sum+=v
    return min\_,min\_index,max\_,max\_index,sum/count

```

得到平均误差达到了 6 元, 也就是拿拟合的数据与实际数据比, 平均的误差是 6 元. 并且顺便显示下最大误差的达到了 36 元, 来自于韦尔转债.

![](https://article-images.zsxq.com/FqtxmRwGNg_gcexmPYHfIhPjEB1l)

这可能因为当时热门板块是芯片类, 并且该整正股的质地也不错. 导致的追的人比较多, 把价格推高了, 当时的转债价值为 144 元. 上市首日收盘价格为 172, 而实际预测价格才 140 不到. 因为趋势上看, 转股价值越高 (高于 150), 其上市的溢价率是越低的. 毕竟其债底保护已经很小了.

本人开启了星球号, 欢迎关注. 里面提供回测数据与相关详细代码, 供读者学习, 同时随时间推移定时更新以前回测结果, 分享更多技术投资类文章.

本文需要的可转债分析数据已上传到网盘:

链接: [https://pan.baidu.com/s/1MH5E_Ii9UBAKeB-nrp--7Q](https://pan.baidu.com/s/1MH5E_Ii9UBAKeB-nrp--7Q)

提取码: shen 
 [https://articles.zsxq.com/id_n4b7nvixjpwk.html](https://articles.zsxq.com/id_n4b7nvixjpwk.html)
