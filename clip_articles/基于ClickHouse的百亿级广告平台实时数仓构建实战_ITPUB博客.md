# 基于ClickHouse的百亿级广告平台实时数仓构建实战_ITPUB博客
[基于 ClickHouse 的百亿级广告平台实时数仓构建实战\_ITPUB 博客](http://blog.itpub.net/70016482/viewspace-2902722/) 

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-52-08/29ec0816-987d-4d82-ae2c-c6009d6201db.png?raw=true)

**编辑 | 韩楠**

约 5100 字 | 10 分钟阅读

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-52-08/caebbc15-c364-4173-bb82-fb14624a4552.png?raw=true)

我们公司是一个每天百亿增量数据的互联网广告公司，作为大数据专家我主要的职责是负责系统的优化和迭代，在系统优化和迭代中，我和我们的团队一直努力寻求一种既能快速完成数据分析又能节省服务器资源的数据解决方案。  

我们的广告平台的数据，包括实时数据和离线数据。实时数据首先会发送到消息系统，然后在消息系统中进行流转批，最终将转换后的数据存储在 S3 上，离线数据则直接调用 S3 的 API 将数据存储在 S3 上。S3 上存储的数据格式有：CSV、TXT、Parquet、ORC 等。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-52-08/1414a79e-c109-4574-bca4-127e8505e4d6.png?raw=true)

图 1

查询的时候通过 Athena、Spark 或者 Hadoop 进行查询分析。最终分析的数据用于 BI、统计报表和告警规则等。 

这种方案的好处是多源数据入库方便，并且查询的时候，在 Athena 中通过标准 SQL 按照不同的需求，可以进行任何维度的数据分析。

但是这种方案有一个很大的缺点就是查询效率慢，并且多源数据的一致性难以保障。一般对 TB 级别数据量的分析耗时在 20 秒以上，因此它更适合于离线数据分析。这也是大数据行业比较典型的离线数仓方案。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-52-08/34f5b744-8f53-4f6f-a097-8aa09ffb5729.png?raw=true)

图 2  

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-52-08/eb1f5dcd-f976-47b0-ab5c-1010ac3e70ce.png?raw=true)

怎么做既能够将多源数据进行融合，又能实现快速实时的分析数据。同时服务器的预算又能控制在合理的范围内，这个问题，可以说一直是我们架构优化的方向。  

**由于离线数据的缺点随着数据量的增大越来越被放大，因此我们迫切需要一个既能快速融合和分析多源数据，同时又经济实惠的实时数仓的方案。**  经过内部讨论，这个方案需要满足这样的一些条件，梳理了七点。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-52-08/3015d49b-7d2f-4ab7-a28d-4cd704a06ad3.png?raw=true)

图 3

上面 7 个关键点， **第 3、4 点是目前实时数仓建立的核心需求，也是难点。** 

> 值得注意的是：
>
> 任何解决方案都没有完美的，我们只能在大量的实践摸索中不断加以完善、丰富。我们的实时数仓基于 ClickHouse 的 projection 在构建主题上探索出了比较好的方案，但是基于数据变更的自动感知，目前仍然不是个比较完美的解决方案，后续的实践中还需要进一步完善。行业里该方案其实也处于探索阶段。

言归正传，刚刚经过前面的需求分析，我们十分明确的一点是 “需要一个实时数仓平台”。但是如何根据我们目前的需求在投入最少的情况下，获得最多的收入呢？为了达到上面的目标，我们对市场上的实时数仓方案进行了分析，最终确定基于 StartRocks 或者 ClickHouse 构建实时数仓。来一起看看具体的实现方式：

1.  将数据源上的实时数据直接写入消费服务。
2.  对于数据源为离线文件的情况，有两种处理方式，一种是将文件转为流式数据写入 Kafka，另外一种情况是直接将文件通过 SQL 导入 ClickHouse 集群。
3.  ClickHouse 接入 Kafka 消息，并将数据写入对应的原始表，基于原始表可以构建物化视图、Project 等实现数据聚合和统计分析。
4.  应用服务基于 ClickHouse 数据对外提供 BI、统计报表、告警规则等服务。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-52-08/c191a7b5-bce3-4a0f-9474-97f6af99b027.png?raw=true)

