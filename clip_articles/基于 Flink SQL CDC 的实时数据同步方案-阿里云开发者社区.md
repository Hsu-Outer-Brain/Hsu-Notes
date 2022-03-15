# 基于 Flink SQL CDC 的实时数据同步方案-阿里云开发者社区
整理：陈政羽（Flink 社区志愿者）

Flink 1.11 引入了 Flink SQL CDC，CDC 能给我们数据和业务间能带来什么变化？本文由 Apache Flink PMC，阿里巴巴技术专家伍翀 (云邪）分享，内容将从传统的数据同步方案，基于 Flink CDC 同步的解决方案以及更多的应用场景和 CDC 未来开发规划等方面进行介绍和演示。

1、传统数据同步方案  
2、基于 Flink SQL CDC 的数据同步方案（Demo）  
3、Flink SQL CDC 的更多应用场景  
4、Flink SQL CDC 的未来规划  
直播回顾：  
[https://www.bilibili.com/video/BV1zt4y1D7kt/](https://www.bilibili.com/video/BV1zt4y1D7kt/)

## 传统的数据同步方案与 Flink SQL CDC 解决方案

业务系统经常会遇到需要更新数据到多个存储的需求。例如：一个订单系统刚刚开始只需要写入数据库即可完成业务使用。某天 BI 团队期望对数据库做全文索引，于是我们同时要写多一份数据到 ES 中，改造后一段时间，又有需求需要写入到 Redis 缓存中。

![](https://ucc.alicdn.com/pic/developer-ecology/af3289bba8e84e8487339cdd4b56fd86.png)

很明显这种模式是不可持续发展的，这种双写到各个数据存储系统中可能导致不可维护和扩展，数据一致性问题等，需要引入分布式事务，成本和复杂度也随之增加。我们可以通过 CDC（Change Data Capture）工具进行解除耦合，同步到下游需要同步的存储系统。通过这种方式提高系统的稳健性，也方便后续的维护。

![](https://ucc.alicdn.com/pic/developer-ecology/10391f919ed74c259e398e0746f403df.png)

### Flink SQL CDC 数据同步与原理解析

CDC 全称是 Change Data Capture ，它是一个比较广义的概念，只要能捕获变更的数据，我们都可以称为 CDC 。业界主要有基于查询的 CDC 和基于日志的 CDC ，可以从下面表格对比他们功能和差异点。

![](https://ucc.alicdn.com/pic/developer-ecology/7806a56be96b4731892494775f393a39.jpg)

经过以上对比，我们可以发现基于日志 CDC 有以下这几种优势：

· 能够捕获所有数据的变化，捕获完整的变更记录。在异地容灾，数据备份等场景中得到广泛应用，如果是基于查询的 CDC 有可能导致两次查询的中间一部分数据丢失  
· 每次 DML 操作均有记录无需像查询 CDC 这样发起全表扫描进行过滤，拥有更高的效率和性能，具有低延迟，不增加数据库负载的优势  
· 无需入侵业务，业务解耦，无需更改业务模型  
· 捕获删除事件和捕获旧记录的状态，在查询 CDC 中，周期的查询无法感知中间数据是否删除

![](https://ucc.alicdn.com/pic/developer-ecology/1f5bd7fa81cd4a9ca6e21b13d1739a5a.png)

### 基于日志的 CDC 方案介绍

从 ETL 的角度进行分析，一般采集的都是业务库数据，这里使用 MySQL 作为需要采集的数据库，通过 Debezium 把 MySQL Binlog 进行采集后发送至 Kafka 消息队列，然后对接一些实时计算引擎或者 APP 进行消费后把数据传输入 OLAP 系统或者其他存储介质。

Flink 希望打通更多数据源，发挥完整的计算能力。我们生产中主要来源于业务日志和数据库日志，Flink 在业务日志的支持上已经非常完善，但是在数据库日志支持方面在 Flink 1.11 前还属于一片空白，这就是为什么要集成 CDC 的原因之一。

Flink SQL 内部支持了完整的 changelog 机制，所以 Flink 对接 CDC 数据只需要把 CDC 数据转换成 Flink 认识的数据，所以在 Flink 1.11 里面重构了 TableSource 接口，以便更好支持和集成 CDC。

![](https://ucc.alicdn.com/pic/developer-ecology/5d5c18c7805c4a9c89796d6591171c04.png)

![](https://ucc.alicdn.com/pic/developer-ecology/c3c6a01ee8604339907bacf133fab4e3.png)

重构后的 TableSource 输出的都是 RowData 数据结构，代表了一行的数据。在 RowData 上面会有一个元数据的信息，我们称为 RowKind 。RowKind 里面包括了插入、更新前、更新后、删除，这样和数据库里面的 binlog 概念十分类似。通过 Debezium 采集的 JSON 格式，包含了旧数据和新数据行以及原数据信息，op 的 u 表示是 update 更新操作标识符，ts_ms 表示同步的时间戳。因此，对接 Debezium JSON 的数据，其实就是将这种原始的 JSON 数据转换成 Flink 认识的 RowData。

### 选择 Flink 作为 ETL 工具

当选择 Flink 作为 ETL 工具时，在数据同步场景，如下图同步结构：

![](https://ucc.alicdn.com/pic/developer-ecology/768d5052ec0c4ced8d0e79f71843c869.png)

通过 Debezium 订阅业务库 MySQL 的 Binlog 传输至 Kafka ，Flink 通过创建 Kafka 表指定 format 格式为 debezium-json ，然后通过 Flink 进行计算后或者直接插入到其他外部数据存储系统，例如图中的 Elasticsearch 和 PostgreSQL。

![](https://ucc.alicdn.com/pic/developer-ecology/ea195acd250d491ca9e55e67be428006.png)

但是这个架构有个缺点，我们可以看到采集端组件过多导致维护繁杂，这时候就会想是否可以用 Flink SQL 直接对接 MySQL 的 binlog 数据呢，有没可以替代的方案呢？

答案是有的！经过改进后结构如下图：

![](https://ucc.alicdn.com/pic/developer-ecology/6231a71c5b78493783bc0f3c024d9e65.png)

社区开发了 flink-cdc-connectors 组件，这是一个可以直接从 MySQL、PostgreSQL 等数据库直接读取全量数据和增量变更数据的 source 组件。目前也已开源，开源地址：

> [https://github.com/ververica/flink-cdc-connectors](https://github.com/ververica/flink-cdc-connectors)

flink-cdc-connectors 可以用来替换 Debezium+Kafka 的数据采集模块，从而实现 Flink SQL 采集 + 计算 + 传输（ETL）一体化，这样做的优点有以下：

· 开箱即用，简单易上手  
· 减少维护的组件，简化实时链路，减轻部署成本  
· 减小端到端延迟  
· Flink 自身支持 Exactly Once 的读取和计算  
· 数据不落地，减少存储成本  
· 支持全量和增量流式读取  
· binlog 采集位点可回溯\*

## 基于 Flink SQL CDC 的数据同步方案实践

下面给大家带来 3 个关于 Flink SQL + CDC 在实际场景中使用较多的案例。在完成实验时候，你需要 Docker、MySQL、Elasticsearch 等组件，具体请参考每个案例参考文档。

### 案例 1 : Flink SQL CDC + JDBC Connector

这个案例通过订阅我们订单表（事实表）数据，通过 Debezium 将 MySQL Binlog 发送至 Kafka，通过维表 Join 和 ETL 操作把结果输出至下游的 PG 数据库。具体可以参考 Flink 公众号文章：《Flink JDBC Connector：Flink 与数据库集成最佳实践》案例进行实践操作。

[https://www.bilibili.com/video/BV1bp4y1q78d](https://www.bilibili.com/video/BV1bp4y1q78d)

![](https://ucc.alicdn.com/pic/developer-ecology/9fbb325b6cfc46d0b6ea3414fb0a473f.png)

### 案例 2 : CDC Streaming ETL

模拟电商公司的订单表和物流表，需要对订单数据进行统计分析，对于不同的信息需要进行关联后续形成订单的大宽表后，交给下游的业务方使用 ES 做数据分析，这个案例演示了如何只依赖 Flink 不依赖其他组件，借助 Flink 强大的计算能力实时把 Binlog 的数据流关联一次并同步至 ES 。

![](https://ucc.alicdn.com/pic/developer-ecology/fe46d5094ddc4f778aa81380ba4a4866.png)

例如如下的这段 Flink SQL 代码就能完成实时同步 MySQL 中 orders 表的全量 + 增量数据的目的。

````null
CREATE TABLE orders (
  order_id INT,
  order_date TIMESTAMP(0),
  customer_name STRING,
  price DECIMAL(10, 5),
  product_id INT,
  order_status BOOLEAN
) WITH (
  'connector' = 'mysql-cdc',
  'hostname' = 'localhost',
  'port' = '3306',
  'username' = 'root',
  'password' = '123456',
  'database-name' = 'mydb',
  'table-name' = 'orders'
);

SELECT * FROM orders```

为了让读者更好地上手和理解，我们还提供了 docker-compose 的测试环境，更详细的案例教程请参考下文的视频链接和文档链接。

> 视频链接：  
> [https://www.bilibili.com/video/BV1zt4y1D7kt](https://www.bilibili.com/video/BV1zt4y1D7kt)  
> 文档教程：  
> [https://github.com/ververica/flink-cdc-connectors/wiki/](https://github.com/ververica/flink-cdc-connectors/wiki/)中文教程

### 案例 3 : Streaming Changes to Kafka

下面案例就是对 GMV 进行天级别的全站统计。包含插入/更新/删除，只有付款的订单才能计算进入 GMV ，观察 GMV 值的变化。

![](https://ucc.alicdn.com/pic/developer-ecology/708256232bc64896abc327a7cf0a01ae.png)

> 视频链接：  
> [https://www.bilibili.com/video/BV1zt4y1D7kt](https://www.bilibili.com/video/BV1zt4y1D7kt)  
> 文档教程：  
> [https://github.com/ververica/flink-cdc-connectors/wiki/](https://github.com/ververica/flink-cdc-connectors/wiki/)中文教程

Flink SQL CDC 的更多应用场景
---------------------

Flink SQL CDC 不仅可以灵活地应用于实时数据同步场景中，还可以打通更多的场景提供给用户选择。

### Flink 在数据同步场景中的灵活定位

· 如果你已经有 Debezium/Canal + Kafka 的采集层 (E)，可以使用 Flink 作为计算层 (T) 和传输层 (L)  
· 也可以用 Flink 替代 Debezium/Canal ，由 Flink 直接同步变更数据到 Kafka，Flink 统一 ETL 流程  
· 如果不需要 Kafka 数据缓存，可以由 Flink 直接同步变更数据到目的地，Flink 统一 ETL 流程

### Flink SQL CDC : 打通更多场景

· 实时数据同步，数据备份，数据迁移，数仓构建  
优势：丰富的上下游（E & L），强大的计算（T），易用的 API（SQL），流式计算低延迟  
· 数据库之上的实时物化视图、流式数据分析  
· 索引构建和实时维护  
· 业务 cache 刷新  
· 审计跟踪  
· 微服务的解耦，读写分离  
· 基于 CDC 的维表关联

下面介绍一下为何用 CDC 的维表关联会比基于查询的维表查询快。

**■ 基于查询的维表关联**

![](https://ucc.alicdn.com/pic/developer-ecology/48d3e43e60ae4c2cb6e810d26d24ac44.png)

目前维表查询的方式主要是通过 Join 的方式，数据从消息队列进来后通过向数据库发起 IO 的请求，由数据库把结果返回后合并再输出到下游，但是这个过程无可避免的产生了 IO 和网络通信的消耗，导致吞吐量无法进一步提升，就算使用一些缓存机制，但是因为缓存更新不及时可能会导致精确性也没那么高。

**■ 基于 CDC 的维表关联**

![](https://ucc.alicdn.com/pic/developer-ecology/095230f5273d4dd2b92468f64eced044.png)

我们可以通过 CDC 把维表的数据导入到维表 Join 的状态里面，在这个 State 里面因为它是一个分布式的 State ，里面保存了 Database 里面实时的数据库维表镜像，当消息队列数据过来时候无需再次查询远程的数据库了，直接查询本地磁盘的 State ，避免了 IO 操作，实现了低延迟、高吞吐，更精准。

Tips：目前此功能在 1.12 版本的规划中，具体进度请关注 FLIP-132 。

未来规划
----

· FLIP-132 ：Temporal Table DDL（基于 CDC 的维表关联）  
· Upsert 数据输出到 Kafka  
· 更多的 CDC formats 支持（debezium-avro, OGG, Maxwell）  
· 批模式支持处理 CDC 数据  
· flink-cdc-connectors 支持更多数据库

总结
--

本文通过对比传统的数据同步方案与 Flink SQL CDC 方案分享了 Flink CDC 的优势，与此同时介绍了 CDC 分为日志型和查询型各自的实现原理。后续案例也演示了关于 Debezium 订阅 MySQL Binlog 的场景介绍，以及如何通过 flink-cdc-connectors 实现技术整合替代订阅组件。除此之外，还详细讲解了 Flink CDC 在数据同步、物化视图、多机房备份等的场景，并重点讲解了社区未来规划的基于 CDC 维表关联对比传统维表关联的优势以及 CDC 组件工作。

希望通过这次分享，大家对 Flink SQL CDC 能有全新的认识和了解，在未来实际生产开发中，期望 Flink CDC 能带来更多开发的便捷和更丰富的使用场景。

Q & A
-----

**1、GROUP BY 结果如何写到 Kafka ？**

因为 group by 的结果是一个更新的结果，目前无法写入 append only 的消息队列中里面去。更新的结果写入 Kafka 中将在 1.12 版本中原生地支持。在 1.11 版本中，可以通过 flink-cdc-connectors 项目提供的 changelog-json format 来实现该功能，具体见文档。

> 文档链接：  
> [https://github.com/ververica/flink-cdc-connectors/wiki/Changelog-JSON-Format](https://github.com/ververica/flink-cdc-connectors/wiki/Changelog-JSON-Format)

**2、CDC 是否需要保证顺序化消费？**

是的，数据同步到 kafka ，首先需要 kafka 在分区中保证有序，同一个 key 的变更数据需要打入到同一个 kafka 的分区里面。这样 flink 读取的时候才能保证顺序。 
 [https://developer.aliyun.com/article/777502](https://developer.aliyun.com/article/777502)
````
