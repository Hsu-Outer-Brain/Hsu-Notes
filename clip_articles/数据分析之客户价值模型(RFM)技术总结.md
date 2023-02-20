# 数据分析之客户价值模型(RFM)技术总结
[数据分析之客户价值模型 (RFM) 技术总结](https://baijiahao.baidu.com/s?id=1724552575497144820&wfr=spider&for=pc) 

 管理学中有一个重要概念那就是客户关系管理 (CRM)，它核心目的就是为了提高企业的核心竞争力，通过提高企业与客户间的交互，优化客户管理方式，从而实现吸引新客户、保留老客户以及将已有客户转化为忠实客户的运营机制。

而这其中最为经典的实现模型那就是 RFM 模型，它主要通过对每个客户的近期消费时间，购买频率和购买金额来对不同的客户进行价值状态划分。

从而使得我们可以有针对性的对不同用户进行个性化运营和营销。

该维度指的是最近一次消费时间间隔 (R), 也就是上一次消费的时间间隔，该值越小客户价值越高，这是因为消费间隔越近的客户越有可能产生二次消费。

消费频次 (F) 体现了客户的购买频率，那么购买频次越高，越能体现用户的消费活跃程度，因此，客户价值也就越高。

消费金额 (M) 这个从字面意思即可知道，用户的消费金额越高，用户的消费能力越强，那么自然用户的价值也就越高。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/9286a775-2bc1-44e7-986b-5a8441f9fd84.webp?raw=true)

为了解决现存方法的缺陷，作者首次提出了将 MCL、SSL 和 Excel 是实现 RFM 模型的一个重要且十分直接的工具，只需要灵活使用 Excel 自带的函数就可以实现数据的汇总计算，得到 RFM 模型的三个指标值，从而将用户的价值类型提取出来，让我们有针对性的进行业务推广策略。

接下来我们给大家演示一个用 Excel 实现的 RFM 模型：

根据现有订单数据，构建店铺用户价值模型，从而为后续的精细化运营不同的客户群体打下基础

数据量大概有 3989 条，可以在 excel 内处理，也可以使用 python 对大批量的数据进行处理。

【买家会员名】、【总金额】、【订单付款时间】、并设置对应的数据类型，我们要在同页面进行 RFM 值的计算。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/14033a40-20fa-4270-a13f-22d4603c7dfe.png?raw=true)

**3.2 计算 Recency, Monetary,Frequency**

**Recency：**  通过对【买家会员名】、【总金额】、【订单付款时间】三列数据做透视表，对订单付款时间求最大值，即最近消费时间，然后与观测时间进行求差运算，可得 R 值

**Monetary：** 对总金额下的客户不同消费进行平均值运算，即可获得该客户的 M 值

**Frequency：** 对订单付款时间进行计数运算，就是该客户的消费频次 F 值

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/6ebe14d3-0cca-4dfd-b5ab-ec385a83e8cf.png?raw=true)

计算完客户的 R、F、M 值后，接下来就可以实现客户 RFM 模型的评估了。

此时可以先计算出来 R、F、M 三个值的平均值，然后对客户的每个维度与该维度的平均值进行比较，如果超出平均值就是高，否则就是低。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/7fcbb321-3a91-49f3-832e-a03904f38bf7.png?raw=true)

然后将三列字段通过’&’连接符链接起来，生成 RFM 辅助列。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/fc781de3-42b4-4a93-8018-e41478ea3a9f.png?raw=true)

然后通过我们预先准备好的价值模型参考表，生成用户价值模型。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/08f8fa2f-337e-4153-b519-d248304f7286.webp?raw=true)

最后通过 excel 的 vlookup 函数提取客户类型字段到计算表中，就实现了我们的最终结果。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/555fbf3a-12a1-4cdb-92d3-305bca2ee113.png?raw=true)

通过用户的 R、F、M 值与对应值的极差 (最大值与最小值的差)，来确定 R-Score, F-Score,M-Score。

因此首先计算 R、F、M 的最大值、最小值、极差三等分距

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/f6796d88-5b61-43b7-af33-d0ff889ac3fd.jpeg?raw=true)

**最大值：** 通过 “=max(B5:B1204)” 计算,（计算 F 时 B 换成 C，M 时 B 换成 D 即可）

**最小值：** 通过 “=min(B5:B1204)” 计算（计算 F 时 B 换成 C，M 时 B 换成 D 即可）

**极差：** 通过 “=(F1-F2)/3” 计算（计算 F 时 F 换成 G，M 时 F 换成 H 即可）

“=IF(ROUNDUP((B5-$F$2)/$F$3,0)=0,1,ROUNDUP((B5-$F$2)/$F$3,0)) ”

“=IF(ROUNDUP((C5-$G$2)/$G$3,0)=0,1,ROUNDUP((C5-$G$2)/$G$3,0))”

