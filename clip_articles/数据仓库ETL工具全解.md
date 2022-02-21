# 数据仓库ETL工具全解
傅一平评语：  

这篇文章比较全的介绍了传统 ETL 工具、新型 ETL 工具、主流计算引擎及流程控制引擎。

1、传统 ETL 工具包括 Datastage、Informatica PowerCenter、Kettle、ODI、Sqoop、DataX、Flume、Canal、DTS、GoldenGate、Maxwell、DSG 等等。

2、新型 ETL 工具包括 Streamsets、Waterdrop 等。

3、主流计算引擎包括 MapReduce、Tez、Spark、Flink、ClickHouse 、Doris 等等。

4、流程控制（也称工作流、任务流）是 ETL 重要的组成部分，主要包括 Hudson、Airflow、 Azkaban、Oozie、DolphinScheduler。  

如果要从 0 到 1 学习和引入，作者建议直接上最好的，比如 ETL 工具传统的那些就没必要学了，直接学 StreamSets 或者 WaterDrop 即可；实时计算直接学 Flink 即可不用看 Spark 了；众多的 OLAP 我们直接学 ClickHouse 或者 Doris 即可其它的也不用看了；调度嘛直接 DS 就好了。

## 0x00 前言

ETL 是数据仓库的重要组成部分，但 ETL 也可以独立存在的。本篇我会集中起来给大家介绍一些常用的 ETL 工具或者类 ETL 的集成、同步、计算、流程控制工具。

-   第一部分，主要介绍五种传统 ETL 工具和八种数据同步集成工具。
-   第二部分，主要介绍两种新型 ETL 工具和大数据发展不同阶段产生的六种主要计算引擎。

**第一部分**

## 0x01 传统 ETL 工具

### DataStage

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_2_20220112081354474)

_Datastage 操作界面_

-   对元数据的支持：Datastage 是自己管理 Metadata，不依赖任何数据库。
-   参数控制：Datastage 可以对每个 job 设定参数，并且可以 job 内部引用这个参数名。
-   数据质量：Datastage 有配套用的 ProfileStage 和 QualityStage 保证数据质量。
-   定制开发：提供抽取、转换插件的定制，Datastage 内嵌一种类 BASIC 语言，可以写一段批处理程序来增加灵活性。
-   修改维护：提供图形化界面。这样的好处是直观、傻瓜式的；不好的地方就是改动还是比较费事（特别是批量化的修改）。

Datastage 包含四大部件：Administrator、Manager、Designer、Director。

1.  用 DataStage Administrator 新建或者删除项目，设置项目的公共属性，比如权限。
2.  用 DataStage Designer 连接到指定的项目上进行 Job 的设计；
3.  用 DataStage Director 负责 Job 的运行，监控等。例如设置设计好的 Job 的调度时间。
4.  用 DataStage Manager 进行 Job 的备份等 Job 的管理工作。

### Informatica

Informatica PowerCenter 用于访问和集成几乎任何业务系统、任何格式的数据，它可以按任意速度在企业内交付数据，具有高性能、高可扩展性、高可用性的特点。它提供了一个可视化的、拥有丰富转换库的设计工具，这个转换库使数据转换变成一个简单的 “拖拽” 过程，用户不需在组件时编写脚本语言。可以通过简单的操作，完成需求。使用 PowerCenter，转换组件能够被合并到 mapping 对象中，独立于他们的数据源和目标，有近 20 种数据转换组件和近百个函数可以调用，同时可以调用外部的过程和程序，实现复杂的转化逻辑。

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_3_20220112081354739)

_Informatica 操作界面_

-   对元数据的支持：元数据相对开放，存放在关系数据中，可以很容易被访问。
-   参数控制：参数放在一个参数文件中，理论上的确可以灵活控制参数，但这个灵活性需要用户自己更新文件中的参数值（例如日期更新）。另外，Powercenter 不能在 mapping 或 session 中引用参数名。
-   数据质量：专门有一个产品 Informatica Data Quality 来保证数据质量。
-   定制开发：没有内嵌类 BASIC 语言，参数值需人为更新，且不能引用参数名。
-   修改维护：与 Datastage 相同，Powercenter 也提供图形化界面。这样的好处是直观、傻瓜式的；不好的地方就是改动还是比较费事。

Informatica 的开发分为六个步骤：

1.  定义源，就是定义我们源头数据在哪里。配置数据链接，比如 IP 账号密码等信息。
2.  定义目标，就是我们准备把数据放到哪里。这个是我们事先定义的数据仓库。
3.  创建映射，就是我们的元数据和目标数据的映射关系。
4.  定义任务，就是我们每个表的转换过程，可以同时处理多个表。
5.  创建工作流，将任务按照一定的顺序进行组合。
6.  工作流调度和监控，定时、自动或者手动方式触发工作流。

_有兴趣更详细了解的可以参考这篇文章：_

_[https://blog.csdn.net/water\\\_0815/article/details/76512470](https://blog.csdn.net/water\_0815/article/details/76512470)_

### Kettle

Pentaho Data Integration，是一款国外免费开源的、可视化的、功能强大的 ETL 工具。由于其开源、免费、跨平台、资料文档丰富等特点获得了一大批忠实粉丝。

Kettle 六大特点：

-   免费开源：基于 Java 免费开源软件。
-   易配置：可跨平台，绿色无需安装。
-   不同数据库：ETL 工具集，可管理不同数据库的数据。
-   两种脚本文件：transformation 和 job。transformation 完成针对数据的基础转换，job 则完成整个工作流的控制。
-   图形界面设计：托拉拽，无需写代码。
-   定时功能：在 Job 下的 start 模块，有一个定时功能，可以每日，每周等方式进行定时。

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_4_20220112081354974)

_Kettle 操作界面_

Kettle 的执行分为两个层次：Job 和 Transformation。这两个层次的最主要的区别在于数据的传递和运行方式。

