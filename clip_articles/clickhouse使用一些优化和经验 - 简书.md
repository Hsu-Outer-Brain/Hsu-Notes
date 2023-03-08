# clickhouse使用一些优化和经验 - 简书
[clickhouse 使用一些优化和经验 - 简书](https://www.jianshu.com/p/52f331fafc2a) 

 [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/270bbcef-83d7-41be-bc23-7537f2bca73a.webp?raw=true)
](https://www.jianshu.com/u/9db280d09340)

12022.06.01 15:18:51 字数 1,405 阅读 2,115

1，查询强烈要求带上分区键过滤和主键过滤，如 where day = today() and itime = now()。

2，建表的时候，选择合适的分区键和排序键是优化的关键。

3，如果不允许重复主键 (且不要求去重时效性)，建议使用表类型：ReplicatedReplacingMergeTree 建表语句可参考[https://clickhouse.yandex/docs/en/operations/table_engines/replacingmergetree/](https://links.jianshu.com/go?to=https%3A%2F%2Fclickhouse.yandex%2Fdocs%2Fen%2Foperations%2Ftable_engines%2Freplacingmergetree%2F) , 注意只能保证单节点的数据不重复，无法保证集群的。

4，如果要对某一列过滤，且该列非 partition key 和 orderby key, 且该列过滤前后数据量差异较大，建议使用 prewhere clause 过滤。参考：[https://clickhouse.yandex/docs/en/query_language/select/#prewhere-clause](https://links.jianshu.com/go?to=https%3A%2F%2Fclickhouse.yandex%2Fdocs%2Fen%2Fquery_language%2Fselect%2F%23prewhere-clause)。

5，日期和时间使用 Date, DateTime 类型，不要用 String 类型。

6，建表时，强烈建议低基数 (基数小于 10000) 且类型为 String 的列，使用**LowCardinality**特性，例如国家 (country)，操作系统(os) 皆可用 LowCardinality。查询效益提高可以 40~50%，具体参考[https://altinity.com/blog/2019/3/27/low-cardinality](https://links.jianshu.com/go?to=https%3A%2F%2Faltinity.com%2Fblog%2F2019%2F3%2F27%2Flow-cardinality)。

7，为了使复杂查询尽量本地完成，提前减小数据量和网络传输，加快查询速度，创建分布式表时，尽量按照主键 hash 分 shard。例如欲加快 select count(distinct uid) from table_all group by country, os 的查询速度. 创建分布式表 table_all 时，shard key 为 cityHash64(country, os)，hash 函数参考[https://clickhouse.tech/docs/en/sql-reference/functions/hash-functions/](https://links.jianshu.com/go?to=https%3A%2F%2Fclickhouse.tech%2Fdocs%2Fen%2Fsql-reference%2Ffunctions%2Fhash-functions%2F)。

8，计算不同维度组合的指标值时，用 with rollup 或 with cube 替代 union all 子句。

9，建表时，请遵守命名规范：分布式表名 = 本地表名 + 后缀 "\_all"。 select 请直接操作分布式表。

10，官方已经指出 Nullable 类型几乎总是会拖累性能，因为存储 Nullable 列时需要创建一个额外的文件来存储 NULL 的标记，并且 Nullable 列无法被索引。因此除非极特殊情况，应直接使用字段默认值表示空，或者自行指定一个在业务中无意义的值（例如用 - 1 表示没有商品 ID）

11，稀疏索引不同于 mysql 的 B + 树，不存在最左的原则，所以在 ck 查询的时候，where 条件中，基数较大的列（即区分度较高的列）在前，基数较小的列（区分度较低的列）在后。

12，多表 Join 时要满足小表在右的原则，右表关联时被加载到内存中与左表进行比较

13，多维分析, 查询列不宜过多, 过滤条件带上分区筛选 (select dim1, dim2, agg1(xxx), agg2(xxx) from table where xxxx group by dim1, dim2 )

14，禁止 SELECT \*, 不能拉取原始数据!!!! (clickhouse 不是数据仓库, 纯粹是拉原始表数据的查询应该禁止, 如 select a, b, c, f, e, country from xxx)

**分区键和排序键理论上不能修改，在建表建库的时候尽量考虑清楚**。

0，事实表必须分区，分区粒度根据业务特点决定，不宜过粗或过细。我们当前都是按天分区，按小时、周、月分区也比较常见（系统表中的 query_log、trace_log 表默认就是按月分区的）。

1，分区键能过滤大量数据，分区键建议使用 toYYYYMMDD() 按天分区，如果数据量很少，100w 左右，建议使用 toYYYYMM() 按月分区，过多的分区会占用大量的资源，会对集群的稳定性造成很大的影响。

2，分区键必须使用 date 和 datetime 字段，避免 string 类型的分区键

3，每个 sql 必须要用分区键，否则会导致大量的数据被读取，到了集群的内存限制直接拒绝

4，排序键也是一个非常重要的过滤条件，考虑到 ck 是 OLAP 库，排序键默认也是 ck 的主键，loap 库建议分区键要使用基数比较少的字段，比如 country 就比 timestramp 要好。

5，不要使用过长的分区键，主键 。

6，CK 的索引非 MySQL 的 B 树索引，而是类似 Kafka log 风格的稀疏索引，故不用考虑最左原则，但是建议基数较大的列（即区分度较高的列）在前，基数较小的列（区分度较低的列）在后。另外，基数特别大的列（如订单 ID 等）不建议直接用作索引。

分区数过多会导致一些致命的集群问题。**不建议分区数粒度过细，不建议分区数过多**，经验来看，10 亿数据建议 1-10 个分区差不多了，当然需要参考你的硬件资源如何。

1，select 查询性能降低，分区数过多会导致打开大量文件句柄，影响集群。

2，分区数过多会导致集群重启变慢，zk 压力变大，insert 变慢等问题。

[https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/custom-partitioning-key/](https://links.jianshu.com/go?to=https%3A%2F%2Fclickhouse.tech%2Fdocs%2Fen%2Fengines%2Ftable-engines%2Fmergetree-family%2Fcustom-partitioning-key%2F)  

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/565486bf-c20c-42cb-bc21-e9967d2d1a3c.png?raw=true)

image2021-7-13_9-26-37.png

更多精彩内容，就在简书 APP

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/fec68e13-a9a8-4597-aea8-b6947ca27c61.png?raw=true)

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/69ccd2e6-ddf2-4927-a41b-b67a2239ad05.webp?raw=true)
](https://www.jianshu.com/u/9db280d09340)

-   我被篮球砸伤了脚踝，被送到医院后，竟然遇到我老公全家，陪另一个女人产检。 平时对我的冷言冷语的小姑子吴彤彤挽着小三...

    [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/b3be6096-e324-4b5d-9f4f-048bfbb7bcef.jpeg?raw=true)
    茶点故事](https://www.jianshu.com/u/0f438ff0a55f)阅读 17469 评论 0 赞 8
-   我是他体贴入微，乖巧懂事的金丝雀。 上位后，当着满座宾朋，我播放了他曾经囚禁、霸凌我，还逼疯了我妈妈视频。 1 我...

    [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/6b1893cc-7f5b-49fc-a573-64258662042e.jpeg?raw=true)
    茶点故事](https://www.jianshu.com/u/0f438ff0a55f)阅读 5564 评论 0 赞 2
-   一、 面基当天，我把谈了三个月的网恋对象拉黑了。 事情是这样的。 那天我刚上完早课，叼了个冰棒走在回家的路上，就听...
-   被退婚后，我嫁给了当朝新帝。 本只想在后宫低调度日，却不料新帝是个恋爱脑。 放着家世优越的皇后和貌美如花的贵妃不要...

    [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/a7d755a8-2f7e-48f0-aabc-3f1b7745e492.jpeg?raw=true)
    茶点故事](https://www.jianshu.com/u/0f438ff0a55f)阅读 2319 评论 0 赞 29
-   央央一时 我的男朋友，是个满脑子只有研究的物理系教授。 末世爆发，他变成了丧尸，别的丧尸，一个劲的咬人，而他，一个...
-   正文 我爸妈意外去世后，我才知道，我表妹才是他们的亲女儿。 我只是从福利院抱来，替她挡灾的。 我表妹凌楚，接管了他...

    [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/7dbe0aed-b961-4a02-a404-beaafe117961.jpeg?raw=true)
    茶点故事](https://www.jianshu.com/u/0f438ff0a55f)阅读 942 评论 0 赞 0
-   正文 我心血来潮检查老公的手机，却意外看到他的聊天记录，“你还爱我对吗？”“昨晚的事你就忘了吧。” 虽然只有短短两句...

    [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/8afd5686-0fdd-4e7f-8d43-fae0a01f792d.jpeg?raw=true)
    茶点故事](https://www.jianshu.com/u/0f438ff0a55f)阅读 2619 评论 0 赞 0
-   正文 出嫁四月，我的夫君带回他的心头好白月光。将她抬为平妻，从此与我平起平坐。 我强忍心痛转移注意力，最后偶然发现...

    [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/ba397d52-a0b8-4c7a-8922-bdb0b966091f.jpeg?raw=true)
    茶点故事](https://www.jianshu.com/u/0f438ff0a55f)阅读 417 评论 0 赞 0
-   正文 被顾辰扔到床上的时候，我才参加完一个活动，保姆车刚刚离开，顾辰就敲响了我的房门。 我叫林星，是一个比较出名的...

    [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/8f5bd109-7204-4d85-a750-da82392fa277.jpeg?raw=true)
    茶点故事](https://www.jianshu.com/u/0f438ff0a55f)阅读 537 评论 0 赞 3
-   我吃了整整三年的避子药，因为怀过皇嗣的妃嫔都死了，无一例外。三年前帝后大婚，洞房花烛的第二天早上我就毅然决然饮下了...
-   我跟我前夫在商场里面给小孩买衣服，他现在的老婆不停地给他打电话。 前夫没办法，只能撒谎说自己在公司开会。 对方听清...

    [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/27c1e25e-1179-4f0c-871a-bfb3e9438494.jpeg?raw=true)
    茶点故事](https://www.jianshu.com/u/0f438ff0a55f)阅读 5756 评论 0 赞 1
-   好不容易攻略到头，逮住机会和反派同归于尽。 系统告诉我让我攻略没让我死？ 重开一局，攻略对象变成了上一世跟我一起炸...

    [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/4fbe0994-44a1-4780-b7ab-c21391d73e8e.jpeg?raw=true)
    茶点故事](https://www.jianshu.com/u/0f438ff0a55f)阅读 1351 评论 0 赞 1
-   人人都说尚书府的草包嫡子修了几辈子的福气，才能尚了最受宠的昭宁公主。 只可惜公主虽容貌倾城，却性情淡漠，不敬公婆，...

    [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/95d7e672-16bd-4254-ba71-fc07e0e07047.jpeg?raw=true)
    茶点故事](https://www.jianshu.com/u/0f438ff0a55f)阅读 1433 评论 1 赞 5
-   正文 那天我在老公的包里发现了两个避孕套，他说是社区工作人员宣传防范艾滋病派发的。 我当然不信，那包装明显是便利店...

    [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/dd253452-632a-4da3-800d-6ac7b3870d2e.jpeg?raw=true)
    茶点故事](https://www.jianshu.com/u/0f438ff0a55f)阅读 5215 评论 0 赞 12
-   正文 太子周落回不近女色，只钟情于太子少傅江景，而甄爽身为将门虎女也深深迷恋江景。无奈皇命难为，一道圣旨成功地把甄...

    [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/ca960b8a-7e1c-482e-9eff-8f56a99b9bab.jpeg?raw=true)
    茶点故事](https://www.jianshu.com/u/0f438ff0a55f)阅读 1653 评论 0 赞 0
-   作者：猫打滚儿 文案： 八岁那年，我被我爸当成诬陷村长的诱饵，我最崇拜的叔叔是他的同谋，那夜之后，这三个男人都失踪...
-   皇帝的白月光自尽了，我却瞧着他一点也不伤心。 那日我去见贵妃最后一面，却惊觉她同我长得十分相似。 回宫后我的头就开...

    [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/abe8f120-ccd7-47e0-bf06-6a276c49d8a1.jpeg?raw=true)
    茶点故事](https://www.jianshu.com/u/0f438ff0a55f)阅读 1278 评论 0 赞 1
-   谁能想到有朝一日，逼宫这种事会发生在我身边。 被逼走的是我亲妈，始作俑者是我亲小姨。 为了争得我的抚养权，母亲放弃...

    [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/7881c381-89c0-48ab-9c8d-87d680d2cb10.jpeg?raw=true)
    茶点故事](https://www.jianshu.com/u/0f438ff0a55f)阅读 1540 评论 0 赞 2
-   正文 我的男朋友，被我相处了五年的好闺蜜抢走了。 把他让给我吧，我求你了。」 随后她把高领毛衣往下一扯，漏出很明显...

    [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/cc6ceb4a-4093-4e4b-9878-4981fc9782be.jpeg?raw=true)
    茶点故事](https://www.jianshu.com/u/0f438ff0a55f)阅读 5764 评论 0 赞 15
-   我朋友是做直播的，人长得好看，公司也肯花力气捧。 轻轻松松半年，就在平台上攒了 100 多万粉丝。 而就在昨天开播的时...

    [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/73052947-94fd-47b2-9683-4a391d3ff33f.jpeg?raw=true)
    茶点故事](https://www.jianshu.com/u/0f438ff0a55f)阅读 8877 评论 1 赞 16

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

-   官方文档：[https://clickhouse.yandex](https://clickhouse.yandex) ClickHouse 是什么？有什么？能做什么？ 为什...

    [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2011-18-07/1c2ab034-f590-42a3-a54e-3a12f0352e18.webp?raw=true)
    粮忆雨](https://www.jianshu.com/u/224f57c4918e)阅读 14,889 评论 0 赞 14
-   概述 这是 Alexey Milovidov（ClickHouse 的创建者）给出的关于复合主键\[[https://..](https://..).
-   背景 随着互联网的快速发展和互联网 + 物联网的场景不断增加，信息数据量正在呈几何式的爆发。这些海量数据决定着企业的未...
-   ClickHouse 简介 ClickHouse 是一个用于联机分析 (OLAP) 的列式数据库管理系统(DBMS)。 ...
-   建表，排序，索引及注意事项 [https://clickhouse.tech/docs/zh/operations/..](https://clickhouse.tech/docs/zh/operations/..).
