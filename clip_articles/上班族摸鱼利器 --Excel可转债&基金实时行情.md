# 上班族摸鱼利器 --Excel可转债&基金实时行情
Excel 可转债实时行情小工具

闲来没事，偶然翻了下集思录，发现有集友共享了一个不错的 excel 小工具，可以实时查看股票基金行情。这很适合上班摸鱼的韭菜朋友，毕竟 excel 嘛，不至于有公司禁止安装和打开吧。

![](https://article-images.zsxq.com/Fs41F_C3ZzBTn4x2aGEtitTS2qSY)

这里首先要感谢原作者 cjhren 的分享精神。不过由于对转债行情缺乏了一些重要字段，还有默认提供的转债代码太少，故笔者在其基础上做了一定的修改与调整（临时打开百度：如何修改 excel 宏）

![](https://article-images.zsxq.com/FjucB3THyMCa2rqWtoXT42LzrMTg)

按图索骥地加了字段与以及补上所有的转债代码，可以实时更新整个转债市场的行情。

PS：行情数据用的雪球

同样把基金部分的基金 6 位代码全部给补上，这样可以更新整个市场的行情。

### 转债部分

![](https://article-images.zsxq.com/FiIRP3gtatZ-laBbbDkf74X2ecdY)

新增了可转债的溢价率，双低值，日内振幅，剩余年限，与最高价的距离等字段，其实还有很多字段，嫌太拥挤就没加进来了。 与最高价的距离，指当前的价格与当天最高价的距离，比如截止当前，当天的最高价是 130 元，而现在价格回落到了 125 元，假如昨天的收盘价为 124 元，那么与最高价的距离为 - 1\*（125-130）/124 = 0.04 = 4% ， 这个字段可以观察当前价格是冲高回落了，还是一直上升创日内新高。

### 基金部分

![](https://article-images.zsxq.com/FtviNCo12K-w9qVNvzWtUQWVo_GN)

点击 excel 里的更新按钮，可获取可转债与基金的实时行情。注意，此 excel 需要在电脑端使用，可能需要解除 excel 宏限制才能点击那个更新按钮。

不确定手机版的 excel 能否启用宏，即使能用，屏幕太小，excel 字段太多也会影响使用体验。 
 [https://articles.zsxq.com/id_f49v76kbcqgw.html](https://articles.zsxq.com/id_f49v76kbcqgw.html)