-   Transformation：定义对数据操作的容器，数据操作就是数据从输入到输出的一个过程，可以理解为比 Job 粒度更小一级的容器，我们将任务分解成 Job，然后需要将 Job 分解成一个或多个 Transformation，每个 Transformation 只完成一部分工作。
-   Step：是 Transformation 内部的最小单元，每一个 Step 完成一个特定的功能。
-   Job：负责将 Transformation 组织在一起进而完成某一工作，通常我们需要把一个大的任务分解成几个逻辑上隔离的 Job，当这几个 Job 都完成了，也就说明这项任务完成了。
-   Job Entry：Job Entry 是 Job 内部的执行单元，每一个 Job Entry 用于实现特定的功能，如：验证表是否存在，发送邮件等。可以通过 Job 来执行另一个 Job 或者 Transformation，也就是说 Transformation 和 Job 都可以作为 Job Entry。
-   Hop：用于在 Transformation 中连接 Step，或者在 Job 中连接 Job Entry，是一个数据流的图形化表示。

在 Kettle 中 Job 的 JobEntry 是串行执行的，故 Job 中必须有一个 Start 的 JobEntry；Transformation 中的 Step 是并行执行的。

Kettle 也提供了丰富的组件，主要分为十大类：脚本组件、映射组件、统计组件、连接组件、查询组件、流程组件、应用组件、转换组件、输出组件、输入组件。

**有兴趣更详细了解的可以参考 Kettle 官方文档，很详细的：**

**[https://www.kettle.net.cn/category/base](https://www.kettle.net.cn/category/base)**

### ODI、Data Service

ODI（Oracle Data Integrator）是 Oracle 公司提供的一种数据集成工具，能高效地实现批量数据的抽取、转换和加载。ODI 可以实现当今大多数的主流关系型数据库（Oracle、DB2、SQL Server、MySQL、SyBase）的集成。

ODI 提供了图形化客户端和 Agent（代理）运行程序。客户端软件主要用于对整个数据集成服务的设计，包括创建对数据源的连接架构、创建模型及反向表结构、创建接口、生成方案和计划等。Agent 运行程序是通过命令行方式在 ODI 服务器上启动的服务，对 Agent 下的执行计划周期性地执行。

ODI 的常见应用场景：

-   数据仓库：比如 ETL 阶段。
-   数据迁移：比如将某一源系统的数据迁移到新系统中。
-   数据集成：比如两个系统间高效的点到点数据传递。
-   数据复制：比如将一个 Instance 的数据复制另外一个 Instance 中。

SAP Data Services 软件能够提高整个企业的数据质量。利用出色的数据整合、数据质量管理和数据清理功能，你能够从企业的所有结构化和非结构化数据中挖掘价值；将数据转化为随时可用的可靠资源，从中获取业务洞察，并利用这些洞察简化流程提高效率。

传统数仓时代，DataStage 和 Informatica 占据了绝大多数市场份额，Kettle 在中小型 ETL 应用场景上也有广泛应用，ODI 和 DS 等 ETL 工具反而使用的不多。

虽然这些传统 ETL 工具曾经风靡全球，是经过生产检验的，并且产品化程度极高，但都面临着云时代的巨大冲击，以前不想拥抱不拥抱云，现状只能拥抱空气了。这些当时的巨头如今市场规模越来越小，除去非常传统老旧的项目，新的项目已经很少使用了，只有开源、云、SAAS 模式才是出路。

## 0x02 集成同步组件

### Sqoop、DataX

Sqoop，SQL-to-Hadoop 即 “SQL 到 Hadoop 和 Hadoop 到 SQL”。是 Apache 开源的一款在 Hadoop 和关系数据库服务器之间传输数据的工具。主要用于在 Hadoop 与关系型数据库之间进行数据转移，可以将一个关系型数据库（ MySQL ,Oracle 等）中的数据导入到 Hadoop 的 HDFS 中，也可以将 HDFS 的数据导出到关系型数据库中。

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_5_20220112081355224)

Sqoop 的工作机制：

Sqoop 命令的本质是转化为 MapReduce 程序。Sqoop 分为导入（import）和导出（export），策略分为 table 和 query ，模式分为增量和全量。

Sqoop 的优点：

-   可以高效、可控的利用资源，可以通过调整任务数来控制任务的并发度。
-   可以自动的完成数据映射和转换。由于导入数据库是有类型的，它可以自动根据数据库中的类型转换到 Hadoop 中，当然用户也可以自定义它们之间的映射关系
-   支持多种数据库，如 Mysql，Orcale 等数据库

* * *

DataX 是阿里巴巴集团内被广泛使用的离线数据同步工具 / 平台，实现包括 MySQL、Oracle、SqlServer、Postgre、HDFS、Hive、ADS、HBase、TableStore(OTS)、MaxCompute(ODPS)、DRDS 等各种异构数据源之间高效的数据同步功能。

开源地址：[https://github.com/alibaba/DataX](https://github.com/alibaba/DataX)

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_6_20220112081355286)

数据交换通过 DataX 进行中转，任何数据源只要和 DataX 连接上即可以和已实现的任意数据源同步。

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_7_20220112081355458)

核心组件：  

-   Reader：数据采集模块，负责从源采集数据
-   Writer：数据写入模块，负责写入目标库
-   Framework：数据传输通道，负责处理数据缓冲等

从一个 JOB 来理解 Datax 的核心模块组件：

-   DataX 完成单个数据同步的作业，称为 Job，Job 会负责数据清理、任务切分等工作；
-   任务启动后，Job 会根据不同源的切分策略，切分成多个 Task 并发执行，Task 就是执行作业的最小单元
-   切分完成后，根据 Scheduler 模块，将 Task 组合成 TaskGroup ，每个 group 负责一定的并发和分配 Task

DataX 优点

