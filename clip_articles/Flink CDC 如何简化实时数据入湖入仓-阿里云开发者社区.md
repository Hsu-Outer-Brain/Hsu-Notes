# Flink CDC 如何简化实时数据入湖入仓-阿里云开发者社区
## 一、Flink CDC 介绍

从广义的概念上讲，能够捕获数据变更的技术, 我们都可以称为 CDC 技术。通常我们说的 CDC 技术是一种用于捕获数据库中数据变更的技术。CDC 技术应用场景也非常广泛，包括：

● **数据分发**，将一个数据源分发给多个下游，常用于业务解耦、微服务。

● **数据集成**，将分散异构的数据源集成到数据仓库中，消除数据孤岛，便于后续的分析。

● **数据迁移**，常用于数据库备份、容灾等。

 ![](https://ucc.alicdn.com/pic/developer-ecology/1f200de08ebf4adb8276246931128b9d.png) 

Flink CDC 基于数据库日志的 Change Data Caputre 技术，实现了全量和增量的一体化读取能力，并借助 Flink 优秀的管道能力和丰富的上下游生态，支持捕获多种数据库的变更，并将这些变更实时同步到下游存储。

目前，Flink CDC 的上游已经支持了 MySQL、MariaDB、PG、Oracle、MongoDB 等丰富的数据源，对 Oceanbase、TiDB、SQLServer 等数据库的支持也已经在社区的规划中。

Flink CDC 的下游则更加丰富，支持写入 Kafka、Pulsar 消息队列，也支持写入 Hudi、Iceberg 等数据湖，还支持写入各种数据仓库。

同时，通过 Flink SQL 原生支持的 Changelog 机制，可以让 CDC 数据的加工变得非常简单。用户可以通过 SQL 便能实现数据库全量和增量数据的清洗、打宽、聚合等操作，极大地降低了用户门槛。 此外， Flink DataStream API 支持用户编写代码实现自定义逻辑，给用户提供了深度定制业务的自由度。

 ![](https://ucc.alicdn.com/pic/developer-ecology/ea09874996034e4d90357b60dc1c0cc2.png) 

Flink CDC 技术的核心是支持将表中的全量数据和增量数据做实时一致性的同步与加工，让用户可以方便地获每张表的实时一致性快照。比如一张表中有历史的全量业务数据，也有增量的业务数据在源源不断写入，更新。Flink CDC 会实时抓取增量的更新记录，实时提供与数据库中一致性的快照，如果是更新记录，会更新已有数据。如果是插入记录，则会追加到已有数据，整个过程中，Flink CDC 提供了一致性保障，即不重不丢。

那么 Flink CDC 技术能给现有的数据入仓入湖架构带来什么样的改变呢？我们可以先来看看传统数据入仓的架构。

 ![](https://ucc.alicdn.com/pic/developer-ecology/d4e25da99a764bbeae2ab6dd35a02009.png) 

在早期的数据入仓架构中，一般会每天 SELECT 全量数据导入数仓后再做离线分析。这种架构有几个明显的缺点：

(1) 每天查询全量的业务表会影响业务自身稳定性。

(2) 离线天级别调度的方式，天级别的产出时效性差。

(3) 基于查询方式，随着数据量的不断增长，对数据库的压力也会不断增加，架构性能    瓶颈明显。

 ![](https://ucc.alicdn.com/pic/developer-ecology/101b8f47a4134e29a1b16388fcc574f1.png) 

到了数据仓库的 2.0 时代，数据入仓进化到了 Lambda 架构，增加了实时同步导入增量的链路。整体来说，Lambda 架构的扩展性更好，也不再影响业务的稳定性，但仍然存在一些问题：

(1) 依赖离线的定时合并，只能做到小时级产出，延时还是较大；

(2) 全量和增量是割裂的两条链路；

(3) 整个架构链路长，需要维护的组件比较多，该架构的全量链路需要维护 DataX 或 Sqoop 组件，增量链路要维护 Canal 和 Kafka 组件，同时还要维护全量和增量的定时合并链路。

 ![](https://ucc.alicdn.com/pic/developer-ecology/7819699a48f5496a9a49dd1236684670.png) 

对于传统数据入仓架构存在的问题，Flink CDC 的出现为数据入湖架构提供了一些新思路。借助 Flink CDC 技术的全增量一体化实时同步能力，结合数据湖提供的更新能力，整个架构变得非常简洁。我们可以直接使用 Flink CDC 读取 MySQL 的全量和增量数据，并直接写入和更新到 Hudi 中。

这种简洁的架构有着明显的优势。首先，不会影响业务稳定性。其次，提供分钟级产出，满足近实时业务的需求。同时，全量和增量的链路完成了统一，实现了一体化同步。最后，该架构的链路更短，需要维护的组件更少。

## 二、Flink CDC 的核心特性

 ![](https://ucc.alicdn.com/pic/developer-ecology/921c657bd3d44de0b125d8a1ae9d5519.png) 

Flink CDC 的核心特性可以分成四个部分：一是通过增量快照读取算法，实现了无锁读取，并发读取，断点续传等功能。二是设计上对入湖友好，提升了 CDC 数据入湖的稳定性。三是支持异构数据源的融合，能方便地做 Streaming ETL 的加工。四是支持分库分表合并入湖。接下来我们会分别介绍下这几个特性。

  ![](https://ucc.alicdn.com/pic/developer-ecology/1d6d3ab89a194ae9ace4009ecc130947.png) 

在 Flink CDC 1.x 版本时，MySQL CDC 存在三大痛点，影响了生产可用性。一是 MySQL CDC 需要通过全局锁去保证全量和增量数据的一致性，而 MySQL 的全局锁会影响线上业务。二是只支持单并发读取，大表读取非常耗时。三是在全量同步阶段，作业失败后只能重新同步，稳定性较差。针对这些问题，Flink CDC 社区提出了 “增量快照读取算法”，同时实现了无锁读取、并行读取、断点续传等能力，一并解决了上述痛点。

  ![](https://ucc.alicdn.com/pic/developer-ecology/50d390610e7c4410a7c1c0918217fa8c.png) 

简单来说，增量快照读取算法的核心思路就是在全量读取阶段把表分成一个个 chunk 进行并发读取，在进入增量阶段后只需要一个 task 进行单并发读取 binlog 日志，在全量和增量自动切换时，通过无锁算法保障一致性。这种设计在提高读取效率的同时，进一步节约了资源。实现了全增量一体化的数据同步。这也是流批一体道路上一个非常重要的落地。

  ![](https://ucc.alicdn.com/pic/developer-ecology/6740a9cf96e145b7a669fcff609ef82c.png) 

Flink CDC 是一个流式入湖友好的框架。在早期版本的 Flink CDC 设计中，没有考虑数据湖场景，全量阶段不支持 Checkpoint，全量数据会在一个 Checkpoint 中处理，这对依靠 Checkpoint 提交数据的数据湖很不友好。Flink CDC 2.0 设计之初考虑了数据湖场景，是一种流式入湖友好的设计。设计上将全量数据进行分片，Flink CDC 可以将 checkpoint 粒度从表粒度优化到 chunk 粒度，大大减少了数据湖写入时的 Buffer 使用，对数据湖写入更加友好。

  ![](https://ucc.alicdn.com/pic/developer-ecology/1ad03e094118406bbb6b923ab2cf555f.png) 

Flink CDC 区别于其他数据集成框架的一个核心点，就是在于 Flink 提供的流批一体计算能力。这使得 Flink CDC 成为了一个完整的 ETL 工具，不仅仅拥有出色的 E 和 L 的能力，还拥有强大的 Transformation 能力。因此我们可以轻松实现基于异构数据源的数据湖构建。

在上图左侧的 SQL 中，我们可以将 MySQL 中的实时产品表、实时订单表和 PostgreSQL 中的实时物流信息表进行实时关联，即 Streaming Join，关联后的结果实时更新到 Hudi 中，非常轻松地完成异构数据源的数据湖构建。

 ![](https://ucc.alicdn.com/pic/developer-ecology/0c1ea12c41fe4d1b9b8d57f814632711.png) 

在 OLTP 系统中，为了解决单表数据量大的问题，通常采用分库分表的方式将单个大表进行拆分以提高系统的吞吐量。但是为了方便数据分析，通常需要将分库分表拆分出的表在同步到数据仓库、数据湖时，再合并成一个大表。Flink CDC 可以轻松完成这个任务。

在上图左侧的 SQL 中，我们声明了一张 user_source 表去捕获所有 user 分库分表的数据，我们通过表的配置项 database-name、table-name 使用正则表达式来匹配这些表。并且，user_source 表也定义了两个 metadata 列来区分数据是来自哪个库和表。在 Hudi 表的声明中，我们将库名、表名和原表的主键声明成 Hudi 中的联合主键。在声明完两张表后，一条简单的 INSERT INTO 语句就可以将所有分库分表的数据合并写入 Hudi 的一张表中，完成基于分库分表的数据湖构建，方便后续在湖上的统一分析。

## 三、Flink CDC 的开源生态

 ![](https://ucc.alicdn.com/pic/developer-ecology/031e3d1e593f4a318d156feb5dba8576.png) 

Flink CDC 是一个独立的开源项目，项目代码托管在 GitHub 上（[https://github.com/ververica/flink-cdc-connectors](https://github.com/ververica/flink-cdc-connectors)）。采取小步快跑的发布节奏，今年社区已经发布了 5 个版本。 1.x 系列的三个版本推出了一些小功能； 2.0 版本 MySQL CDC 支持了无锁读取、并发读取、断点续传等高级功能，commits 达到了 91 个，贡献者达到了 15 人；2.1 版本则支持了 Oracle、MongoDB 数据库，commits 达到了 115 个，贡献者达到了 28 人。社区的 commits 和 贡献者增长非常明显。

文档和帮助手册也是开源社区非常重要的一部分，为了更好地帮助用户，Flink CDC 社区推出了版本化的文档网站，如 2.1 版本的文档：[https://ververica.github.io/flink-cdc-connectors/release-2.1/](https://ververica.github.io/flink-cdc-connectors/release-2.1/)。文档中还提供了很多快速入门的教程，用户只要有个 Docker 环境就能上手 Flink CDC。此外，还提供了 FAQ 指导手册[https://github.com/ververica/flink-cdc-connectors/wiki/FAQ(ZH)](https://github.com/ververica/flink-cdc-connectors/wiki/FAQ(ZH))，快速解决用户遇到的常见问题。

 ![](https://ucc.alicdn.com/pic/developer-ecology/2cfa6655650d49ddaf5a9a75a5f4cab0.png) 

在过去的 2021 年，Flink CDC 社区取得了迅速的发展，GitHub 的 PR 和 issue 相当活跃，GitHub Star 更是年度同比增长 330%。

## 四、Flink CDC 在阿里巴巴的实践和改进

Flink CDC 入湖入仓在阿里巴巴也有大规模的实践和落地，过程中也遇到了一些痛点和挑战。我们会介绍下我们是如何改进和解决的。

 ![](https://ucc.alicdn.com/pic/developer-ecology/c782b65ddeea4b4bb715900221fd59fd.png) 

我们先来看下 CDC 入湖遇到的一些痛点和挑战。这是某个用户原有的 CDC 数据入湖架构，分为两个链路，有一个全量同步作业做一次性的全量数据拉取，还有一个增量作业通过 Canal 和处理引擎将 Binlog 数据准实时地同步到 Hudi 表中。这个架构虽然利用了 Hudi 的更新能力，无需周期性地调度全量合并任务，能做到分钟级延迟。但是全量和增量仍是割裂的两个作业，全量和增量的切换仍需要人工的介入，并且需要指定一个准确的增量启动位点，否则的话就会有丢失数据的风险。可以看到这种架构是流批割裂的，并不是一个统一的整体。刚刚雪尽也介绍了 Flink CDC 最大的一个优势之一就是全增量的自动切换，所以我们用 Flink CDC 替换了用户原有的入湖架构。

 ![](https://ucc.alicdn.com/pic/developer-ecology/fededccdfe0a44d8894068d86fae9331.png) 

但是用户用了 Flink CDC 后，遇到的第一个痛点就是需要将 MySQL 的 DDL 手工映射成 Flink 的 DDL。手工映射表结构是比较繁琐的，尤其是当表和字段数非常多的时候。而且手工映射也容易出错，比如 说 MySQL 的 BIGINT UNSINGED，它不能映射成 Flink 的 BIGINT，而是要映射成 DECIMAL(20)。 如果系统能自动帮助用户自动去映射表结构就会简单安全很多。

  ![](https://ucc.alicdn.com/pic/developer-ecology/c9316303a50a430bb9e15dcd14be6763.png) 

用户遇到的另一个痛点是表结构的变更导致入湖链路难以维护。例如用户有一张表，原先有 id 和 name 两列，突然增加了一列 Address。新增的这一列数据可能就无法同步到数据湖中，甚至导致入湖链路的挂掉，影响稳定性。除了加列的变更，还可能会有删列、类型变更等等。国外的 Fivetran 做过一个[调研报告](https://fivetran.com/blog/analyst-survey)，发现 60% 的公司，schema 每个月都会变化，30% 每周都会变化。这说明基本每个公司都会面临 schema 变更带来的数据集成上的挑战。

 ![](https://ucc.alicdn.com/pic/developer-ecology/e313b95c05474f6fb628e5d2bef1ca4f.png) 

最后一个是整库入湖的挑战。因为用户主要使用 SQL，这就需要为每个表的数据同步链路定义一个 INSERT INTO 语句。有些用户的 MySQL 实例中甚至有上千张的业务表，用户就要写上千个 INSERT INTO 语句。更令人望而生却的是，每一个 INSERT INTO 任务都会创建至少一个数据库连接，读取一次 Binlog 数据。千表入湖的话就需要上千个连接，上千次的 Binlog 重复读取。这就会对 MySQL 和网络造成很大的压力。

  ![](https://ucc.alicdn.com/pic/developer-ecology/840c3df3a31948a5baa17fe85b997539.png) 

刚刚我们介绍了 cdc 数据入湖的很多痛点和挑战，我们可以站在用户的角度想一想，数据库入湖这个场景用户到底想要的是什么呢？我们可以先把中间的数据集成系统看成一个黑盒，用户会期望这个黑盒提供什么样的能力来简化入湖的工作呢？

  ![](https://ucc.alicdn.com/pic/developer-ecology/21863dbe07a54ccd96ae675083a6d561.png) 

首先，用户肯定想把数据库中全量和增量的数据都同步过去，这就需要这个系统具有全增量一体化、全增量自动切换的能力，而不是割裂的全量链路 + 增量链路。其次，用户肯定不想为每个表去手动映射 schema，这就需要系统具有元信息自动发现的能力，省去用户在 Flink 中创建 DDL 的过程，甚至帮用户自动在 Hudi 中创建目标表。另外，用户还希望源端表结构的变更也能自动同步过去，不管是加列减列和改列，还是加表减表和改表，都能够实时的自动的同步到目标端，从而不丢失任何在源端发生的新增数据，自动化地构建与源端数据库保持数据一致的 ODS 层。最后，还需要有具备生产可用的整库同步能力，不能对源端造成太大压力，影响在线业务。

这四个核心功能基本组成了用户理想中所期待的数据集成系统，而这一切如果只需要一行 SQL，一个 Job 就能完成的话，那就更完美了。我们把中间的这个系统称为 “全自动化数据集成”，因为它全自动地完成了数据库的入湖，解决了目前遇到的几个核心痛点。而且目前看来，Flink 是实现这一目标非常适合的引擎。

 ![](https://ucc.alicdn.com/pic/developer-ecology/9c8516f329d94d1d999567cb23867f14.png) 

所以我们花了很多精力，基于 Flink 去打造这个 “全自动化数据集成”。主要就是围绕刚刚说的这四点。首先 Flink CDC 已经具备了全增量自动切换的能力，这也是 Flink CDC 的亮点之一。在元信息的自动发现上，可以通过 Flink 的 Catalog 接口无缝对接上，我们开发了 MySQL Catalog 来自动发现 MySQL 中的表和 schema，还开发了 Hudi Catalog 自动地去 Hudi 中创建目标表的元信息。在表结构变更的自动同步方面，我们引入了一个 Schema Evolution 的内核，使得 Flink Job 无需依赖外部服务就能实时同步 schema 变更。在整库同步方面，我们引入了 CDAS 语法，一行 SQL 语句就能完成整库同步作业的定义，并且引入了 source 合并的优化，减轻对源端数据库的压力。

 ![](https://ucc.alicdn.com/pic/developer-ecology/da2c0dbcfcdb4e9e938fefcf272e4cd0.png) 

为了支持整库同步，我们还引入了 CDAS 和 CTAS 的数据同步语法。它的语法非常简单，CDAS 语法就是 create database as database，主要用于整库同步，像这里展示的这行语句就完成了从 MySQL 的 tpc_ds 库，整库同步至 Hudi 的 ods 库中。与之类似的，我们还有一个 CTAS 语法，可以方便的用来支持表级别的同步，还可以通过正则表达式指定库名和表名，来完成分库分表合并同步。像这里就完成了 MySQL 的 user 分库分表合并到了 Hudi 的 users 表中。CDAS CTAS 的语法，会自动地去目标端创建目标表，然后启动一个 Flink Job 自动同步全量 + 增量的数据，并且也会实时同步表结构变更。

 ![](https://ucc.alicdn.com/pic/developer-ecology/6f1c2b57f8b74b49a6efddfed632d599.png) 

之前提到千表入湖时，建立的数据库连接过多，Binlog 重复读取会造成源库的巨大压力。为了解决这个问题，我们引入了 source 合并的优化，我们会尝试合并同一作业中的 source，如果都是读的同一数据源，则会被合并成一个 source 节点，这时数据库只需要建立一个连接，binlog 也只需读取一次，实现了整库的读取，降低了对数据库的压力。  

为了更直观地了解我们是如何简化数据入湖入仓的工作，我们还额外提供了一个 Demo 视频，感兴趣的朋友可以在 Flink Forward Asia 2021 大会上，观看《Flink CDC 如何简化实时数据入湖入仓》的分享。

## 五、Flink CDC 的未来规划

 ![](https://ucc.alicdn.com/pic/developer-ecology/815dad4cf3ba458cb170996fcf386e53.png) 

最后关于 Flink CDC 的未来主要有三个方面的规划。第一，我们会继续完善 CDAS 和 CTAS 的语法和接口，打磨 Schema Evolution 的内核，为开源做准备。第二，我们会扩展更多的 CDC 数据源，包括 TiDB、OceanBase、SQLServer 这些都已经规划中了。第三，我们会将目前的增量快照读取算法抽象成通用框架，使得有更多的数据库能通过简单几个接口就对接到这个框架上，具备全增量一体化的能力。

也希望有更多的志同道合之士能加入到 Flink CDC 开源社区的建设和贡献中，一起打造新一代的数据集成框架！ 
 [https://developer.aliyun.com/article/852295](https://developer.aliyun.com/article/852295)
