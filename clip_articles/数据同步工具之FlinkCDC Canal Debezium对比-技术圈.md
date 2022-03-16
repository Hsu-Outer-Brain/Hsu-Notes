# 数据同步工具之FlinkCDC/Canal/Debezium对比-技术圈
## 前言

数据准实时复制（CDC）是目前行内实时数据需求大量使用的技术，随着国产化的需求，我们也逐步考虑基于开源产品进行准实时数据同步工具的相关开发，逐步实现对商业产品的替代。本文把市面上常见的几种开源产品，Canal、Debezium、Flink CDC 从原理和适用做了对比，供大家参考。

> 本文首发微信公众号《import_bigdata》

### Debezium

> Debezium is an open source distributed platform for change data capture. Start it up, point it at your databases, and your apps can start responding to all of the inserts, updates, and deletes that other apps commit to your databases. Debezium is durable and fast, so your apps can respond quickly and never miss an event, even when things go wrong.

Debezium 是一种 CDC（Change Data Capture）工具，工作原理类似大家所熟知的 Canal, DataBus, Maxwell 等，是通过抽取数据库日志来获取变更。

Debezium 最初设计成一个 Kafka Connect 的 Source Plugin，目前开发者虽致力于将其与 Kafka Connect 解耦，但当前的代码实现还未变动。下图引自 Debeizum 官方文档，可以看到一个 Debezium 在一个完整 CDC 系统中的位置。