-   可靠的数据质量监控：让数据可以完整无损的传输到目的端。
-   丰富的数据转换功能
-   精准的速度控制：新版本 DataX3.0 提供了包括通道 (并发)、记录流、字节流三种流控模式，可以随意控制你的作业速度，让你的作业在库可以承受的范围内达到最佳的同步速度。
-   强劲的同步性能：每一种读插件都有一种或多种切分策略，都能将作业合理切分成多个 Task 并行执行，单机多线程执行模型可以让 DataX 速度随并发成线性增长。
-   健壮的容错机制：多层次局部 / 全局的重试。
-   极简的使用体验：下载即可用、详细的日志信息。

* * *

Sqoop 和 DataX 都是非常流行的来源大数据离线同步工具，相比传统 ETL 工具易用性肯定会差很多（传统工具基本都能实现零代码开发纯图形界面操作），但由于天然具备的大数据处理能力而迅速得到普及。

DataX 面世晚了许多，所以拥有比 Sqoop 更多、更全、更强的功能，从而被广泛接受和使用。Sqoop 是 Hadoop 生态系统的重要一员问世比较早了，由于功能简单稳定成熟，甚至今年 05 月 06 日 [Apache 董事会宣布终止 Apache Sqoop 项目](http://mp.weixin.qq.com/s?__biz=MzA5MTc0NTMwNQ==&mid=2650731202&idx=1&sn=8f963124ebf9bf990063f5329f33c701&chksm=887defb4bf0a66a2d18384f0b8c5c135fd40a6018dca23bcfbf302aa9b314e4cb27be32cd01a&scene=21#wechat_redirect)。当然这里的中止并非不让用了，只是不在维护更新代码了，当然再次之前 Sqoop 代码已经三年没有更新了。

### Flume、Canal

Flume 是 Cloudera 提供的一个高可用的，高可靠的，分布式的海量日志采集、聚合和传输的系统，Flume 支持在日志系统中定制各类数据发送方，用于收集数据；同时，Flume 提供对数据进行简单处理，并写到各种数据接受方（可定制）的能力。

当前 Flume 有两个版本 Flume 0.9X 版本的统称 Flume-og，Flume1.X 版本的统称 Flume-ng。由于 Flume-ng 经过重大重构，与 Flume-og 有很大不同，使用时请注意区分。

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_8_20220112081355583)

_提示 官方这个图的 Agent4 的 Sink 画错了，不应该是 Avro Sink ，应该是 HDFS Sink 。_

上图是 Flume 设置多级 Agent 连接的方式传输 Event 数据。也支持扇入和扇出的部署方式，类似于负载均衡方式或多点同时备份的方式。

Flume 工作的机制：  

-   Flume-og 采用了多 Master 的方式。为了保证配置数据的一致性，Flume 引入了 ZooKeeper，用于保存配置数据，ZooKeeper 本身可保证配置数据的一致性和高可用，另外，在配置数据发生变化时，ZooKeeper 可以通知 Flume Master 节点。Flume Master 使用 gossip 协议同步数据。
-   Flume-ng 最明显的改动就是取消了集中管理配置的 Master 和 Zookeeper，变为一个纯粹的传输工具。Flume-ng 另一个主要的不同点是读入数据和写出数据由不同的工作线程处理（称为 Runner）。在 Flume-og 中，读入线程同样做写出工作（除了故障重试）。如果写出慢的话（不是完全失败），它将阻塞 Flume 接收数据的能力。这种异步的设计使读入线程可以顺畅的工作而无需关注下游的任何问题。

Flume 优势：  

-   Flume 可以将应用产生的数据存储到任何集中存储器中，比如 HDFS、HBase。
-   当收集数据的速度超过将写入数据的时候，也就是当收集信息遇到峰值时，这时候收集的信息非常大，甚至超过了系统的写入数据能力，这时候， Flume 会在数据生产者和数据收容器间做出调整，保证其能够在两者之间提供平稳的数据。
-   提供上下文路由特征。
-   Flume 的管道是基于事务，保证了数据在传送和接收时的一致性。
-   Flume 是可靠的，容错性高的，可升级的，易管理的, 并且可定制的。

* * *

Canal 是阿里巴巴旗下的一款开源项目，纯 Java 开发。基于数据库增量日志解析，提供增量数据实时订阅和消费，目前主要支持了 MySQL，也支持 mariaDB。

很多大型的互联网项目生产环境中使用，包括阿里、美团等都有广泛的应用，是一个非常成熟的数据库同步方案，基础的使用只需要进行简单的配置即可。

github 地址：[https://github.com/alibaba/canal](https://github.com/alibaba/canal)  

当前的 Canal 支持源端 MySQL 版本包括 5.1.x , 5.5.x , 5.6.x , 5.7.x , 8.0.x

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_9_20220112081355661)

Canal 是通过模拟成为 MySQL 的 slave 的方式，监听 mysql 的 binlog 日志来获取数据，binlog 设置为 row 模式以后，不仅能获取到执行的每一个增删改的脚本，同时还能获取到修改前和修改后的数据，基于这个特性，Canal 就能高性能的获取到 MySQL 数据数据的变更。  

* * *

Flume 和 Canal 都是适用于特定场景下的大数据同步组件，通常用于实时数据处理场景：

-   Flume 主要用于将日志文件实时同步到 Kafka 或 HDFS，供下游消费。  
-   Cannal 主要是解析 Mysql binlog 日志，在不影响业务的前提下将数据实时同步到 Kafka。

### DTS、GoldenGate

DTS（Data Transmission Service）是阿里云提供的一种数据传输服务，功能非常强大。支持 RDBMS、NoSQL、OLAP、Kafka 等各种数据源间的数据交互，集数据同步、迁移、订阅、集成、加工于一体，助您构建安全、可扩展、高可用的数据架构。  