图 4

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-52-08/2215ef43-4a14-487c-8c0a-5be359fadb89.png?raw=true)

在 StarRocks、ClickHouse 这两种方案中，我们经过性能和功能上的比较，最终选择了 ClickHouse。这里可能有人会想了，那为什么选择 ClickHouse 呢？主要原因有这几点。 

1.  **通过具体的测试发现 ClickHouse 在 50TB 约 140 亿数据上，整体的查询稳定性和性能优于 StarRocks。** 
2.  ClickHouse 支持源数据载入，具体包括 Kafka、S3、HTTP、JDBC 等。
3.  ClickHouse 支持联邦查询，可以轻松将 MySQL 或者 MongoDB 的数据和 ClickHouse 的数据进行关联查询。
4.  **基于 project 支持数据的实时统计。** 
5.  **基于物化视图支持基于原始的统计数据存储。** 
6.  支持数据权限。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-52-08/aecb57e0-3ee3-4c23-8f52-ab996b782065.png?raw=true)

图 5

介绍完了我们选择 ClickHouse 的原因，下面进入到本文的重头戏，如何通过 ClickHouse 实现我们最初在构建实时数仓时提出的要求。

### （一）如何保障各种来源的数据能实时地入仓

一般我们将多源数据分为离线数据、实时数据和其他周边库数据。离线数据主要指文件数据，实时数据主要是 Kafka 消息队列中的数据、周边库数据指的的类似 MySQL、MongoDB 等数据。接下来从实战角度，分别说下如何接入各种数据源。

（1）MySQL 的数据接入，可以通过使用 JDBC 方式实现。