![](https://filescdn.proginn.com/b3a928c97a568c48ebef867672693abc/be52c146afcab849ad5f27b8e7fbd5de.webp)

Kafka Connect 为 Source Plugin 提供了一系列的编程接口，最主要的就是要实现 SourceTask 的 poll 方法，其返回`List<SourceRecord>`将会被以最少一次语义的方式投递至 Kafka。

#### Debezium MySQL 架构

![](https://filescdn.proginn.com/789cad63362a1dfb80efb4c160ed8981/1a195756c164459e3ded9500115c561a.webp)

Debezium 抽取原理

Reader 体系构成了 MySQL 模块中代码的主线，我们的分析从 Reader 开始。

![](https://filescdn.proginn.com/ab8da090cad3365d86aa4e660a7909d1/61999f94e706f06115e87c4b633d0dfe.webp)

Reader 继承关系

从名字上应该可以看出，真正主要的是 SnapshotReader 和 BinlogReader，分别实现了对 MySQL 数据的全量读取和增量读取，他们继承于 AbstractReader，里面封装了共用逻辑，下图是 AbstractReader 的内部设计。

![](https://filescdn.proginn.com/8b66e9c48aaaa4cab0e4a142796c583e/b014d49d506fc84ee1e8987b0213f361.webp)

可以看到，AbstractReader 在实现时，并没有直接将 enqueue 喂进来的 record 投递进 Kafka，而是通过一个内存阻塞队列 BlockingQueue 进行了解耦，这种设计有诸多好处：

1.  职责解耦

如上的图中，在喂入 BlockingQueue 之前，要根据条件判断是否接受该 record；在向 Kafka 投递 record 之前，判断 task 的 running 状态。这样把同类的功能限定在特定的位置。

2.  线程隔离

BlockingQueue 是一个线程安全的阻塞队列，通过 BlockingQueue 实现的生产者消费者模型，是可以跑在不同的线程里的，这样避免局部的阻塞带来的整体的干扰。如上图中的右侧，消费者会定期判断 running 标志位，若 running 被 stop 信号置为了 false，可以立刻停止整个 task, 而不会因 MySQL IO 阻塞延迟相应。

3.  Single 与 Batch 的互相转化

Enqueue record 是单条的投递 record，drain_to 是批量的消费 records。这个用法也可以反过来，实现 batch 到 single 的转化。

可能你还知道阿里开源的另一个 MySQL CDC 工具 canal，他只负责 stream 过程，并没有处理 snapshot 过程，这也是 debezium 相较于 canal 的一个优势。

对于 Debezium 来说，基本沿用了官方搭建从库的这一思路，让我们看下官方文档描述的详细步骤。

MySQL 连接器每次获取快照的时候会执行以下的步骤：

1.  获取一个全局读锁，从而阻塞住其他数据库客户端的写操作。
2.  开启一个可重复读语义的事务，来保证后续的在同一个事务内读操作都是在一个一致性快照中完成的。
3.  读取 binlog 的当前位置。
4.  读取连接器中配置的数据库和表的模式（schema）信息。
5.  释放全局读锁，允许其他的数据库客户端对数据库进行写操作。
6.  （可选）把 DDL 改变事件写入模式改变 topic（schema change topic），包括所有的必要的 DROP 和 CREATEDDL 语句。
7.  扫描所有数据库的表，并且为每一个表产生一个和特定表相关的 kafka topic 创建事件（即为每一个表创建一个 kafka topic）。
8.  提交事务。
9.  记录连接器成功完成快照任务时的连接器偏移量。

#### 部署

##### 基于 Kafka Connect

最常见的架构是通过 Apache Kafka Connect 部署 Debezium。Kafka Connect 为在 Kafka 和外部存储系统之间系统数据提供了一种可靠且可伸缩性的方式。它为 Connector 插件提供了一组 API 和一个运行时：Connect 负责运行这些插件，它们则负责移动数据。通过 Kafka Connect 可以快速实现 Source Connector 和 Sink Connector 进行交互构造一个低延迟的数据 Pipeline：

-   Source Connector（例如，Debezium）：将记录发送到 Kafka
-   Sink Connector：将 Kafka Topic 中的记录发送到其他系统

![](https://filescdn.proginn.com/1498c2f30dc76c84989f81a1a7029e43/8b4f7c420d88d43fa11a68286d9e77cb.webp)

如上图所示，部署了 MySQL 和 PostgresSQL 的 Debezium Connector 以捕获这两种类型数据库的变更。每个 Debezium Connector 都会与其源数据库建立连接：

-   MySQL Connector 使用客户端库来访问 binlog。
-   PostgreSQL Connector 从逻辑副本流中读取数据。

除了 Kafka Broker 之外，Kafka Connect 也作为一个单独的服务运行。默认情况下，数据库表的变更会写入名称与表名称对应的 Kafka Topic 中。如果需要，您可以通过配置 Debezium 的 Topic 路由转换来调整目标 Topic 名称。例如，您可以：

-   将记录路由到名称与表名不同的 Topic 中
-   将多个表的变更事件记录流式传输到一个 Topic 中

变更事件记录在 Apache Kafka 中后，Kafka Connect 生态系统中的不同 Sink Connector 可以将记录流式传输到其他系统、数据库，例如 Elasticsearch、数据仓库、分析系统或者缓存（例如 Infinispan）。

##### Debezium Server

另一种部署 Debezium 的方法是使用 Debezium Server。Debezium Server 是一个可配置的、随时可用的应用程序，可以将变更事件从源数据库流式传输到各种消息中间件上。

下图展示了基于 Debezium Server 的变更数据捕获 Pipeline 架构：

![](https://filescdn.proginn.com/64e54b6b8ea4238f9e704cfbda80d813/ff16300a88216ec18a497378a7ce2b75.webp)

Debezium Server 配置使用 Debezium Source Connector 来捕获源数据库中的变更。变更事件可以序列化为不同的格式，例如 JSON 或 Apache Avro，然后发送到各种消息中间件，例如 Amazon Kinesis、Google Cloud Pub/Sub 或 Apache Pulsar。

##### 嵌入式引擎

使用 Debezium Connector 的另一种方法是嵌入式引擎。在这种情况下，Debezium 不会通过 Kafka Connect 运行，而是作为嵌入到您自定义 Java 应用程序中的库运行。这对于在您的应用程序本身内获取变更事件非常有帮助，无需部署完整的 Kafka 和 Kafka Connect 集群，也不用将变更流式传输到 Amazon Kinesis 等消息中间件上。

#### 特性

Debezium 是一组用于 Apache Kafka Connect 的 Source Connector。每个 Connector 都通过使用该数据库的变更数据捕获 (CDC) 功能从不同的数据库中获取变更。与其他方法（例如轮询或双重写入）不同，Debezium 的实现基于日志的 CDC：

-   确保捕获所有的数据变更。
-   以极低的延迟生成变更事件，同时避免因为频繁轮询导致 CPU 使用率增加。例如，对于 MySQL 或 PostgreSQL，延迟在毫秒范围内。
-   不需要更改您的数据模型，例如 ‘Last Updated’ 列。
-   可以捕获删除操作。
-   可以捕获旧记录状态以及其他元数据，例如，事务 ID，具体取决于数据库的功能和配置。

### Flink CDC

-   2020 年 7 月提交了第一个 commit，这是基于个人兴趣孵化的项目；
-   2020 年 7 中旬支持了 MySQL-CDC；
-   2020 年 7 月末支持了 Postgres-CDC；

一年的时间，该项目在 GitHub 上的 star 数已经超过 800。

![](https://filescdn.proginn.com/6260f47533c281f4f3abc1055df438d7/b7d4fd62a0402da92b208f46fad7abea.webp)

#### Flink CDC 发展

Flink CDC 底层封装了 Debezium， Debezium 同步一张表分为两个阶段：

-   全量阶段：查询当前表中所有记录；
-   增量阶段：从 binlog 消费变更数据。

大部分用户使用的场景都是全量 + 增量同步，加锁是发生在全量阶段，目的是为了确定全量阶段的初始位点，保证增量 + 全量实现一条不多，一条不少，从而保证数据一致性。从下图中我们可以分析全局锁和表锁的一些加锁流程，左边红色线条是锁的生命周期，右边是 MySQL 开启可重复读事务的生命周期。

![](https://filescdn.proginn.com/8ac9c8448e1c7e07e0275cb68027aed2/9d589e58cc9b5aa1aba4ae14e77ff88f.webp)

以全局锁为例，首先是获取一个锁，然后再去开启可重复读的事务。这里锁住操作是读取 binlog 的起始位置和当前表的 schema。这样做的目的是保证 binlog 的起始位置和读取到的当前 schema 是可以对应上的，因为表的 schema 是会改变的，比如如删除列或者增加列。在读取这两个信息后，SnapshotReader 会在可重复读事务里读取全量数据，在全量数据读取完成后，会启动 BinlogReader 从读取的 binlog 起始位置开始增量读取，从而保证全量数据 + 增量数据的无缝衔接。

表锁是全局锁的退化版，因为全局锁的权限会比较高，因此在某些场景，用户只有表锁。表锁锁的时间会更长，因为表锁有个特征：锁提前释放了可重复读的事务默认会提交，所以锁需要等到全量数据读完后才能释放。

经过上面分析，接下来看看这些锁到底会造成怎样严重的后果：

![](https://filescdn.proginn.com/10b5222f24e46b19337dce08b01aa159/7eb734fe6fc7cd5426d24cd799b8e121.webp)

Flink CDC 1.x 可以不加锁，能够满足大部分场景，但牺牲了一定的数据准确性。Flink CDC 1.x 默认加全局锁，虽然能保证数据一致性，但存在上述 hang 住数据的风险。

Flink CDC 1.x 得到了很多用户在社区的反馈，主要归纳为三个：

![](https://filescdn.proginn.com/704e9423a78d7bf7b081d0cd8be1c4de/5485a5ce44ab873d23995005a100af9c.webp)

-   全量 + 增量读取的过程需要保证所有数据的一致性，因此需要通过加锁保证，但是加锁在数据库层面上是一个十分高危的操作。底层 Debezium 在保证数据一致性时，需要对读取的库或表加锁，全局锁可能导致数据库锁住，表级锁会锁住表的读，DBA 一般不给锁权限。
-   不支持水平扩展，因为 Flink CDC 底层是基于 Debezium，起架构是单节点，所以 Flink CDC 只支持单并发。在全量阶段读取阶段，如果表非常大 (亿级别)，读取时间在小时甚至天级别，用户不能通过增加资源去提升作业速度。
-   全量读取阶段不支持 checkpoint：CDC 读取分为两个阶段，全量读取和增量读取，目前全量读取阶段是不支持 checkpoint 的，因此会存在一个问题：当我们同步全量数据时，假设需要 5 个小时，当我们同步了 4 小时的时候作业失败，这时候就需要重新开始，再读取 5 个小时。

通过上面的分析，可以知道 2.0 的设计方案，核心要解决上述的三个问题，即支持无锁、水平扩展、checkpoint。

![](https://filescdn.proginn.com/62cb88c16be3e82cb1ef7873c5e62314/3fde547f99b55512a2c5f0258a93b421.webp)

目前，Flink CDC 2.0 也已经正式发布，此次的核心改进和提升包括：

-   并发读取，全量数据的读取性能可以水平扩展；
-   全程无锁，不对线上业务产生锁的风险；
-   断点续传，支持全量阶段的 checkpoint。

> 本文发自微信公众号《import_bigdata》

### Canal

canal \[kə'næl]，译意为水道 / 管道 / 沟渠，主要用途是基于 MySQL 数据库增量日志解析，提供增量数据订阅和消费。

早期阿里巴巴因为杭州和美国双机房部署，存在跨机房同步的业务需求，实现方式主要是基于业务 trigger 获取增量变更。从 2010 年开始，业务逐步尝试数据库日志解析获取增量变更进行同步，由此衍生出了大量的数据库增量订阅和消费业务。

基于日志增量订阅和消费的业务包括：

-   数据库镜像
-   数据库实时备份
-   索引构建和实时维护 (拆分异构索引、倒排索引等)
-   业务 cache 刷新
-   带业务逻辑的增量数据处理

**当前的 canal 支持源端 MySQL 版本**包括 5.1.x , 5.5.x , 5.6.x , 5.7.x , 8.0.x。

#### 工作原理

**MySQL 主备复制原理**

![](https://filescdn.proginn.com/53704c9541ae1aa99a71a7e34a56e8b2/80a4dfe48edecc7452f4b3c7f677e64d.webp)

-   MySQL master 将数据变更写入二进制日志 (binary log, 其中记录叫做二进制日志事件 binary log events，可以通过 show binlog events 进行查看)
-   MySQL slave 将 master 的 binary log events 拷贝到它的中继日志 (relay log)
-   MySQL slave 重放 relay log 中事件，将数据变更反映它自己的数据

**canal 工作原理**

-   canal 模拟 MySQL slave 的交互协议，伪装自己为 MySQL slave, 向 MySQL master 发送 dump 协议
-   MySQL master 收到 dump 请求，开始推送 binary log 给 slave (即 canal)
-   canal 解析 binary log 对象 (原始为 byte 流)

#### Binlog 获取详解

Binlog 发送接收流程，流程如下图所示:

![](https://filescdn.proginn.com/c2ec0b12686363ecb23c75654b2110c4/fd75578e0540fa59a92e525b0d4f889e.webp)

首先，我们需要伪造一个 slave，向 master 注册，这样 master 才会发送 binlog event。注册很简单，就是向 master 发送 COM_REGISTER_SLAVE 命令，带上 slave 相关信息。这里需要注意，因为在 MySQL 的 replication topology 中，都需要使用一个唯一的 server id 来区别标示不同的 server 实例，所以这里我们伪造的 slave 也需要一个唯一的 server id。

接着实现 binlog 的 dump。MySQL 只支持一种 binlog dump 方式，也就是指定 binlog filename + position，向 master 发送 COM_BINLOG_DUMP 命令。在发送 dump 命令的时候，我们可以指定 flag 为 BINLOG_DUMP_NON_BLOCK，这样 master 在没有可发送的 binlog event 之后，就会返回一个 EOF package。不过通常对于 slave 来说，一直把连接挂着可能更好，这样能更及时收到新产生的 binlog event。

Dump 命令包图如下所示:

![](https://filescdn.proginn.com/e50bd094fdeca5db8841bce6672f723c/e3e8dcbda217354469e025313b8c6212.webp)

如上图所示, 在报文中塞入 binlogPosition 和 binlogFileName 即可让 master 从相应的位置发送 binlog event。

#### canal 结构

![](https://filescdn.proginn.com/e3876f7de70c73c957b77e17f8345489/cbcf2513c390aa84ac9fca2bc8a8c27c.webp)

说明：

-   server 代表一个 canal 运行实例，对应于一个 jvm，也可以理解为一个进程
-   instance 对应于一个数据队列 （1 个 server 对应 1..n 个 instance)，每一个数据队列可以理解为一个数据库实例。

#### Server 设计

![](https://filescdn.proginn.com/967e1399036e10b41c44bce63d4ffdd6/9954b0ff61456f35965260e13843dfc9.webp)

server 代表了一个 canal 的运行实例，为了方便组件化使用，特意抽象了 Embeded(嵌入式) / Netty(网络访问) 的两种实现

-   Embeded : 对 latency 和可用性都有比较高的要求，自己又能 hold 住分布式的相关技术 (比如 failover)
-   Netty : 基于 netty 封装了一层网络协议，由 canal server 保证其可用性，采用的 pull 模型，当然 latency 会稍微打点折扣，不过这个也视情况而定。(阿里系的 notify 和 metaq，典型的 push/pull 模型，目前也逐步的在向 pull 模型靠拢，push 在数据量大的时候会有一些问题)

#### Instance 设计

![](https://filescdn.proginn.com/0b18a349c4287788b19ea432719aac52/21e2856411588ee8f77ee146ec9a19dd.webp)

instance 代表了一个实际运行的数据队列，包括了 EventPaser,EventSink,EventStore 等组件。

抽象了 CanalInstanceGenerator，主要是考虑配置的管理方式：

manager 方式：和你自己的内部 web console/manager 系统进行对接。(目前主要是公司内部使用，Otter 采用这种方式) spring 方式：基于 spring xml + properties 进行定义，构建 spring 配置.

下面是 canalServer 和 instance 如何运行：

\`canalServer.setCanalInstanceGenerator(new CanalInstanceGenerator() {

            public CanalInstance generate(String destination) {  
                Canal canal = canalConfigClient.findCanal(destination);  
                // 此处省略部分代码 大致逻辑是设置 canal 一些属性

                CanalInstanceWithManager instance = new CanalInstanceWithManager(canal, filter) {

                    protected CanalHAController initHaController() {  
                        HAMode haMode = parameters.getHaMode();  
                        if (haMode.isMedia()) {  
                            return new MediaHAController(parameters.getMediaGroup(),  
                                parameters.getDbUsername(),  
                                parameters.getDbPassword(),  
                                parameters.getDefaultDatabaseName());  
                        } else {  
                            return super.initHaController();  
                        }  
                    }

                    protected void startEventParserInternal(CanalEventParser parser, boolean isGroup) {  
                        // 大致逻辑是 设置支持的类型  
                        // 初始化设置 MysqlEventParser 的主库信息，这处抽象不好，目前只支持 mysql  
                    }

                };  
                return instance;  
            }  
        });  
        canalServer.start(); // 启动 canalServer

        canalServer.start(destination);// 启动对应 instance  
        this.clientIdentity = new ClientIdentity(destination, pipeline.getParameters().getMainstemClientId(), filter);  
        canalServer.subscribe(clientIdentity);// 发起一次订阅，当监听到 instance 配置时，调用 generate 方法注入新的 instance

\`

instance 模块：

-   eventParser (数据源接入，模拟 slave 协议和 master 进行交互，协议解析)
-   eventSink (Parser 和 Store 链接器，进行数据过滤，加工，分发的工作)
-   eventStore (数据存储)
-   metaManager (增量订阅 & 消费信息管理器)

#### EventParser 设计

大致过程：

![](https://filescdn.proginn.com/d230e190bdf6bedafc28dec70fce5f6e/5a369b870f597237e4d0e38afbdf88ea.webp)

整个 parser 过程大致可分为几步：

-   Connection 获取上一次解析成功的位置 (如果第一次启动，则获取初始指定的位置或者是当前数据库的 binlog 位点)
-   Connection 建立链接，发送 BINLOG_DUMP 指令

`// 0. write command number  
// 1. write 4 bytes bin-log position to start at  
// 2. write 2 bytes bin-log flags  
// 3. write 4 bytes server id of the slave  
// 4. write bin-log file name  
`

-   Mysql 开始推送 Binaly Log
-   接收到的 Binaly Log 的通过 Binlog parser 进行协议解析，补充一些特定信息 (补充字段名字，字段类型，主键信息，unsigned 类型处理)
-   传递给 EventSink 模块进行数据存储，是一个阻塞操作，直到存储成功
-   存储成功后，由 CanalLogPositionManager 定时记录 Binaly Log 位置

#### EventSink 设计

![](https://filescdn.proginn.com/89e1cadc45ebb42b3f6ebfd04d08ab26/8679b22f6ffdd35e7879b639de3a1bbb.webp)

说明：

-   数据过滤：支持通配符的过滤模式，表名，字段内容等
-   数据路由 / 分发：解决 1:n (1 个 parser 对应多个 store 的模式)
-   数据归并：解决 n:1 (多个 parser 对应 1 个 store)
-   数据加工：在进入 store 之前进行额外的处理，比如 join

**数据 1:n 业务**

为了合理的利用数据库资源， 一般常见的业务都是按照 schema 进行隔离，然后在 mysql 上层或者 dao 这一层面上，进行一个数据源路由，屏蔽数据库物理位置对开发的影响，阿里系主要是通过 cobar/tddl 来解决数据源路由问题。

所以，一般一个数据库实例上，会部署多个 schema，每个 schema 会有由 1 个或者多个业务方关注。

**数据 n:1 业务**

同样，当一个业务的数据规模达到一定的量级后，必然会涉及到水平拆分和垂直拆分的问题，针对这些拆分的数据需要处理时，就需要链接多个 store 进行处理，消费的位点就会变成多份，而且数据消费的进度无法得到尽可能有序的保证。

所以，在一定业务场景下，需要将拆分后的增量数据进行归并处理，比如按照时间戳 / 全局 id 进行排序归并。

#### EventStore 设计

1.  目前仅实现了 Memory 内存模式，后续计划增加本地 file 存储，mixed 混合模式。
2.  借鉴了 Disruptor 的 RingBuffer 的实现思路

RingBuffer 设计：

![](https://filescdn.proginn.com/c07a36d6234b06daadce941f0cc3e757/3bd0c7600cfb45d05c6826834d40c057.webp)

定义了 3 个 cursor

Put : Sink 模块进行数据存储的最后一次写入位置 Get : 数据订阅获取的最后一次提取位置 Ack : 数据消费成功的最后一次消费位置

借鉴 Disruptor 的 RingBuffer 的实现，将 RingBuffer 拉直来看：

![](https://filescdn.proginn.com/b32902a1df77414a2a5056ef09c64688/139557144a8c05ea6822e49ab70dea1f.webp)

实现说明：

Put/Get/Ack cursor 用于递增，采用 long 型存储 buffer 的 get 操作，通过取余或者与操作。(与操作：cusor & (size - 1) , size 需要为 2 的指数，效率比较高)

#### HA 机制设计

canal 的 ha 分为两部分，canal server 和 canal client 分别有对应的 ha 实现

-   canal server: 为了减少对 mysql dump 的请求，不同 server 上的 instance 要求同一时间只能有一个处于 running，其他的处于 standby 状态.
-   canal client: 为了保证有序性，一份 instance 同一时间只能由一个 canal client 进行 get/ack/rollback 操作，否则客户端接收无法保证有序。

整个 HA 机制的控制主要是依赖了 zookeeper 的几个特性，watcher 和 EPHEMERAL 节点 (和 session 生命周期绑定)，可以看下我之前 zookeeper 的相关文章。

Canal Server:

![](https://filescdn.proginn.com/5a507f75139c733a39ec1ab2ecd34bf9/942851ac3b9c84f0f148e727c6785792.webp)

大致步骤：

-   canal server 要启动某个 canal instance 时都先向 zookeeper 进行一次尝试启动判断 (实现：创建 EPHEMERAL 节点，谁创建成功就允许谁启动)
-   创建 zookeeper 节点成功后，对应的 canal server 就启动对应的 canal instance，没有创建成功的 canal instance 就会处于 standby 状态
-   一旦 zookeeper 发现 canal server A 创建的节点消失后，立即通知其他的 canal server 再次进行步骤 1 的操作，重新选出一个 canal server 启动 instance
-   canal client 每次进行 connect 时，会首先向 zookeeper 询问当前是谁启动了 canal instance，然后和其建立链接，一旦链接不可用，会重新尝试 connect

Canal Client 的方式和 canal server 方式类似，也是利用 zookeeper 的抢占 EPHEMERAL 节点的方式进行控制。

> 本文发自微信公众号《import_bigdata》

### 总结

CDC 的技术方案非常多，目前业界主流的实现机制可以分为两种：

**基于查询的 CDC：** 

-   离线调度查询作业，批处理。把一张表同步到其他系统，每次通过查询去获取表中最新的数据；
-   无法保障数据一致性，查的过程中有可能数据已经发生了多次变更；
-   不保障实时性，基于离线调度存在天然的延迟。

**基于日志的 CDC：** 

-   实时消费日志，流处理，例如 MySQL 的 binlog 日志完整记录了数据库中的变更，可以把 binlog 文件当作流的数据源；
-   保障数据一致性，因为 binlog 文件包含了所有历史变更明细；
-   保障实时性，因为类似 binlog 的日志文件是可以流式消费的，提供的是实时数据。

对比常见的开源 CDC 方案，我们可以发现：

![](https://filescdn.proginn.com/fe533b3f620f580d4a2650a44258df98/b6dfa94cd1049c800f5d38a79bcc5f65.webp)

-   对比增量同步能力:

        - 基于日志的方式，可以很好的做到增量同步；- 而基于查询的方式是很难做到增量同步的。
-   对比全量同步能力，基于查询或者日志的 CDC 方案基本都支持，除了 Canal。
-   而对比全量 + 增量同步的能力，只有 Flink CDC、Debezium、Oracle Goldengate 支持较好。
-   从架构角度去看，该表将架构分为单机和分布式，这里的分布式架构不单纯体现在数据读取能力的水平扩展上，更重要的是在大数据场景下分布式系统接入能力。例如 Flink CDC 的数据入湖或者入仓的时候，下游通常是分布式的系统，如 Hive、HDFS、Iceberg、Hudi 等，那么从对接入分布式系统能力上看，Flink CDC 的架构能够很好地接入此类系统。
-   在数据转换 / 数据清洗能力上，当数据进入到 CDC 工具的时候是否能较方便的对数据做一些过滤或者清洗，甚至聚合？


-   在 Flink CDC 上操作相当简单，可以通过 Flink SQL 去操作这些数据；
-   但是像 DataX、Debezium 等则需要通过脚本或者模板去做，所以用户的使用门槛会比较高。

另外，在生态方面，这里指的是下游的一些数据库或者数据源的支持。Flink CDC 下游有丰富的 Connector，例如写入到 TiDB、MySQL、Pg、HBase、Kafka、ClickHouse 等常见的一些系统，也支持各种自定义 connector。 
 [https://jishuin.proginn.com/p/763bfbd6a1f5](https://jishuin.proginn.com/p/763bfbd6a1f5)