Oracle GoldenGate 软件提供了一个单一的平台，这个平台可以为任何企业环境实现秒一级的灾难备份。GoldenGate 是一种基于日志的结构化数据复制方式，它通过解析源数据库在线日志或归档日志获得数据的增删改变化（数据量只有日志的四分之一左右），再将这些变化应用到目标数据库，实现源数据库与目标数据库同步、双活。

DTS 和 GoldenGate 不同时期诞生两款收费的数据同步工具或服务，都能够实现异构数据间的实时同步。

ColdenGate 诞生于传统数仓时期，后来被 Oracle 收购了以闭源软件工具的形式售卖，通常在关系型数据库间实现实时同步。DTS 诞生于阿里云，以一种 SAAS 服务的形式对外售卖，支持目前市面上几乎所有的数据源之间的实时同步，可以完全替代 Cannal。

ColdenGate 主要提供的是后台功能好像没有前端页面，但 DTS 跟现在大多数付费版的大数据组件一样提供一套 web 版本的操作和进度查看页面。   

### Maxwell、DSG

这两个数据同步工具我也没听过，只是群友们有提到过，这里列出来也给大家个参考。  

* * *

Maxwell 是一个能实时读取 MySQL 二进制日志 binlog，并生成 JSON 格式的消息，作为生产者发送给 Kafka，Kinesis、RabbitMQ、Redis、Google Cloud Pub/Sub、文件或其它平台的应用程序。

常见应用场景： ETL、维护缓存、收集表级别的 dml 指标、增量到搜索引擎、数据分区迁移、切库 binlog 回滚方案等。

