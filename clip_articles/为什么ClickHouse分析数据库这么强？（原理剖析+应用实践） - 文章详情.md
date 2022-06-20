# 为什么ClickHouse分析数据库这么强？（原理剖析+应用实践） - 文章详情
#### ![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/819F2DEE9A549351B4FAA191A49E17D1.jpg)

ClickHouse 简介

2020 年下半年在 OLAP 领域有一匹黑马以席卷之势进入大数据开发者的领域，它就是 ClickHouse。在 2019 年小编也曾介绍过 ClickHouse，大家可以参考这里进行入门：

[来自俄罗斯的凶猛彪悍的分析数据库 - ClickHouse](https://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247492821&idx=1&sn=6b27fb340afc633b6152cb9e022db611&chksm=fd3ea240ca492b56a34ee8a140528a6b3a2e6c82f164b1b622dbbbb8b54d89aa3d401283346f&token=1901516241&lang=zh_CN&scene=21#wechat_redirect)

[基于 ClickHouse 的用户行为分析实践](https://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247492803&idx=1&sn=04283aa0913674e158a8519120fc14cd&chksm=fd3ea256ca492b40f60ac90eee4d23f9c2b8156a9c83439297317efcb826b32779b762f554e4&token=1901516241&lang=zh_CN&scene=21#wechat_redirect)

[Prometheus+Clickhouse 实现业务告警](https://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247489470&idx=1&sn=734886246f2413f72d605cad726b59f2&chksm=fd3d512bca4ad83d516e5d6f0dd4c21c83867de758238277a162fecad0ec21a9eff9a3a59c23&token=1901516241&lang=zh_CN&scene=21#wechat_redirect)

那么我们有必要先从全局了解一下 ClickHouse 到底是个什么样的数据库？

ClickHouse 是一个开源的，面向列的分析数据库，由 Yandex 为 OLAP 和大数据用例创建。ClickHouse 对实时查询处理的支持使其适用于需要亚秒级分析结果的应用程序。ClickHouse 的查询语言是 SQL 的一种方言，它支持强大的声明性查询功能，同时为最终用户提供熟悉度和较小的学习曲线。

ClickHouse 全称是 Click Stream,Data Warehouse，简称 ClickHouse 就是基于页面的点击事件流，面向数据仓库进行 OLAP 分析。ClickHouse 是一款开源的数据分析数据库，由战斗民族俄罗斯 Yandex 公司研发的，Yandex 是做搜索引擎的，就类似与 Google，百度等。

我们都知道搜索引擎的营收主要来源与流量和广告业务，所以搜索引擎公司会着重分析用户网路流量，像 Google 有 Anlytics，百度有百度统计，那么 Yandex 就对应于 Yandex.Metrica。ClickHouse 就式在 Yandex.Metrica 下产生的技术。

面向列的数据库将记录存储在按列而不是行分组的块中。通过不加载查询中不存在的列的数据，面向列的数据库在完成查询时花费的时间更少。因此，对于某些工作负载（如 OLAP），这些数据库可以比传统的基于行的系统更快地计算和返回结果。

#### 为什么 ClickHouse 能够异军突起

**优异的性能**

根据官网的介绍 ([https://clickhouse.tech/benchmark/dbms/](https://clickhouse.tech/benchmark/dbms/))，ClickHouse 在相同的服务器配置与数据量下，平均响应速度：

-   Vertica 的 2.63 倍 (Vertica 是一款收费的列式存储数据库)
-   InfiniDB 的 17 倍 (可伸缩的分析数据库引擎，基于 Mysql 搭建)
-   MonetDB 的 27 倍 (开源的列式数据库)
-   Hive 的 126 倍
-   MySQL 的 429 倍
-   Greenplum 的 10 倍
-   Spark 的 1 倍

性能是衡量 OLAP 数据库的关键指标，我们可以通过 ClickHouse 官方测试结果感受下 ClickHouse 的极致性能，其中绿色代表性能最佳，红色代表性能较差，红色越深代表性能越弱。

![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/33379ABEACEC9A25222342DE4CA86476.jpg)

我们在之前用了一篇文章[《数据告诉你 ClickHouse 有多快》](https://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247493466&idx=1&sn=bc5bddf48e3b8227dc2796ff883062dc&chksm=fd3ea1cfca4928d918cbf978dad994f6f53e5ea54cd0cbbd62e983d541f8db9ee3808f22562c&token=1901516241&lang=zh_CN&scene=21#wechat_redirect) 详细讲解了 ClickHouse 的优异性能表现。

**优势和局限性**

ClickHouse 主要特点：

-   ROLAP(关系型的联机分析处理，和它一起比较的还有 OLTP 联机事务处理，我们常见的 ERP,CRM 系统就属于 OLTP)
-   在线实时查询
-   完整的 DBMS(关系数据库)
-   列式存储 (区别与 HBase，ClickHouse 的是完全列式存储，HBase 具体说是列族式存储)
-   不需要任何数据预处理
-   支持批量更新
-   拥有完善的 SQl 支持和函数
-   支持高可用 (多主结构，在后面的结构设计中会讲到)
-   不依赖 Hadoop 复杂生态 (像 ES 一样，开箱即用)

一些不足：

-   不支持事务 (这其实也是大部分 OLAP 数据库的缺点)
-   不擅长根据主键按行粒度查询 (但是支持这种操作)
-   不擅长按行删除数据 (但是支持这种操作)

#### 核心概念和原理

ClickHouse 采用了典型的分组式的分布式架构，集群架构如下图所示：

![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/EBD4082E3CCE02DB799232D952462E0C.jpg)

这其中的角色包括：

-   Shard ：集群内划分为多个分片或分组（Shard 0 … Shard N），通过 Shard 的线性扩展能力，支持海量数据的分布式存储计算。
-   Node ：每个 Shard 内包含一定数量的节点（Node，即进程），同一 Shard 内的节点互为副本，保障数据可靠。ClickHouse 中副本数可按需建设，且逻辑上不同 Shard 内的副本数可不同。
-   ZooKeeper Service ：集群所有节点对等，节点间通过 ZooKeeper 服务进行分布式协调。

ClickHouse 基础架构如下图所示：

![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/8832F2F35C7A4DEACB83FC575F8D9CA2.jpg)

-   Column 与 Field

Column 和 Field 是 ClickHouse 数据最基础的映射单元。内存中的一列数据由一个 Column 对象表示。Column 对象分为接口和实现两个部分，在 IColumn 接口对象中，定义了对数据进行各种关系运算的方法。在大多数场合，ClickHouse 都会以整列的方式操作数据，但凡事也有例外。如果需要操作单个具体的数值 (也就是单列中的一行数据)，则需要使用 Field 对象，Field 对象代表一个单值。与 Column 对象的泛化设计思路不同，Field 对象使用了聚合的设计模式。在 Field 对象内部聚合了 Null、UInt64、String 和 Array 等 13 种数据类型及相应的处理逻辑。

-   DataType

数据的序列化和反序列化工作由 DataType 负责。IDataType 接口定义了许多正反序列化的方法，它们成对出现。IDataType 也使用了泛化的设计模式，具体方法的实现逻辑由对应数据类型的实例承载。DataType 虽然负责序列化相关工作，但它并不直接负责数据的读取，而是转由从 Column 或 Field 对象获取。

-   Block 与 Block 流

ClickHouse 内部的数据操作是面向 Block 对象进行的，并且采用了流的形式。Block 对象可以看作数据表的子集。Block 对象的本质是由数据对象、数据类型和列名称组成的三元组，即 Column、DataType 及列名称字符串。仅通过 Block 对象就能完成一系列的数据操作。Block 并没有直接聚合 Column 和 DataType 对象，而是通过 ColumnWithTypeAndName 对象进行间接引用。Block 流操作有两组顶层接口：IBlockInputStream 负责数据的读取和关系运算，IBlockOutputStream 负责将数据输出到下一环节。IBlockInputStream 接口定义了读取数据的若干个 read 虚方法，而具体的实现逻辑则交由它的实现类来填充。IBlockInputStream 接口总共有 60 多个实现类，这些实现类大致可以分为三类：

-   第一类用于处理数据定义的 DDL 操作
-   第二类用于处理关系运算的相关操作
-   第三类则是与表引擎呼应，每一种表引擎都拥有与之对应的 BlockInputStream 实现

IBlockOutputStream 的设计与 IBlockInputStream 如出一辙。这些实现类基本用于表引擎的相关处理，负责将数据写入下一环节或者最终目的地。

-   Table

在数据表的底层设计中并没有所谓的 Table 对象，它直接使用 IStorage 接口指代数据表。表引擎是 ClickHouse 的一个显著特性，不同的表引擎由不同的子类实现。IStorage 接口负责数据的定义、查询与写入。IStorage 负责根据 AST 查询语句的指示要求，返回指定列的原始数据。后续的加工、计算和过滤则由下面介绍的部分进行。

-   Parser 与 Interpreter

Parser 分析器负责创建 AST 对象；而 Interpreter 解释器则负责解释 AST，并进一步创建查询的执行管道。它们与 IStorage 一起，串联起了整个数据查询的过程。Parser 分析器可以将一条 SQL 语句以递归下降的方法解析成 AST 语法树的形式。不同的 SQL 语句，会经由不同的 Parser 实现类解析。Interpreter 解释器的作用就像 Service 服务层一样，起到串联整个查询过程的作用，它会根据解释器的类型，聚合它所需要的资源。首先它会解析 AST 对象；然后执行 "业务逻辑" (例如分支判断、设置参数、调用接口等)；最终返回 IBlock 对象，以线程的形式建立起一个查询执行管道。

-   Functions 与 Aggregate Functions

ClickHouse 主要提供两类函数—普通函数（Functions）和聚合函数（Aggregate Functions）。普通函数由 IFunction 接口定义，拥有数十种函数实现，采用向量化的方式直接作用于一整列数据。聚合函数由 IAggregateFunction 接口定义，相比无状态的普通函数，聚合函数是有状态的。以 COUNT 聚合函数为例，其 AggregateFunctionCount 的状态使用整型 UInt64 记录。聚合函数的状态支持序列化与反序列化，所以能够在分布式节点之间进行传输，以实现增量计算。

-   Cluster 与 Replication

ClickHouse 的集群由分片 (Shard) 组成，而每个分片又通过副本 ( Replica ) 组成。这种分层的概念，在一些流行的分布式系统中十分普遍。这里有几个与众不同的特性。ClickHouse 的 1 个节点只能拥有 1 个分片，也就是说如果要实现 1 分片、1 副本，则至少需要部署 2 个服务节点。分片只是一个逻辑概念，其物理承载还是由副本承担的。

#### ClickHouse 的核心特性

ClickHouse 的特性有很多，一款被认可的 OLAP 引擎能够得到大家的频繁使用一定是有独特的特性，小编列举了几个 ClickHouse 异于常人的特性：

**列式存储 & 数据压缩**

按列存储与按行存储相比，前者可以有效减少查询时所需扫描的数据量，这一点可以用一个示例简单说明。假设一张数据表 A 拥有 50 个字段 A1～A50，以及 100 行数据。现在需要查询前 5 个字段并进行数据分析，那么通过列存储，我们仅需读取必要的列数据，相比于普通行存，可减少 10 倍左右的读取、解压、处理等开销，对性能会有质的影响。

如果数据按行存储，数据库首先会逐行扫描，并获取每行数据的所有 50 个字段，再从每一行数据中返回 A1～A5 这 5 个字段。不难发现，尽管只需要前面的 5 个字段，但由于数据是按行进行组织的，实际上还是扫描了所有的字段。如果数据按列存储，就不会发生这样的问题。由于数据按列组织，数据库可以直接获取 A1～A5 这 5 列的数据，从而避免了多余的数据扫描。

按列存储相比按行存储的另一个优势是对数据压缩的友好性。ClickHouse 的数据按列进行组织，属于同一列的数据会被保存在一起，列与列之间也会由不同的文件分别保存 (这里主要指 MergeTree 表引擎)。数据默认使用 LZ4 算法压缩，在 Yandex.Metrica 的生产环境中，数据总体的压缩比可以达到 8:1 ( 未压缩前 17PB，压缩后 2PB )。列式存储除了降低 IO 和存储的压力之外，还为向量化执行做好了铺垫。

**向量化执行**

坊间有句玩笑，即 "能用钱解决的问题，千万别花时间"。而业界也有种调侃如出一辙，即 "能升级硬件解决的问题，千万别优化程序"。有时候，你千辛万苦优化程序逻辑带来的性能提升，还不如直接升级硬件来得简单直接。这虽然只是一句玩笑不能当真，但硬件层面的优化确实是最直接、最高效的提升途径之一。向量化执行就是这种方式的典型代表，这项寄存器硬件层面的特性，为上层应用程序的性能带来了指数级的提升。

向量化执行，可以简单地看作一项消除程序中循环的优化。这里用一个形象的例子比喻。小胡经营了一家果汁店，虽然店里的鲜榨苹果汁深受大家喜爱，但客户总是抱怨制作果汁的速度太慢。小胡的店里只有一台榨汁机，每次他都会从篮子里拿出一个苹果，放到榨汁机内等待出汁。如果有 8 个客户，每个客户都点了一杯苹果汁，那么小胡需要重复循环 8 次上述的榨汁流程，才能榨出 8 杯苹果汁。如果制作一杯果汁需要 5 分钟，那么全部制作完毕则需要 40 分钟。为了提升果汁的制作速度，小胡想出了一个办法。他将榨汁机的数量从 1 台增加到了 8 台，这么一来，他就可以从篮子里一次性拿出 8 个苹果，分别放入 8 台榨汁机同时榨汁。此时，小胡只需要 5 分钟就能够制作出 8 杯苹果汁。为了制作 n 杯果汁，非向量化执行的方式是用 1 台榨汁机重复循环制作 n 次，而向量化执行的方式是用 n 台榨汁机只执行 1 次。

为了实现向量化执行，需要利用 CPU 的 SIMD 指令。SIMD 的全称是 Single Instruction Multiple Data，即用单条指令操作多条数据。现代计算机系统概念中，它是通过数据并行以提高性能的一种实现方式 (其他的还有指令级并行和线程级并行)，它的原理是在 CPU 寄存器层面实现数据的并行操作。

在计算机系统的体系结构中，存储系统是一种层次结构。典型服务器计算机的存储层次结构如图 1 所示。一个实用的经验告诉我们，存储媒介距离 CPU 越近，则访问数据的速度越快。

![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/FA0B559E064230B7E27C3AC15D2D1473.jpg)

从上图中可以看到，从左向右，距离 CPU 越远，则数据的访问速度越慢。从寄存器中访问数据的速度，是从内存访问数据速度的 300 倍，是从磁盘中访问数据速度的 3000 万倍。所以利用 CPU 向量化执行的特性，对于程序的性能提升意义非凡。

ClickHouse 目前利用 SSE4.2 指令集实现向量化执行。

**关系模型与 SQL 查询**

相比 HBase 和 Redis 这类 NoSQL 数据库，ClickHouse 使用关系模型描述数据并提供了传统数据库的概念 (数据库、表、视图和函数等)。与此同时，ClickHouse 完全使用 SQL 作为查询语言 ( 支持 GROUP BY、ORDER BY、JOIN、IN 等大部分标准 SQL )，这使得它平易近人，容易理解和学习。因为关系型数据库和 SQL 语言，可以说是软件领域发展至今应用最为广泛的技术之一，拥有极高的 "群众基础"。也正因为 ClickHouse 提供了标准协议的 SQL 查询接口，使得现有的第三方分析可视化系统可以轻松与它集成对接。在 SQL 解析方面，ClickHouse 是大小写敏感的，这意味着 SELECT a 和 SELECT A 所代表的语义是不同的。

关系模型相比文档和键值对等其他模型，拥有更好的描述能力，也能够更加清晰地表述实体间的关系。更重要的是，在 OLAP 领域，已有的大量数据建模工作都是基于关系模型展开的 (星型模型、雪花模型乃至宽表模型)。ClickHouse 使用了关系模型，所以将构建在传统关系型数据库或数据仓库之上的系统迁移到 ClickHouse 的成本会变得更低，可以直接沿用之前的经验成果。

**多样化的表引擎**

也许因为 Yandex.Metrica 的最初架构是基于 MySQL 实现的，所以在 ClickHouse 的设计中，能够察觉到一些 MySQL 的影子，表引擎的设计就是其中之一。与 MySQL 类似，ClickHouse 也将存储部分进行了抽象，把存储引擎作为一层独立的接口。截至本书完稿时，ClickHouse 共拥有合并树、内存、文件、接口和其他 6 大类 20 多种表引擎。其中每一种表引擎都有着各自的特点，用户可以根据实际业务场景的要求，选择合适的表引擎使用。

通常而言，一个通用系统意味着更广泛的适用性，能够适应更多的场景。但通用的另一种解释是平庸，因为它无法在所有场景内都做到极致。

在软件的世界中，并不会存在一个能够适用任何场景的通用系统，为了突出某项特性，势必会在别处有所取舍。其实世间万物都遵循着这样的道理，就像信天翁和蜂鸟，虽然都属于鸟类，但它们各自的特点却铸就了完全不同的体貌特征。信天翁擅长远距离飞行，环绕地球一周只需要 1 至 2 个月的时间。因为它能够长时间处于滑行状态，5 天才需要扇动一次翅膀，心率能够保持在每分钟 100 至 200 次之间。而蜂鸟能够垂直悬停飞行，每秒可以挥动翅膀 70～100 次，飞行时的心率能够达到每分钟 1000 次。如果用数据库的场景类比信天翁和蜂鸟的特点，那么信天翁代表的可能是使用普通硬件就能实现高性能的设计思路，数据按粗粒度处理，通过批处理的方式执行；而蜂鸟代表的可能是按细粒度处理数据的设计思路，需要高性能硬件的支持。

将表引擎独立设计的好处是显而易见的，通过特定的表引擎支撑特定的场景，十分灵活。对于简单的场景，可直接使用简单的引擎降低成本，而复杂的场景也有合适的选择。

**多线程与分布式**

ClickHouse 几乎具备现代化高性能数据库的所有典型特征，对于可以提升性能的手段可谓是一一用尽，对于多线程和分布式这类被广泛使用的技术，自然更是不在话下。

如果说向量化执行是通过数据级并行的方式提升了性能，那么多线程处理就是通过线程级并行的方式实现了性能的提升。相比基于底层硬件实现的向量化执行 SIMD，线程级并行通常由更高层次的软件层面控制。现代计算机系统早已普及了多处理器架构，所以现今市面上的服务器都具备良好的多核心多线程处理能力。由于 SIMD 不适合用于带有较多分支判断的场景，ClickHouse 也大量使用了多线程技术以实现提速，以此和向量化执行形成互补。

如果一个篮子装不下所有的鸡蛋，那么就多用几个篮子来装，这就是分布式设计中分而治之的基本思想。同理，如果一台服务器性能吃紧，那么就利用多台服务的资源协同处理。为了实现这一目标，首先需要在数据层面实现数据的分布式。因为在分布式领域，存在一条金科玉律—计算移动比数据移动更加划算。在各服务器之间，通过网络传输数据的成本是高昂的，所以相比移动数据，更为聪明的做法是预先将数据分布到各台服务器，将数据的计算查询直接下推到数据所在的服务器。ClickHouse 在数据存取方面，既支持分区 (纵向扩展，利用多线程原理)，也支持分片 ( 横向扩展，利用分布式原理 )，可以说是将多线程和分布式的技术应用到了极致。

**多主架构**

HDFS、Spark、HBase 和 Elasticsearch 这类分布式系统，都采用了 Master-Slave 主从架构，由一个管控节点作为 Leader 统筹全局。而 ClickHouse 则采用 Multi-Master 多主架构，集群中的每个节点角色对等，客户端访问任意一个节点都能得到相同的效果。这种多主的架构有许多优势，例如对等的角色使系统架构变得更加简单，不用再区分主控节点、数据节点和计算节点，集群中的所有节点功能相同。所以它天然规避了单点故障的问题，非常适合用于多数据中心、异地多活的场景。

**在线查询**

ClickHouse 经常会被拿来与其他的分析型数据库作对比，比如 Vertica、SparkSQL、Hive 和 Elasticsearch 等，它与这些数据库确实存在许多相似之处。例如，它们都可以支撑海量数据的查询场景，都拥有分布式架构，都支持列存、数据分片、计算下推等特性。这其实也侧面说明了 ClickHouse 在设计上确实吸取了各路奇技淫巧。与其他数据库相比，ClickHouse 也拥有明显的优势。例如，Vertica 这类商用软件价格高昂；SparkSQL 与 Hive 这类系统无法保障 90% 的查询在 1 秒内返回，在大数据量下的复杂查询可能会需要分钟级的响应时间；而 Elasticsearch 这类搜索引擎在处理亿级数据聚合查询时则显得捉襟见肘。

正如 ClickHouse 的 "广告词" 所言，其他的开源系统太慢，商用的系统太贵，只有 Clickouse 在成本与性能之间做到了良好平衡，即又快又开源。ClickHouse 当之无愧地阐释了 "在线" 二字的含义，即便是在复杂查询的场景下，它也能够做到极快响应，且无须对数据进行任何预处理加工。

**数据分片与分布式查询**

数据分片是将数据进行横向切分，这是一种在面对海量数据的场景下，解决存储和查询瓶颈的有效手段，是一种分治思想的体现。ClickHouse 支持分片，而分片则依赖集群。每个集群由 1 到多个分片组成，而每个分片则对应了 ClickHouse 的 1 个服务节点。分片的数量上限取决于节点数量 (1 个分片只能对应 1 个服务节点)。

ClickHouse 并不像其他分布式系统那样，拥有高度自动化的分片功能。ClickHouse 提供了本地表 (Local Table) 与分布式表 ( Distributed Table ) 的概念。一张本地表等同于一份数据的分片。而分布式表本身不存储任何数据，它是本地表的访问代理，其作用类似分库中间件。借助分布式表，能够代理访问多个数据分片，从而实现分布式查询。

这种设计类似数据库的分库和分表，十分灵活。例如在业务系统上线的初期，数据体量并不高，此时数据表并不需要多个分片。所以使用单个节点的本地表 (单个数据分片) 即可满足业务需求，待到业务增长、数据量增大的时候，再通过新增数据分片的方式分流数据，并通过分布式表实现分布式查询。这就好比一辆手动挡赛车，它将所有的选择权都交到了使用者的手中。

#### 简单入门

关于 ClickHouse 的安装，我们在这里不再详细展开，官网有详细的文档可以参考。

我们使用 Java 客户端连接 ClickHouse 进行一些简单的操作。首先 Clickhouse 有两种 JDBC 驱动实现：

一种是官方给出的

<dependency>  
    <groupId>ru.yandex.clickhouse</groupId>  
    <artifactId>clickhouse-jdbc</artifactId>  
    <version>0.2.4</version>  
</dependency>  

还有一种第三方提供的驱动：

<dependency>  
    <groupId>com.github.housepower</groupId>  
    <artifactId>clickhouse-native-jdbc</artifactId>  
    <version>2.5.2</version>  
</dependency>  

两者间的主要区别如下：

驱动类加载路径不同，分别为 ru.yandex.clickhouse.ClickHouseDriver 和 com.github.housepower.jdbc.ClickHouseDriver  
默认连接端口不同，分别为 8123 和 9000  
连接协议不同，官方驱动使用 HTTP 协议，而三方驱动使用 TCP 协议  

需要注意的是，两种驱动不可共用，同个项目中只能选择其中一种驱动。

Class.forName("com.github.housepower.jdbc.ClickHouseDriver");  
Connection connection = DriverManager.getConnection("jdbc:clickhouse://192.168.60.131:9000");

Statement statement = connection.createStatement();  
statement.executeQuery("create table test.example(day Date, name String, age UInt8) Engine=Log");

PreparedStatement pstmt = connection.prepareStatement("insert into test.example values(?, ?, ?)");

// insert 10 records  
for (int i = 0; i &lt; 10; i++) {  
    pstmt.setDate(1, new Date(System.currentTimeMillis()));  
    pstmt.setString(2, "panda\_" + (i + 1));  
    pstmt.setInt(3, 18);  
    pstmt.addBatch();  
}  
pstmt.executeBatch();

Statement statement = connection.createStatement();

String sql = "select \* from test.jdbc_example";  
ResultSet rs = statement.executeQuery(sql);

while (rs.next()) {  
    // ResultSet 的下标值从 1 开始，不可使用 0，否则越界，报 ArrayIndexOutOfBoundsException 异常  
    System.out.println(rs.getDate(1) + "," + rs.getString(2) + "," + rs.getInt(3));  
}

通过 clickhouse-client 命令行界面查看表情况：

ck-master :) show tables;

SHOW TABLES

┌─name─────────┐  
│ hits         │  
│ jdbc_example │  
└──────────────┘

ck-master :) select \* from example;

SELECT \*  
FROM jdbc_example

┌────────day─┬─name─────┬─age─┐  
│ 2019-04-25 │ panda_1  │  18 │  
│ 2019-04-25 │ panda_2  │  18 │  
│ 2019-04-25 │ panda_3  │  18 │  
│ 2019-04-25 │ panda_4  │  18 │  
│ 2019-04-25 │ panda_5  │  18 │  
│ 2019-04-25 │ panda_6  │  18 │  
│ 2019-04-25 │ panda_7  │  18 │  
│ 2019-04-25 │ panda_8  │  18 │  
│ 2019-04-25 │ panda_9  │  18 │  
│ 2019-04-25 │ panda_10 │  18 │  
└────────────┴──────────┴─────┘

#### 企业级应用

从 2019 年起，已经有很多大厂开始在生产环境使用 ClickHouse，小编在这里根据各大公司使用情况作了一些总结，适用场景、方案、优化等等。

**ClickHouse 在携程酒店数据智能平台的应用**

下图是携程实际应用 ClickHouse 的架构图，底层数据大部分是离线的，一部分是实时的，离线数据现在大概有将近 3000 多个 job 每天都是把数据从 HIVE 拉到 ClickHouse 里面去，实时数据主要是接外部数据，然后批量写到 ClickHouse 里面。数据智能平台 80% 以上的数据都在 ClickHouse 上面。

![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/D83BDD6C6BC20075942B49AB7317E7C2.jpg)

同时，携程还存在全量数据同步和增量数据同步的场景。

全量数据同步的流程流程如下图：

-   清空 A_temp 表，将最新的数据从 Hive 通过 ETL 导入到 A_temp 表
-   将 A rename 成 A_temp_temp
-   将 A_temp rename 成 A
-   将 A_temp_temp rename 成 A_tem

![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/ACDFA420AB12FD69BD03EFE9AE229BCC.jpg)

需要注意的是，ClickHouse 每执行一个 insert 的时候会产生一个进程 ID，如果没有执行完，直接 Rename 就会造成数据丢失，数据就不对了，所以必须要有一个 job 在轮询看这边是不是执行完了，只有当 insert 的进程 id 执行完成后再做后面一系列的 rename。

增量的数据同步流程入下图所示：

-   清空 A_temp 表，将最近 3 个月的数据从 Hive 通过 ETL 导入到 A_temp 表
-   将 A 表中 3 个月之前的数据 select into 到 A_temp 表
-   将 A rename 成 A_temp_temp
-   将 A_temp rename 成 A
-   将 A_temp_temp rename 成 A_tem

![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/EFE47E464AE042801C231E728EB3AA3A.jpg)

以下是携程使用 ClickHouse 的经验：

1、数据导入之前要评估好分区字段

ClickHouse 因为是根据分区文件存储的，如果说你的分区字段真实数据粒度很细，数据导入的时候就会把你的物理机打爆。其实数据量可能没有多少，但是因为你用的字段不合理，会产生大量的碎片文件，磁盘空间就会打到底。

2、数据导入提前根据分区做好排序，避免同时写入过多分区导致 clickhouse 内部来不及 Merge

数据导入之前我们做好排序，这样可以降低数据导入后 ClickHouse 后台异步 Merge 的时候涉及到的分区数，肯定是涉及到的分区数越少服务器压力也会越小。

3、左右表 join 的时候要注意数据量的变化

再就是左右表 join 的问题，ClickHouse 它必须要大表在左边，小表在右边。但是我们可能某些业务场景跑着跑着数据量会返过来了，这个时候我们需要有监控能及时发现并修改这个 join 关系。

4、根据数据量以及应用场景评估是否采用分布式

分布式要根据应用场景来，如果你的应用场景向上汇总后数据量已经超过了单物理机的存储或者 CPU / 内存瓶颈而不得不采用分布式 ClickHouse 也有很完善的 MPP 架构，但同时你也要维护好你的主 keyboard。

5、监控好服务器的 CPU / 内存波动

再就是做好监控，我前面说过 ClickHouse 的 CPU 拉到 60% 的时候，基本上你的慢查询马上就出来了，所以我这边是有对 CPU 和内存的波动进行监控的，类似于 dump，这个我们抓下来以后就可以做分析。

6、数据存储磁盘尽量采用 SSD

数据存储尽量用 SSD，因为我之前也开始用过机械硬盘，机械硬盘有一个问题就是当你的服务器要运维以后需要重启，这个时候数据要加载，我们现在单机数据量存储有超过了 200 亿以上，这还是我几个月前统计的。这个数据量如果说用机械硬盘的话，重启一次可能要等上好几个小时服务器才可用，所以尽量用 SSD，重启速度会快很多。

当然重启也有一个问题就是说会导致你的数据合并出现错乱，这是一个坑。所以我每次维护机器的时候，同一个集群我不会同时维护几台机器，我只会一台一台维护，A 机器好了以后会跟它的备用机器对比数据，否则机器起来了，但是数据不一定是对的，并且可能是一大片数据都是不对的。

7、减少数据中文本信息的冗余存储

要减少一些中文信息的冗余存储，因为中文信息会导致整个服务器的 IO 很高，特别是导数据的时候。

8、特别适用于数据量大，查询频次可控的场景，如数据分析、埋点日志系统

对于它的应用，我认为从成本角度来说，就像以前我们有很多业务数据的修改日志，大家开发的时候可能都习惯性的存到 MySQL 里面，但是实际上我认为这种数据非常适合于落到 ClickHouse 里面，比落到 MySQL 里面成本会更低，查询速度会更快。

**Clickhouse 在快手的应用**

![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/53349BEAF475769C248B7C781052048E.jpg)
![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/977DA0696F1C99058D6A8616119CBC71.jpg)
![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/FC4E90DBFA49A2E3A5E355D4591BD0E6.jpg)
![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/3422678357B8E0D98294E80AA38A653D.jpg)
![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/BF4650E0E7E4F3221E5E1F57FBBF4BC5.jpg)
![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/BC8D3884543B36C23453DEF173C322A6.jpg)
![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/464EF3905F36580AC00F7BD15C00B43B.jpg)

**Clickhouse 在 QQ 的应用**

QQ 音乐大数据团队基于 ClickHouse+Superset 等基础组件，结合腾讯云 EMR 产品的云端能力，搭建起高可用、低延迟的实时 OLAP 分析计算可视化平台。

集群日均新增万亿数据，规模达到上万核 CPU，PB 级数据量。整体实现秒级的实时数据分析、提取、下钻、监控数据基础服务，大大提高了大数据分析与处理的工作效率。

![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/2CF3FB057FB6EDD097BC7DB7761CF7EC.jpg)

通过 OLAP 分析平台，极大降低了探索数据的门槛，做到全民 BI，全民数据服务，实现了实时 PV、UV、营收、用户圈层、热门歌曲等各类指标高效分析，全链路数据秒级分析定位，加强数据上报规范，形成一个良好的正循环。

面对上万核集群规模、PB 级的数据量，经过 QQ 音乐大数据团队和腾讯云 EMR 双方技术团队无数次技术架构升级优化，性能优化，逐步形成高可用、高性能、高安全的 OLAP 计算分析平台。

（1）基于 SSD 盘的 ZooKeeper

ClickHouse 依赖于 ZooKeeper 实现分布式系统的协调工作，在 ClickHouse 并发写入量较大时，ZooKeeper 对元数据存储处理不及时，会导致 ClickHouse 副本间同步出现延迟，降低集群整体性能。

解决方案：采用 SSD 盘的 ZooKeeper 大幅提高 IO 的性能，在表个数小于 100，数据量级在 TB 级别时，也可采用 HDD 盘，其他情况都建议采用 SSD 盘。

![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/C179225747ACBD932144D5BC5620B01E.jpg)

（2）数据写入一致性

数据在写入 ClickHouse 失败重试后内容出现重复，导致了不同系统，如 Hive 离线数仓中分析结果，与 ClickHouse 集群中运算结果不一致。

![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/DD58A555D1394D1B5433B6CD31D35593.jpg)

解决方案：基于统一全局的负载均衡调度策略，完成数据失败后仍然可写入同一 Shard，实现数据幂等写入，从而保证在 ClickHouse 中数据一致性。

（3）实时离线数据写入

ClickHouse 数据主要来自实时流水上报数据和离线数据中间分析结果数据，如何在架构中完成上万亿基本数据的高效安全写入，是一个巨大的挑战。

解决方案：基于 Tube 消息队列，完成统一数据的分发消费，基于上述的一致性策略实现数据幂同步，做到实时和离线数据的高效写入。

![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/37A470EBEEC101F076DD037E6215FA19.jpg)

（4）表分区数优化

部分离线数据仓库采用按小时落地分区，如果采用原始的小时分区更新同步，会造成 ClickHouse 中 Select 查询打开大量文件及文件描述符，进而导致性能低下。

解决方案：ClickHouse 官方也建议，表分区的数量建议不超过 10000，上述的数据同步架构完成小时分区转换为天分区，同时程序中完成数据幂等消费。

（5）读 / 写分离架构

频繁的写动作，会消耗大量 CPU / 内存 / 网卡资源，后台合并线程得不到有效资源，降低 Merge Parts 速度，MergeTree 构建不及时，进而影响读取效率，导致集群性能降低。

解决方案：ClickHouse 临时节点预先完成数据分区文件构建，动态加载到线上服务集群，缓解 ClickHouse 在大量并发写场景下的性能问题，实现高效的读 / 写分离架构，具体步骤和架构如下：

a）利用 K8S 的弹性构建部署能力，构建临时 ClickHouse 节点，数据写入该节点完成数据的 Merge、排序等构建工作；

b）构建完成数据按 MergeTree 结构关联至正式业务集群。

当然对一些小数据量的同步写入，可采用 10000 条以上批量的写入。

![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/08C174C07E42AB1C3A773FB03B4B3FDD.jpg)

（6）跨表查询本地化

在 ClickHouse 集群中跨表进行 Select 查询时，采用 Global IN/Global Join 语句性能较为低下。分析原因，是在此类操作会生成临时表，并跨设备同步该表，导致查询速度慢。

解决方案：采用一致性 hash，将相同主键数据写入同一个数据分片，在本地 local 表完成跨表联合查询，数据均来自于本地存储，从而提高查询速度。

这种优化方案也有一定的潜在问题，目前 ClickHouse 尚不提供数据的 Reshard 能力，当 Shard 所存储主键数据量持续增加，达到磁盘容量上限需要分拆时，目前只能根据原始数据再次重建 CK 集群，有较高的成本。

![](https://image.z.itpub.net/zitpub.net/JPG/2021-04-23/4EC0F77DB8EEA84705BAE8C6E9B378DA.jpg)

ClickHouse 从进入大众视野到开始广泛应用实践并不久，ClickHouse 社区也在飞速发展中。ClickHouse 仍然年轻，虽然在某些方面存在不足，但极致性能的存储引擎，使得 ClickHouse 成为一个非常优秀的存储底座。 
 [https://z.itpub.net/article/detail/7486371E68AF3E686A20BE30859E27EA](https://z.itpub.net/article/detail/7486371E68AF3E686A20BE30859E27EA)