“=IF(ROUNDUP((D5-$H$2)/$H$3,0)=0,1,ROUNDUP((D5-$H$2)/$H$3,0))”

RFM-Score 计算采用将 R、F、M 以百分位、十分位、个位组成三位数的方式实现，共有 3\*3\*3=27 种组合方式。

**H5 单元格的公式：“=E5\*100+F5\*10+G5”**

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/38a30e4f-bec3-403e-9d7a-6e5deffb8f6a.png?raw=true)

然后对数据表区域 A4 到 H3996 进行数据透视：汇总不同的 RFM-Score 对应的客户群体。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/d3b2ffde-744f-4076-8a5b-e4b34465e1c2.webp?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/65f40c0e-590d-4243-a67a-165eb0085e44.png?raw=true)

通过 Python 处理数据时，我们首先需要关注我们提取进来的数据是否需要预处理，比如数据的类型是否符合预期，字段名是否需要调整，缺失值是否需要填充，重复值是否需要去除等等，因此第一步我们首先需要对数据进行初步的熟悉了解：

常用于初步了解数据的方法有很多比如：shape(了解数据的大小，几行几列)，head(显示其中的前几条数据)，tail(显示数据源最后几条数据)，sample(随机提取几条数据),info(显示数据源的各字段数据类型)，describe(对数据源进行数学描述)。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/50214e51-2385-4019-a472-7bc232659ceb.webp?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/95c37c18-853c-4a25-86ca-161a3875eca5.webp?raw=true)

通过上图我们发现交易记录里面会有一些无效订单，那么我们首先就要排除这类订单，那么就可以通过 pandas 的布尔索引来进行数据的筛选：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/5782dd72-6a37-4d9e-8819-0e6c36a3f71a.png?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/cafd2de0-ad5a-4cc8-bbb0-5cf05fc29ee4.png?raw=true)

鉴于我们仅需要买家付款时间，购买日期，实付金额这三个字段，我们仅需要对他们进行数据处理，因此可以排除其他字段。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/d4972607-e981-4e19-bebc-3a22ca1b1f6a.png?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/143895f0-ee89-4372-886a-16272b022db1.png?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/e0ab50a8-9415-4bf8-b838-045c5d7ae79c.png?raw=true)

添加天数字段，将付款时间与观察日期进行日期计算得到 R 值

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/a1f1a89d-d858-4225-b9d3-3d78765e4fdf.webp?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/9a84b2da-67eb-46f8-936b-3b0e564b888e.png?raw=true)

通过聚合函数，对买家昵称进行计数运算获得消费频次 F 值，计算天数字段的最小值获得客户的 R 值，通过实付金额的求和运算获得客户的 M 值。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/645e9a95-b70e-4370-8fed-ccc5dbeb8e66.webp?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/f2d8571b-e9af-4c59-a02c-8552b1ce7aa0.png?raw=true)

通过上述计算，我们可以根据不同的分数段来对客户 R、F、M 值进行打分，就本案例来讲：

R 值：我们得出的最小值是 660，以 30 天作为间隔，660-690 天，打 5 分；690-720，打 4 分；720-750 打 3 分；750-780 打 2 分；>780，打 1 分。

F 值：我们得出的最小值是 1 次，以 1 次作为时间间隔，0-2，打 1 分；2-3，打 2 分；3-4，打 3 分；4-5，打 4 分；>5，打 5 分。

M 指：我们得出的最小值是 0.005 元，我们以 500 元作为时间间隔，0-50，打 1 分；50-100，打 2 分；100-150，打 3 分；150-200，打 4 分；>200，打 5 分。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/8c3b005f-ac01-4429-8453-5aae5e62d393.webp?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/4ed25337-fd85-49f7-8ba5-e48b5dfc248f.png?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/8575f6d2-e111-462f-a017-15442a4c6b65.png?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/2c3eb6a0-750a-4a61-b4be-ed26dbcc883a.png?raw=true)

第二步：验证用户各项指标是否超出平均值，是则计一分，否则不计分。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/58273f42-6fa7-4b17-8827-3ead2d2dd10e.png?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/54dbb161-37d8-4db4-870a-436b752e6dc6.webp?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/92f986b8-7d31-43b2-9d83-97031dbc5d0c.png?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/61f06614-f134-48c9-993a-b62e872c89bd.webp?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/a948d029-36f8-4132-bcb3-8ceb77c664ce.png?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-20%2009-54-23/cf631ff0-458c-4f1d-8d0b-70cafbc573e7.png?raw=true)

**通过以上数据分析工具的分析，我们可以发现在实现 RFM 模型的方法中，Python 具有更为强大的可用性和灵活性，且拥有完备的数据分析手段，从数据预处理、分析到最后的数据呈现。** 

而 Excel 在实际工作中应用场景也是非常多见的，通过本案例可以很好的实践 Excel 相关函数，希望本文对你的数据分析之旅有所帮助。