官网：[http://maxwells-daemon.io](http://maxwells-daemon.io)

GitHub 地址：[https://github.com/zendesk/maxwell](https://github.com/zendesk/maxwell)

Maxwell 主要提供了下列功能：

-   支持 SELECT \* FROM table 的方式进行全量数据初始化
-   支持在主库发生 failover 后，自动恢复 binlog 位置 (GTID)
-   可以对数据进行分区，解决数据倾斜问题，发送到 kafka 的数据支持 database、table、column 等级别的数据分区
-   工作方式是伪装为 Slave，接收 binlog events，然后根据 schemas 信息拼装，可以接受 ddl、xid、row 等各种 event

除了 Maxwell 外，目前常用的 MySQL Binlog 解析工具主要有阿里的 Canal、mysql_streamer 。

* * *

DSG-RealSync Oracle 数据库同步复制及容灾技术。与传统的数据复制技术不同，DSG RealSync 技术是针对数据库提供了基于逻辑的交易复制方式。该方式通过直接捕获源数据库的交易，将数据库的改变逻辑复制到目标系统数据库中，实现源系统和目标系统数据的一致性。

该技术在复制上存在以下几个特点：

-   按需复制：查询和统计系统往往不需要所有的原始数据，因此完全可以按需要复制数据。RealSync 系统支持对指定信息的按需复制，减少存储和网络带宽的成本。
-   多种同步模式：实时复制、定时复制、手工复制
-   对生产系统的低干扰性：DSG 实时数据复制技术不需要通过任何数据库的引擎来获取变更数据，而是通过数据库自身的信息获取源系统上的改变并传送给目的系统，不会对生产系统造成性能影响。
-   系统异构可提供更多的优化空间：源数据库系统和目的数据库系统的可异构，主要包括索引规则和存储参数（如数据块大小、回滚段等）。因此可以在目标数据库上根据业务特点进行调整和优化，完全不受源系统的限制。
-   支持的多种复制策略：RealSync 可以被灵活配置，以支持各种复制策略，支持各种增值应用，如：一对一单向复制；一对多复制；多对一复制等。

_有兴趣了解的可以参考这篇文章：  
_

_[https://www.cnblogs.com/oracle-dsg/archive/2010/05/27/1745477.html](https://www.cnblogs.com/oracle-dsg/archive/2010/05/27/1745477.html)_

但说实话，大清早亡了，开源技术那么多，我们不见得非要使用这些古老的技术组件了。  

**第二部分**

**承上，我们接着介绍两种新型 ETL 工具、大数据发展不同阶段产生的六种主要计算引擎、五种流程控制组件。** 

最后我们简单讨论两个话题：

-   这么多组件我们该如何抉择？
-   如何快速将工具引入生产实践？

0x01 新型 ETL 工具

### 传统 ETL 工具，通常工具化程度很高，不需要编程能力且提供一套可视化的操作界面供广大数据从业者使用，但是随着数据量的激增，跟关系型数据库一样只能纵向扩展去增加单机的性能，这样数据规模的增长跟硬件的成本的增长不是线性的。

而新型 ETL 工具天然适应大数据量的同步集成计算，且支持实时处理，但缺点也很明显，就是工具化可视化程度低，搭建配置难度也比传统 ETL 工具要高，并且需要数据从业者具备一定的程序开发功底而传统数仓环境中的数据人绝大多数是不懂开发的。

但相信随着大数据技术的进一步成熟，终究还会走向低代码和 SQL 化的方向上去的。那时候少部分人负责组件 / 平台的开发和维护，大部分人使用这些组件去完成业务开发。  

### StreamSets

Streamsets 是由 Informatica 前首席产品官 Girish Pancha 和 Cloudera 前开发团队负责人 Arvind Prabhakar 于 2014 年创立的公司，总部设在旧金山。

Streamsets 产品是一个开源、可扩展、UI 很不错的大数据 ETL 工具，支持包括结构化和半 / 非结构化数据源，拖拽式的可视化数据流程设计界面。Streamsets 利用管道处理模型（Pipeline）来处理数据流。你可以定义很多 Pipeline，一个 Pipeline 你理解为一个 Job 。

Streamsets 旗下有如下三个产品： 

-   Streamsets data collector(核心产品, 开源)：大数据 ETL 工具。
-   Streamsets data collector Edge(开源): 将这个组件安装在物联网等设备上，占用少的内存和 CPU。
-   Streamsets control hub(收费项目)：可以将 collector 编辑好的 pipeline 放入 control hub 进行管理，可实现定时调度、管理和 pipeline 拓扑。

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_10_20220112081355911)

_StreamSets 开发页面_

在管道的创建上分为了三个管道：

-   data collector pipeline：用户普通 collector 开发。
-   data collector Edge Pipeline：将开发好的 pipeline 上传到对应 Edge 系统。
-   microservice pipeline：提供微服务。

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_11_20220112081356114)

管道创建好后，会根据需要去选择对应的组件信息。  

主要有以下几类组件：

-   origins (extract)：数据来源，数据从不同的数据源抽取。（一个 pipeline 中只能有一个数据来源）
-   processor(transform)：数据转化，将抽取来的数据进行过滤，清洗。
-   destination(load)：数据存储，将数据处理完后存入目标系统或者转入另一个 pipeline 进行再次处理。
-   executor：由处理数据组件的事件触发 executor , 执行相应任务。例如：某个组件处理失败，发送邮件通知。

### WarterDrop

Waterdrop 项目由 Interesting Lab 开源，是一个非常易用，高性能、支持实时流式和离线批处理的海量数据处理产品，架构于 Apache Spark 和 Apache Flink 之上。

Spark 固然是一个优秀的分布式数据处理工具，但是直接使用 Spark 开发是个不小的工程，需要一定的 Spark 基础以及使用经验才能开发出稳定高效的 Spark 代码。除此之外，项目的编译、打包、部署以及测试都比较繁琐，会带来不少的时间成本和学习成本。

除了开发方面的问题，数据处理时可能还会遇到以下不可逃避的麻烦：

-   数据丢失与重复
-   任务堆积与延迟
-   吞吐量低
-   应用到生产环境周期长
-   缺少应用运行状态监控

Waterdrop 诞生的目的就是为了让 Spark 的使用更简单，更高效，并将业界使用 Spark 的优质经验固化到 Waterdrop 这个产品中，明显减少学习成本，加快分布式数据处理能力在生产环境落地。

_gitHub 地址：_

_[https://github.com/InterestingLab/waterdrop](https://github.com/InterestingLab/waterdrop)_

_软件包地址：_

_[https://github.com/InterestingLab/waterdrop/releases](https://github.com/InterestingLab/waterdrop/releases)_

_文档地址：_

_[https://interestinglab.github.io/waterdrop-docs/](https://interestinglab.github.io/waterdrop-docs/)_

_项目负责人 _

_Gary(微信: garyelephant) , RickyHuo(微信: chodomatte1994)_

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_12_20220112081356333)

\__Waterdrop_ 架构\_  

Waterdrop 使用场景：

-   海量数据 ETL
-   海量数据聚合
-   多源数据处理

Waterdrop 的特性：

-   简单易用，灵活配置，无需开发；可运行在单机、Spark Standalone 集群、Yarn 集群、Mesos 集群之上。
-   实时流式处理, 高性能, 海量数据处理能力。
-   模块化和插件化，易于扩展。Waterdrop 的用户可根据实际的需要来扩展需要的插件，支持 Java/Scala 实现的 Input、Filter、Output 插件。
-   支持利用 SQL 做数据处理和聚合。
-   方便的应用运行状态监控。

## 0x02 计算引擎

### 上边两种新型 ETL 工具的出现简化了数据处理操作，同步、集成、计算可以统一在一个工具内完成且有不错的界面可以使用，但对于一些更加复杂灵活的场景不一定能够支撑。

### 大数据场景下计算引擎还是主流，并且衍生出了许许多多的组件。我们这里无法一一列举，就分别挑选不同时期被广泛使用的几个做介绍吧。

### MapReduce

MapReduce 将复杂的、运行于大规模集群上的并行计算过程高度地抽象到了两个函数：Map 和 Reduce。它采用 “分而治之” 策略，一个存储在分布式文件系统中的大规模数据集，会被切分成许多独立的分片（split），这些分片可以被多个 Map 任务并行处理。  

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_13_20220112081356489)

_MapReduce 工作流程，来源于网络_

不同的 Map 任务之间不会进行通信

不同的 Reduce 任务之间也不会发生任何信息交换

用户不能显式地从一台机器向另一台机器发送消息

所有的数据交换都是通过 MapReduce 框架自身去实现的

MapReduc 是 Hadoop 组件里的计算框架模型，另外还有分布式存储组件 HDFS、资源管理组件 Yarn。一开始计算和资源管理是耦合在一起的，Hadoop 2.0 才将其拆分开，这大大增加 Hadoop 使用的灵活性。  

MapReduce 的缺陷：

-   第一，MapReduce 模型的抽象层次低，大量的底层逻辑都需要开发者手工完成。
-   第二，只提供 Map 和 Reduce 两个操作。很多现实的数据处理场景并不适合用这个模型来描述。实现复杂的操作很有技巧性，也会让整个工程变得庞大以及难以维护。
-   第三，在 Hadoop 中，每一个 Job 的计算结果都会存储在 HDFS 文件存储系统中，所以每一步计算都要进行硬盘的读取和写入，大大增加了系统的延迟。

### Tez

Hadoop（MapReduce/Yarn、HDFS） 虽然能处理海量数据、水平扩展，但使用难度很大，而 Hive 的出现恰好解决这个问题，这使得 Hive 被迅速的推广普及成为大数据时代数据仓库组件的代名词（存储使用 hdfs，计算使用 MapReduce。Hive 只是一个壳根据自身维护的表字段跟底层存储之间映射关系 Hcatlog，对用户提交的 SQL 进行解析、优化，然后调用底层配置的执行引擎对底层数据进行计算）。  

为解决 Hive 执行性能太差的问题，在计算引擎方面出现了 Tez，数据存储方面出现了 ORC（一种专门针对 Hive 开发的列式存储压缩格式。当然 HDFS 本身也有一些存储压缩格式，另外还有一个比较流行的列示存储格式 Parquet）这也使得 Hive 的性能有了质的提升。  

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_14_20220112081356693)

_MR 与 Tez 的比较，来源于网络_

MapReduce 每一步都会落磁盘，这大大影响力执行效率

Tez 是 Apache 开源的支持 DAG （有向无环图，Directed Acyclic Graph）作业的计算框架。它把 Ｍap/Reduce 过程拆分成若干个子过程，同时可以把多个 Ｍap/Reduce 任务组合成一个较大的 DAG 任务，减少了 Ｍap/Reduce 之间的文件存储。同时合理组合其子过程，也可以减少任务的运行时间。加上内存计算 Tez 的计算性能实际上跟 Spark 不相上下。

Tez 直接源于 MapReduce 框架，核心思想是将 Map 和 Reduce 两个操作进一步拆分，即 Map 被拆分成 Input、Processor、Sort、Merge 和 Output， Reduce 被拆分成 Input、Shuffle、Sort、Merge、Processor 和 Output 等，这样，这些分解后的元操作可以任意灵活组合，产生新的操作，这些操作经过一些控制程序组装后，可形成一个大的 DAG 作业。

### Spark 、Flink

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_15_20220112081356802)