CREATE TABLE user_table(    \`id\` Int32,    \`user_name\` String,    \`height\` Float32,    \`password\` Nullable(String))ENGINE JDBC('jdbc:mysql://localhost:3306/?user=root&password=root', 'test', 'test')

code1

通过前面代码将 MySQL 表接入到 ClickHouse 后，就可以直接在 ClickHouse 中直接查询 MySQL 表中的数据。

SELECT \* FROM user_table

code2

同样也可以向 MySQL 中插入数据。

INSERT INTO user_table(\`id\`, \`user_name\`) VALUES(1,'alex')

code3

（2）下面咱们看下 MongoDB 的数据接入：

CREATE TABLE \[IF NOT EXISTS] testdb.test_collection(id UInt64,    name String,) ENGINE = MongoDB(127.0.0.1:27017, testdb, test_collection, 'your_user_name','your_password');

code4

表建立好后便可以执行查询语句。

SELECT COUNT() FROM test_collection;

code5

（3）再看下 S3 的数据接入：

CREATE TABLE s3_table (name String, value UInt32)     ENGINE=S3(')    SETTINGS input_format_with_names_use_header = 0;

code6

S3 表数据建立好后，我们向其中插入数据。

INSERT INTO s3_table VALUES ('a', 1), ('b', 2), ('c', 3);

code7

同样可以像使用关系型数据库一样查询 S3 数据表。

SELECT \* FROM s3_table LIMIT 2;

code8

（4) Kafka 数据接入：ClickHouse 提供了 Kafka 数据接入引擎，可以方便地实现将 Kafka 的数据接入到 ClickHouse 中，并将其插入到 ClickHouse 表中。

CREATE TABLE kafka_source_test (level String,    type String,   name String,   time DateTime64) ENGINE = Kafka SETTINGS kafka_broker_list = 'localhost:9092',                            kafka_topic_list = 'test_topic',                            kafka_group_name = 'test_group',                            kafka_format = 'JSONEachRow',                            kafka_num_consumers = 4;

code9

这段代码接入了地址为 localhost:9092，topic 为 test_topic 的 Kafka 实时数据。其中消费者分组为 test_group，ClickHouse 会实时将数据消费到 ClickHouse 的表 kafka_source_test 中。当 Kafka 的 Topic 有数据产生时，数据会实时被消费和处理，并插入 kafka_source_test 表中。

  SELECT \* FROM kafka_source_test LIMIT 5

code10

通过前面的介绍，你可以看到 ClickHouse 接入数据源还是比较简单的，其中 Kafka 是构建实时数仓最常用的数据源。

### （二）数据入仓能及时被分析

这里的数据分析分为两种情况，一种情况是有明确需求的报表分析，这种情况下可以对实时数据边入库边分析，具体可以通过建立物化视图的方式实现，也可以通过 ClickHouse 的 projection 实现。另外一种情况是随机性的数据分析需求，像这种需求 ClickHouse 基于索引可以做到 TB 级别数据秒级计算返回。具体的代码实现后面将会介绍。  

### （三）基于实时数仓能方便的根据业务需求建立主题数据

我们构建实时数仓的很大一个初衷是：当客户有基于主题的报表需求时，我们通过一个配置或者一条 SQL 语句，就能实现业务的统计报表。那么具体在 ClickHouse 中如何实现呢？

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-52-08/1269f170-482c-438d-b85e-e3a51a619d83.png?raw=true)

**1. 物化视图**

### 

虽然 ClickHouse 的性能在 TB 级别的数据的查询上，已经能达到秒级返回了，近乎完美。但是在超大规模和特别复杂的计算场景下计算耗时，还是面对一些挑战，这时我们就需要通过一个类似触发器的机制，将原始数据进行实时汇总，这样我们统计分析时直接查询汇总后的表就可以，如此一来，无论对于服务器负载还是业务快速分析的需求都是友好的。那么如何对 ClickHouse 数据进行实时分析汇总呢？答案就是物化视图。  

这里得先看下物化视图的原理。

物化视图的构建过程中包括一个原表，一个物化视图和一个目标表。物化视图定义了数据物化的计算逻辑，当我们向原表插入一条数据时，会触发物化计算，计算完成后将计算结果实时写入目标表。用户查询的时候查询目标表即可完成数据的分析。  

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-52-08/9671a329-b364-4702-959b-5df2f66ee5a6.png?raw=true)

图 6

在实际应用中我们对于报表需求，我们通过物化视图来实现，例如对于 kafka_source_test 中的数据，我们需要基于 type 字段按照 “天” 进行统计。

首先我们需要建立一个物化视图的基础表。

CREATE TABLE statistics_type_day(type String,    day Date,    type_count UInt32)ENGINE = SummingMergeTree()ORDER BY (type, day)

code11

上述建表语句中使用了 SummingMergeTree 引擎，该引擎用于统计数据的存储，它能在合并分区的时候，按照预定义条件对数据进行聚合，将相同分组的多行数据统一到一行，从而显著减少存储空间，加快查询的效率。

接下来我们创建一个物化视图，将 kafka_source_test 表中的数据实时进行统计分析并写入 statistics_type_day 表中。

CREATE MATERIALIZED VIEW if not exists statistics_type_day_mv TO statistics_type_day AS SELECT type, day, count(1) type_countfrom kafka_source_testgroup by type, day

code12

这样当 kafka_source_test 中有数据的时候，便能实时统计并将统计结果物化下来。查询的时候直接查询物化表 statistics_type_day 就行。

select \* from statistics_type_day where day='2022-03-20'

code13

**2.projection：数据投影**  

ClickHouse 的 projection 本质上实现的是数据聚合，小伙伴可能要问了，既然 ClickHouse 已经有物化视图了，为何还需要 projection 呢？

试想这样一个场景，我们有一个 150 个字段的表，基于该表我们针对不同业务构建了 20 个物化视图。那我们业务在分析的时候如果查询基础数据就到基础表查询，如果查询的是统计数据则需要在对应的物化视图的统计表中查询。

但是在实际的使用过程中，常常因为构建视图和使用视图的人不在一个团队，一种情况是使用者不知道有哪些视图可以使用，所以在写 SQL 的是时候就直接查基表了，另外一种情况是业务需要针对基础表的 SQL 和统计表的 SQL 维护多个业务 SQL，在查询上不统一。  

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-52-08/00d76738-62fc-433f-be2f-2c5518560a5d.png?raw=true)

 图 7

那么如何构建一个表，在该表上既能完成基础数据的查询，也能进行统计数据的查询，如果能实现的话，之前的 1+20 个表 SQL 的维护就变成了 1 张表 SQL 的维护。使用起来将大大简化应用层开发，应用层也不用关注哪些是统计数据，哪些是基础数据。

ClickHouse 的 projection 给了这个问题一个很好的解决方案。

projection 在数据分析功能上类似于物化视图，我们可以在 projection 中预定义表达式，当数据写入的时候，会将原始数据和针对 projection 中表达式计算后的聚合数据一起写入存储。  

在查询的时候会经过一个智能路由的过程，智能路由通过分析 SQL 语句，如果发现 SQL 语句查询的数据都在聚合数据中，则直接查询聚合数据并将结果返回给用户，这块便大大减少了服务器资源的开销，只有到查询的数据在各个 projection 都不存在的时候才会去查基表进行统计。也可以理解为 ClickHouse 基于 projection 自动完成了查询优化。（具体参见下图）  

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-52-08/365ffc3c-9710-436c-a667-e07b76fd7833.png?raw=true)

图 8

了解了 projection 原理后，接下来我们看下如何使用 projection，首先需要定义 projection。

CREATE TABLE test_projection_table(    level String,   type String,   name String,   city String,   time DateTime64,    PROJECTION projection_1    (SELECT             name,             count(1)         GROUP BY  level    ) ,     PROJECTION projection_2    (SELECT             name,             count(1)         GROUP BY type     ) ) ENGINE = MergeTree() ORDER BY (name , level, type)

code14

上述代码在定义表的时候定义了 projection_1 和 projection_2，其中 projection_1 在 level 维度对数据进行汇总，projection_2 在 type 维度对数据进行汇总。在创建好 projection 后还可以对 projection 进行修改。

ALTER TABLE test_projection_table     ADD PROJECTION projection_3    (SELECT             count(1),             name        GROUP BY type, level     )

code15

那么查询的时候，如何才能命中 projection 呢？得满足这样几个条件：

1.  select 表达式必须为 projection 定义中 select 表达式的子集。
2.  group by 列必须为 projection 定义中 group by 列的子集。
3.  where 条件 必须为 projeciton 定义中的 group by 列 的子集。

例如这里 SQL 查询的时候会走表达式：

select name,count() from test_projection_table where type='A' group by type

code16

而 SQL 不会走 projection，因为 city 不在 projection 的定义中。

select name,city,count() from test_projection_table where type='A' group by city

code17

具体可以通过 explain 查看执行计划，如果出现 ReadFromStorage (MergeTree(with projection)) ，表示命中 projection。

### （四）实时数仓需要提供元数据变更的自动感知能力。

对于元数据的变更 ClickHouse 无法做到自动感知，我们构建了一个元数据管理平台，实时监 ClickHouse Insert 语句，当发现有新的 Schema 变更后以告警的形式通知研发人员，有研发人员修改表结构以兼容元数据的变更。

### （五）实时数仓难免需要要外部数据交互，因此实时数仓需要具备联邦查询的能力。

对于联邦查询，我们将外部库表数据接入 ClickHouse 并和 ClickHouse 进行联邦查询。

### （六）对外提供一个统一的数据分析服务，用户不用关心数仓底层数据复杂的关系关系。

在查询上我们利用 projection 特性将查询统一到一个表上，除非有特别特殊的需求会走物化视图。

### （七）需要确保实时数仓中存储的数据量在合理的访问内，以确保不会产生过多的服务器资源账单。

在控制数据的存储量上我们通过 ClickHouse 的冷热分离思想，将热数据存储在 ClickHouse 上，将冷数据存储在 S3 上，以控制服务器资源。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-52-08/d5d8e35f-bf11-477a-8d06-776635cd60e9.png?raw=true)

图 9

**结束语**

本文我介绍了如何基于 ClickHouse 的百亿级广告平台实时数仓构建实战。在数仓的构建前期我们首先基于数据要达成的目标进行了梳理。  

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-52-08/597c2641-b952-49a4-9ee7-00fdd625392f.png?raw=true)

图 10

我们在 StarRocks、ClickHouse 选型上，基于性能原因我们选择了 ClickHouse。在具体的实现细节上我们没有像传统数仓那样构建完成的 ODS、DWD、DWS、ADS 层，而是借助 ClickHouse 的物化视图和 projection 完成我们构建实时数仓的目标。

其实在该架构中，可以将接入 Kafka 原表和外部表的数据理解为 ODS（贴源数据层），将经过数据清洗转换后的基础表理解为 DWD（数据明细层），将基于基础表衍生的物化视图和 projection 等统计数据理解为 DWS（轻度汇总层），基于视图我们还可以接着构建视图，表和 projection。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-52-08/d66f991d-d9c6-496f-bbc0-3f7674cdae00.png?raw=true)

图 11

**延伸思考**

前面给大家介绍了我们的实时数仓的方案，那么实时数仓是如何演化来的呢？下面多说几句，跟大家从宏观上说下大数据的行业发展，以便于能够从一个更高视角理解实时数仓。只有大家了解了大数据的发展历程，才能更好地理解在当下企业建立实时数仓的必要性，以及我们未来实时数仓未来的发展方向。  

* * *

快速方便地构建基于主题的统计数据，一直是大数据行业的痛点，最初解决该问题的方案是 Hadoop 和 Spark 的离线计算，但随着业务的发展，这两种方案在数据实时性上的短板越来越明显。行业急需一个实时数仓方案，但是在实时数仓中如果基于 Flink 进行实时计算的话，对于业务的频繁变更带来的开发成本又变得不可控。因此大家又把目光投向了 MPP 架构，这种架构基本上达到了 1 个 SQL 就能满足一个业务报表的需求，方便快速，数据又是实时计算的，基本满足了我们的要求。但是其在分布式事务和元数据的自动感知上，还有待完善。

从上面的趋势中也可以看到大数据发展的阶段，第一阶段大家基于大型 MySQL 的分库分表实现大规模数据的存储和分析，但是随着数据量的增大，这个方案玩不转了。于是出现了 Hadoop 和 Spark 来解决海量数据计算问题。等海量数据计算的需求满足后大家又对数据的实时性要求更高了，业务总想看着最新的数据，于是出现了实时数仓。

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-52-08/c8bc6547-9d67-4173-98dc-49dc4493e4a1.png?raw=true)
 图 12

> 但是问题到这里并没有完，后续必然在实时数仓的分布式数据一致性，事务的支持，元数据的动态管理和数据治理上，有更多的需求出现。这个也是大数据行业下个阶段要解决的重点问题。

因此大数据不像关系型数据库那样有稳定的解决方案，而是根据行业的变化和用户的需求在不断的迭代和更新。其最终目标是既能像关系数据库那样满足 ACID 的需求，又具有大规模数据实时计算的能力和灵活的数据分析能力。

最后要说的是 ClickHouse 之所以性能优异，是因为它采用了向量化计算技术。将多次 CPU 计算优化为一次 CPU 计算，从而大大提升了 CPU 的效率。具体实现技术手段为采用 SIMD (Single Instruction Multiple Data) 技术实现单条指令操作多条数据，在 CPU 寄存器层面实现数据的并行操作。 

向量化计算可以说是 MPP 架构的 “银弹”，目前 ClickHouse 和 StarRocks 都采用了这个技术方案来达到海量数据快速分析的，后续肯定会有更多的数据库方案跟进该技术。限于篇幅问题在这里不能为大家详细介绍向量化计算的魅力，后续如果大家感兴趣我再单独拿出来讲一下。

最后，欢迎你在评论区留言与交流，也可以把这一篇分享给你的朋友，我们下期再会。

THE END 

转载请联系 ITPUB 官方公众号获得授权

—————————————————————————————————

欢迎各领域技术人员投稿  

投稿邮箱 |      hannan@it168.com

来自 “ITPUB 博客” ，链接：[http://blog.itpub.net/70016482/viewspace-2902722/，如需转载，请注明出处，否则将追究法律责任。](http://blog.itpub.net/70016482/viewspace-2902722/，如需转载，请注明出处，否则将追究法律责任。)