Apache Spark 是一个围绕速度、易用性和复杂分析构建的大数据处理框架，用于大规模数据处理的统一分析引擎，致力于一个组件满足大数据处理和分析的所有计算场景。

Spark 是当今最流行的分布式大规模数据处理引擎，被广泛应用在各类大数据处理场景。2009 年，美国加州大学伯克利分校的 AMP 实验室开发了 Spark。2013 年，Spark 成为 Apache 软件基金会旗下的孵化项目。而现在，Spark 已经成为了该基金会管理的项目中最活跃的一个。

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_16_20220112081356911)

_SparkUI Stage 页面_

Spark 应用场景：

-   离线计算：使用算子或 SQL 执行大规模批处理，对标 MapReduce、Hive。同时提供了对各种数据源（文件、各种数据库、HDFS 等）的读写支持。
-   实时处理：以一种微批的方式，使用各种窗口函数对流式数据进行实时计算。主要实现在这两部分：Spark Streaming、Structure Streaming（Spark 2.3 版本推出）。
-   MLlib：一个常用机器学习算法库，算法被实现为对 RDD 的 Spark 操作。这个库包含可扩展的学习算法，比如分类、回归等需要对大量数据集进行迭代的操作。
-   GraphX：控制图、并行图操作和计算的一组算法和工具的集合。GraphX 扩展了 RDD API，包含控制图、创建子图、访问路径上所有顶点的操作。

Spark 数据结构：

-   RDD：弹性分布式数据集，它代表一个可以被分区（partition）的只读数据集，它内部可以有很多分区，每个分区又有大量的数据记录（record）。RDD 表示已被分区、不可变的，并能够被并行操作的数据集合。
-   DataFrame：可以被看作是一种特殊的 DataSet 可以被当作 DataSet\[Row] 来处理，我们必须要通过解析才能获取各列的值。
-   DataSet：数据集的意思，它是 Spark 1.6 新引入的接口。就像关系型数据库中的表一样，DataSet 提供数据表的 schema 信息比如列名列数据类型。

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_17_20220112081357114)

_RDD、DataFrame、DataSet 对比_

Spark 数据结构发展历史：

-   RDD API 在第一代 Spark 中就存在，是整个 Spark 框架的基石。
-   接下来，为了方便熟悉关系型数据库和 SQL 的开发人员使用，在 RDD 的基础上，Spark 创建了 DataFrame API。依靠它，我们可以方便地对数据的列进行操作。
-   DataSet 最早被加入 Spark SQL 是在 Spark 1.6，它在 DataFrame 的基础上添加了对数据的每一列的类型的限制。
-   在 Spark 2.0 中，DataFrame 和 DataSet 被统一。DataFrame 作为 DataSet\[Row]存在。在弱类型的语言，如 Python 中，DataFrame API 依然存在，但是在 Java 中，DataFrame API 已经不复存在了。

* * *

Flink 起源于 2008 年柏林理工大学一个研究性项目， 在 2014 年被 Apache 孵化器所接受，然后迅速地成为了 ASF（Apache Software Foundation）的顶级项目之一。德国人对 Flink 的推广力度跟美国人对 Spark 的推广差的比较远，直到 2019 年阿里下场才使得 Flink 在国内得到广泛应用，并且以很高的频率进行版本迭代。

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_18_20220112081357208)

_Flink 组件栈_

基于流执行引擎，Flink 提供了诸多更高抽象层的 API 以便用户编写分布式任务：

-   DataSet API：对静态数据进行批处理操作，将静态数据抽象成分布式的数据集，用户可以方便地使用 Flink 提供的各种操作符对分布式数据集进行处理，支持 Java、Scala 和 Python。
-   DataStream API：对数据流进行流处理操作，将流式的数据抽象成分布式的数据流，用户可以方便地对分布式数据流进行各种操作，支持 Java 和 Scala。
-   Table API：对结构化数据进行查询操作，将结构化数据抽象成关系表，并通过类 SQL 的 DSL 对关系表进行各种查询操作，支持 Java 和 Scala。
-   Flink ML：Flink 的机器学习库，提供了机器学习 Pipelines API 并实现了多种机器学习算法。
-   Gelly：Flink 的图计算库，提供了图计算的相关 API 及多种图计算算法实现。

* * *

如上所述，Flink 等于说是把 Spark 的功能重新实现了一遍，区别在于 Spark 是由批入流 Flink 是由流入批。由于起步较晚，Flink 能够大量吸收 Hadoop、Spark 的优秀经验，凭借更高层次的抽象、更简洁的调用方式、高的吞吐、更少的资源占用，在实时计算、实时数仓等场景迅速超越了 Spark。但 Flink 想要完全超越 Spark 还有很长的路要走，比如对 SQL 的支持、批流一体的实现、机器学习、图计算等等。

对于数据开发者来说，Spark 比 MapReduce 支持的场景更广使用起来也容易的多，Flink 相比 Spark 同样更易用了。所以往后大数据开发的门槛将会越来越低：完全 SQL 化、低代码甚至会像传统 ETL 工具一样无代码。大数据从业者未来的路该怎么走？这是个值得思考的问题。

### ClickHouse 、Doris

ClickHouse 是 Yandex 在 20160615 开源的一个数据分析的 MPP 数据库。并且在 18 年初成立了 ClickHouse 中文社区，应该是易观负责运营的。

ClickHouse 实质上是一个数据库。为了获得极致的性能，ClickHouse 在计算层做了非常细致的工作，竭尽所能榨干硬件能力，提升查询速度。它实现了单机多核并行、分布式计算、向量化执行与 SIMD 指令、代码生成等多种重要技术。普通大数据集群，单机十几亿数据检索秒出。因此许多即席查询场景 ClickHouse 被广泛使用。

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_19_20220112081357271)

* * *

Apache Doris 是一个现代化的 MPP 分析型数据库产品，百度开源并贡献给 Apache 社区。仅需亚秒级响应时间即可获得查询结果，有效地支持实时数据分析。Apache Doris 的分布式架构非常简洁，易于运维，并且可以支持 10PB 以上的超大数据集。

Apache Doris 可以满足多种数据分析需求，例如固定历史报表，实时数据分析，交互式数据分析和探索式数据分析等。令您的数据分析工作更加简单高效！

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_20_20220112081357646)

* * *

ClickHouse 确实是一个非常优秀的产品。但为了获得查询时的高性能我们放弃了一些东西：

-   ClickHouse 过度依赖大宽表。
-   ClickHouse 难以支持高并发的业务场景。
-   并不完全能够支持标准 SQL ，UDF 也是最近才支持的。
-   ClickHouse 集群的运维复杂度也一定曾让您感到过头疼。

Doris 的诞生试图去解决 ClickHouse 的这些问题，让我们拭目以待吧。

## 0x03 流程控制组件

### 流程控制（也称工作流、任务流）是 ETL 重要的组成部分，通常是以 DAG 的方式配置，每次调用都会沿着有向无环图从前往后依次执行直至最后一个任务完成。

流程控制可以在 ETL 工具内配置，也可以在调度系统配置。传统 ETL 工具基本上都是单机版的，如果 ETL 的任务节点分布在多个服务器上，整体的流程依赖就会变的复杂起来（跨服务器的调度无法解决，就只剩下两种方法了：预估前置依赖完成时间、监控前置依赖运行状态比如将运行状态写入数据库等），这时候使用调度工具里的流程控制功能就是最优解。 

### Hudson

Hudson 是一个可扩展的持续集成引擎，是 SUN 公司时期就有的 CI 工具，后来因为 ORACLE 收购 SUN 之后的商标之争，创始人 KK 搞了新的分支叫 Jenkins 。今天的 Hudson 还在由 ORACLE 持续维护，但风头已经远不如社区以及 CloudBees 驱动的 Jenkins。 

主要用于：

-   持续、自动地构建 / 测试软件项目，如 CruiseControl 与 DamageControl。
-   监控一些定时执行的任务。

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_21_20220112081357927)
_Hudson 操作界面_

Hudson 拥有的特性包括：

-   易于安装：只要把 hudson.war 部署到 servlet 容器，不需要数据库支持。
-   易于配置：所有配置都是通过其提供的 web 界面实现。
-   集成 RSS/E-mail/IM：通过 RSS 发布构建结果或当构建失败时通过 e-mail 实时通知。
-   生成 JUnit/TestNG 测试报告。
-   分布式构建支持，Hudson 能够让多台计算机一起构建 / 测试。
-   文件识别：Hudson 能够跟踪哪次构建生成哪些 jar，哪次构建使用哪个版本的 jar 等。
-   插件支持：Hudson 可以通过插件扩展，你可以开发适合自己团队使用的工具。

Hudson 是我们早期数仓项目中使用的一个调度工具，当然 Hudson 还有其它的一些功能，但我们用到的仅仅是调度。由于 ETL 系统整体的复杂性，源端数据汇总集成、数仓分层计算、数据推送到外部系统，我们分别部署在了三台服务器上，这时候 Hudson 就起到了跨服务器调度依赖控制的作用。  

### Airflow、 Azkaban、Oozie

Airflow 是一个可编程，调度和监控的工作流平台，基于有向无环图 (DAG)，Airflow 可以定义一组有依赖的任务，按照依赖依次执行。Airflow 提供了丰富的命令行工具用于系统管控，而其 web 管理界面同样也可以方便的管控调度任务，并且对任务运行状态进行实时监控，方便了系统的运维和管理。

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_22_2022011208135868)

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_23_20220112081358162)

_上图以及以下两段文字来源于公众号：数据社_

主要有如下几种组件构成：  

-   web server: 主要包括工作流配置，监控，管理等操作。
-   scheduler: 工作流调度进程，触发工作流执行，状态更新等操作。
-   消息队列：存放任务执行命令和任务执行状态报告。
-   worker: 执行任务和汇报状态。
-   mysql: 存放工作流，任务元数据信息。

具体执行流程：

-   scheduler 扫描 dag 文件存入数据库，判断是否触发执行。
-   到达触发执行时间的 dag , 生成 dag_run，task_instance 存入数据库。
-   发送执行任务命令到消息队列。
-   worker 从队列获取任务执行命令执行任务。
-   worker 汇报任务执行状态到消息队列。
-   schduler 获取任务执行状态，并做下一步操作。
-   schduler 根据状态更新数据库。

* * *

Azkaban 是由 Linkedin 开源的一个批量工作流任务调度器。用于在一个工作流内以一个特定的顺序运行一组工作和流程。Azkaban 定义了一种 KV 文件格式来建立任务之间的依赖关系，并提供一个易于使用的 web 用户界面维护和跟踪你的工作流。

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_24_20220112081358380)

_Azkaban 操作界面_

* * *

Oozie 起源于雅虎，主要用于管理与组织 Hadoop 工作流。Oozie 的工作流必须是一个有向无环图，实际上 Oozie 就相当于 Hadoop 的一个客户端，当用户需要执行多个关联的 MR 任务时，只需要将 MR 执行顺序写入 workflow.xml，然后使用 Oozie 提交本次任务，Oozie 会托管此任务流。

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_25_20220112081358521)

以上三个组件都是在大数据环境下使用的调度工具，Oozie 属于非常早期的调度系统了并且深度服务于 Hadoop 生态目前使用的很少了，Azkaban 目前也使用的不多，Airflow 还有一定的市场。

### DolphinScheduler

Apache DolphinScheduler 是一个分布式、去中心化、易扩展的可视化 DAG 工作流任务调度系统，其致力于解决数据处理流程中错综复杂的依赖关系，使调度系统在数据处理流程中开箱即用。

DolphinScheduler 于  2019 年 8 月 29 日 进入 Apache 孵化器，于 2021 年 4 月 9 日成为 Apache 顶级项目。

![](http://image109.360doc.com/DownloadImg/2022/01/1220/237693118_26_20220112081358724)

_DolphinScheduler 操作界面_

DolphinScheduler 提供了许多易于使用的功能，可加快数据 ETL 工作开发流程的效率。其主要特点如下：

-   通过拖拽以 DAG 图的方式将 Task 按照任务的依赖关系关联起来，可实时可视化监控任务的运行状态；
-   支持丰富的任务类型；
-   支持工作流定时调度、依赖调度、手动调度、手动暂停 / 停止 / 恢复，同时支持失败重试 / 告警、从指定节点恢复失败、Kill 任务等操作；
-   支持工作流全局参数及节点自定义参数设置；
-   支持集群 HA，通过 Zookeeper 实现 Master 集群和 Worker 集群去中心化；
-   支持工作流运行历史树形 / 甘特图展示、支持任务状态统计、流程状态统计；
-   支持补数，并行或串行回填数据。

## 0x04 总结

### 这么多组件我们该如何抉择

写到这里，计划中的 ETL 工具以及类 ETL 组件已经全部介绍完了，但我只是挑了不同时期比较流行的很少一部分，刚数了下有 26 个。  

工具组件这么多，做为技术人肯定是学不完的，经常看到一些简历罗列了一二十个，大而全哪哪都不精这样的人市场上是没啥竞争力的。所以我们必须聚焦，在数据处理的全流程，每一类型选取其中一种组件深入学习并努力在生产实践中运用。在特定的场景，多种工具其实实现的功能大体是类似的，无非是后起的在性能、稳定性、易用性上会比早出现的好很多。

-   如果你已经在一家公司做数据了，就先看下公司的技术栈。如果市面上还是比较流行，恭喜你努力的去学精学透，从生产使用技巧到底层运行原理去深挖，其它的类似组件简单了解就行。如果公司使用的技术不好用或者过于陈旧就努力推动促使公司更换技术栈吧。


-   如果你还不是做数据的，或者以后想转数据，或者公司的技术栈陈旧又没法更换，这就需要自我学习了。我们需要挑选不同类型下最流行或者最优秀的那个深入学习，一通百通。比如 ETL 工具传统的那些就没必要学了直接学 StreamSets 或者 WaterDrop 即可；实时计算直接学 Flink 即可不用看 Spark 了；众多的 OLAP 我们直接学 ClickHouse 或者 Doris 即可其它的也不用看了；调度嘛直接 DS 就好了。求职时候尝试把你最最擅长的那一两个组件表现出来反而更容易获得面试官的认可。

### 如何快速将工具引入生产实践

当我们选好一个新的组件后，从入门到精通大致需要以下三个过程：

-   第一步，先用起来，并且对组件有基本的认知。我们需要先想明白我们想让该组件帮我们解决什么问题然后将问题分类细化逐个解决，拿最小集合快速的跑通全流程。
-   第二步，学习组件原理特性，将更多的特性运用到业务中去解决更多的实际问题，同时对现有流程进行调优。
-   第三步，学习源码对组件本身进行优化改造，用于解决更多的现实问题，如果有可能就贡献给社区。当然走到这一步的还是少数人，这是平台开发的事，数据开发很少有这样的机会因为投入产出比很差。

最后，我们举个例子吧：一个项目需要使用一个之前没用过的  ETL 工具，我们如何能够在两周内达到生产应用的水平呢？

首先我们需要搞明白我们需要 ETL 系统做什么？  

-   数据源连接，连接上之后我们能够抽取或者加载数据。  
-   文件的导入导出功能。
-   源端库数据的全量 / 增量抽取。  
-   目标库数据的插入 / 更新 / 删除。
-   流程控制：任务流动方向的控制、串行 / 并行控制、任务成功 / 失败后的处理、必要的条件判断并能依据判断结果执行不同的操作。
-   参数传递与接收：多层 / 多级任务，参数能够从最外层往下一层或者从最上游往下游传递，每个任务节点要能通过变量接收到上层或者上游参数。
-   ETL 执行过程监控，方便后续自动化监控和告警。
-   ETL 运行出错时候的重试 / 补数。  

正常来说，所有 ETL 工具都是能够支持以上功能的，我们需要找到它们（否则就得尽快寻找补救方案），然后就能正常的进行 ETL 开发了。我们需要使用新的工具先让系统稳定、准确的跑起来，同时能够提供有效的自动化运行监控，性能优化是下一步该做的事情。 
 [http://www.360doc.com/content/22/0112/20/48274702_1012993271.shtml](http://www.360doc.com/content/22/0112/20/48274702_1012993271.shtml)
