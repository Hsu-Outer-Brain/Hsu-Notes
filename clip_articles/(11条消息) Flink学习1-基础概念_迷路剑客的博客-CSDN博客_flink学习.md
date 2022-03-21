# (11条消息) Flink学习1-基础概念_迷路剑客的博客-CSDN博客_flink学习
-   更多 Flink 系列文章请点击[Flink 系列文章](https://blog.csdn.net/baichoufei90/article/details/105674793)
-   更多大数据文章请点击[大数据好文推荐](https://blog.csdn.net/baichoufei90/article/details/90264272)

本文是作者学习 Flink 的一些文档整理、记录和心得体会，希望与大家共同学习探讨。

## 1.1 概念

Apache Flink 是一个开源的**分布式**流式处理框架，**高性能**、**高可用**，他有强大的流式和批处理能力，通过语义保证数据处理**精确性**。流式处理方面，Flink 能对有界、无界数据流做有状态的计算（stateful computations）。

## 1.2 特点

他有如下特点：

-   能同时支持高吞吐和低事件延迟（亚秒级）
-   真实时流处理
-   基于数据流模型，支持 DataStream API 中的`event time`和无序处理
-   优雅的 Java 和 Scala API，集成了较丰富的 streaming operator，自定义 operator 也较为方便，并且可以直接调用 API 完成 stream 的 split 和 join，可以完整的表达 DAG 图。
-   跨不同时间语义（`event time`，`processing time`）的弹性**window**（时间，计数，会话，自定义触发器）
-   支持 Flink 托管的 State
-   具有`Exactly Once`处理保证的容错能力
-   支持多种 window 语义，如 Session, Tumbling, Sliding window，方便实时统计
-   流式程序中自然背压
-   支持流、批处理，且将批当作流的特例，最终实现批流统一。
-   Flink 自主实现多层次内存管理而不完全依赖于 JVM，可以在一定程度上避免大量 Full-GC 问题。

    在此基础上，批处理的 lib 包可支持图运算和机器学习 (FlinkML)；流式处理的 lib 支持复杂事件处理；流批统一的关系型 SQL&Table API
-   支持迭代计算
-   程序可自动优化
-   内置支持 DataSet（批处理）API 中的迭代程序（BSP）
-   自定义内存管理，可在内存和核外数据处理算法之间实现高效，可靠的切换。在 JVM 层面实现了内存管理和优化
-   可兼容 Apache Hadoop MapReduce 和 Apache Storm
-   可集成 YARN，HDFS，HBase 和 Apache Hadoop 生态系统的其他组件

## 1.3 示例

### 1.3.1 流处理

下面这个例子展示了用 scala 语言写的一段对 5 秒时间窗口内的数据进行流式 word count 的流处理程序：

```java
case class WordWithCount(word: String, count: Long)
val text = env.socketTextStream(host, port, '\n')
val windowCounts = text.flatMap { w => w.split("\\s") }
  .map { w => WordWithCount(w, 1) }
  .keyBy("word")
  .timeWindow(Time.seconds(5))
  .sum("count")
windowCounts.print()

```

### 1.3.2 批处理

下面这个例子展示了用 scala 语言写的对数据进行 word count 批处理程序：

```java
case class WordWithCount(word: String, count: Long)
val text = env.readTextFile(path)
val counts = text.flatMap { w => w.split("\\s") }
  .map { w => WordWithCount(w, 1) }
  .groupBy("word")
  .sum("count")
counts.writeAsCsv(outputPath)

```

## 1.4 Flink 对比其他

### 1.4.1 Storm

开发 API 较复杂，不支持状态托管， 用户必须自己处理状态持久化和一致性保证，如果某个挂了状态很难恢复

可参考

-   [流计算框架 Flink 与 Storm 的性能对比](https://ververica.cn/developers/stream-computing-framework/)  
    ![](https://img-blog.csdnimg.cn/20201207222922856.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
    -   吞吐  
        Storm 单线程吞吐约为 8.7 万条 / 秒，Flink 单线程吞吐可达 35 万条 / 秒。Flink 吞吐约为 Storm 的 3-5 倍。
    -   要求消息投递语义为 Exactly Once 的场景下只能选择 Flink，Storm 只支持 “至多一次” (At Most Once) 和 “至少一次” (At Least Once) 。
    -   Flink 支持状态管理、窗口统计等功能

![](https://img-blog.csdnimg.cn/20201207223039472.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

### 1.4.2 JStorm

阿里巴巴 fork，现在开始采用 flink 的思想

### 1.4.3 Spark Streaming

微批，非真正的实时流式处理

### 1.4.4 Structured Streaming

同样是纯实时思想. 可参考

-   [Structured Streaming VS Flink](https://mp.weixin.qq.com/s/F7jHlcc-91bUbCNx50hXww)
-   [SparkStreaming StructuredStreaming Flink Storm 对比](https://blog.csdn.net/weixin_42526352/article/details/105317112)
-   [大数据框架 Flink、Blink、Spark Streaming、Structured Streaming 和 Storm 之间的区别](https://blog.csdn.net/tzs_1041218129/article/details/108728710)

比较：

-   都支持丰富时间语义，不过 SS 不支持 IngestionTime
-   处理模式  
    SS 除了定时触发、连续批触发以外，还支持了跟 flink 类似的连续处理模式，不再使用批处理引擎，而是类似 Flink 的持续处理模式，端到端延迟最低可达 1ms。
-   Structured Streaming 将实时数据当做被连续追加的表，流上的每一条数据都类似于将一行新数据添加到表中。  
    Structured Streaming 定义了无界表的概念，即每个流的数据源从逻辑上来说看做一个不断增长的动态表（无界表），从数据源不断流入的每个数据项可以看作为新的一行数据追加到动态表中。用户可以通过静态结构化数据的批处理查询方式（SQL 查询），对数据进行实时查询。
-   都可以注册流表，进行 SQL 分析
-   运行架构  
    类似，SS 主节点是 Driver，Flink 是 JobManager
-   异步 IO 和维表 Join  
    FlinkSql 支持异步维表 Join；

    Structured Streaming 不直接支持与维表的 join 操作，但是可以使用 map、flatmap 及 udf 等来实现该功能，所有的这些都是同步算子，不支持异步 IO 操作。但是 Structured Streaming 可直接与静态数据集 join，也可以帮助实现维表的 join 功能，**当然维表要不可变。** 
-   状态管理  
    Flink 状态不需要用户自己维护，更灵活丰富
-   Join  
    Flink Join 支持更全，SS 限制较多  
    ![](https://img-blog.csdnimg.cn/20201205235950895.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
    ![](https://img-blog.csdnimg.cn/20201206000024314.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

## 2.1 项目架构

![](https://img-blog.csdnimg.cn/20190227105420801.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

### 2.1.1 Deploy - 部署方式

-   Local  
    单 JVM，本地调试
-   Cluster-Standalone  
    Flinke Standalone 模式，不依赖其他资源
-   Cluster-Yarn  
    最常用的模式，依赖 Yarn
-   Cloud

### 2.1.2 Core

分布式的流式数据流的运行时，是 Flink 的核心实现层，包括分窗、各种 Time、一致性语义、任务管理和调度、执行计划等。一般用户不用关心此层代码，只需调用上层 API 即可满足开发需求。

### 2.1.3 DataStream 和 DataSet API

建立在 Core 层之上，分别为流式处理和批处理 API。需要注意的是，Flink 的批处理屙屎建立在流式架构上的。

### 2.1.4 Libraries

建立在 API 层上的一些高级应用 lib 包，如机器学习、关系型 API 等。

## 2.2 Flink 集群架构

### 2.2.1 概述

![](https://img-blog.csdnimg.cn/20200706175142966.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

![](https://img-blog.csdnimg.cn/20200826160459397.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/20200925234931204.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/2020092523542829.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/20201102175928566.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

Flink 集群主要有两个角色，即 Master-JobManager 和 SlaveWorker-TaskManager。

### 2.2.2 JobManager-Master

#### 2.2.2.1 职责

-   协调分布式计算
-   部署在 yarn 上时，负责申请 container 资源以启动 TaskManager
-   Scheduler - 调度任务
-   CheckpointCoordinator 触发、协调 checkpoint
-   故障故障

#### 2.2.2.2 主要组件

-   ResourceManager  
    负责资源分配和回收，内部有[SlotManager](#slotmanager)负责管理`TaskManagerSlot`（最小资源分配和调度单位），整个 Flink Cluster 只有一个 ResourceManager 实例。

    Flink 在不同运行环境 (Yarn、Mesos、K8S 等) 中实现了不同 ResourceManager，如`YarnResourceManager`。

    主要方法：

    -   registerJobManager(JobMasterId, ResourceID, String, JobID, Time)  
        向 ResourceManager 注册一个 JobMaster
    -   requestSlot(JobMasterId, SlotRequest, Time)  
        向 ResourceManager 申请一个 Slot
-   Dispatcher

    -   为用户提供了 Rest 接口来提交执行 Flink 应用
    -   一个 FlinkCluster 只有一个 Dispatcher 实例
    -   持久化 Job
    -   为每个提交的 job 开启一个新的 JobManager 以运行 Job
    -   在 master 出错时恢复 Job
    -   运行 Flink WebUI 以提供 job 运行界面
-   JobMaster  
    ![](https://img-blog.csdnimg.cn/20200926000512102.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

    每个 JobMaster 仅负责管理一个`Job`的执行，监控单个 Job 运行的所有 Task。

    多个 job 可同时在一个 Flink 集群中运行，每个 Job 拥有自己的 JobMaster。

    JobMaster 主要职责有：

    -   单个 Job 生命周期管理，以及 Job 状态查询
    -   该 Job 对应 Slot 资源申请和管理
    -   调度 Task 执行
    -   Failover 容错
    -   CheckpointCoordinator 触发分布式 Checkpoint 快照

    JobMaster 主要组件有：

    -   RPC Endpoint  
        基于 Akka 的 RPC Server，组件之间通讯使用
    -   Scheduler  
        使用 JobGraph 创建 ExecutionGraph，并调度 ExecutionGraph（包括为每个`ExecutionVertex`向 Flink ResourceManager 申请 Slot 资源、获取到资源后创建 ExecutionVertex 部署信息发送给 TM），管理调度策略、Failover 策略、Slot 分配等
        -   会利用创建好的`ExecutionGraph.schedulingTopology`来创建 failover 策略类`FailoverStrategy`的实现类，比如`RestartPipelinedRegionFailoverStrategy`，即当某个节点失败时，重启一个 Region 的所有节点。
        -   创建调度策略`SchedulingStrategy`实现类，如`EagerSchedulingStrategy`，这个实现类表示所有 Task 在同一时间一起调度。调度策略详情可见 2.10.3。
    -   CheckpointCoordinator  
        负责定时触发（发送 Checkpoint 快照任务给各相关 Task）算子和状态的分布式快照 Checkpoint，管理整个分布式 Checkpoint 快照流程，搜集 TM 上报的状态快照 ACK 信息等。
    -   SlotPool
        -   作用是服务由`ExecutionGraph`向本 JM 提起的 Slot 请求，当没有足够 Slot 时就向 ResourceManager 申请新的 Slot。
        -   一旦申请到 Slot，SlotPool 也会保存他们，这样即使 ResourceManager 挂了，依然可以分配这里已空闲的 Slot。
        -   当 Slot 无人使用时，会释放。
        -   所有 Slot 分配由一个自增的`AllocationID`标记

#### 2.2.2.3 其他重要概念

一个 Flink 集群有一个 JobManager 进程实例，一些 HA 场景可能有多个 StandBy JM。

### 2.2.3 TaskManagers-Slave worker

#### 2.2.3.1 职责

-   启动后，连接 JobManager，等待分配任务
-   执行 DataFowGraph 中的 tasks（准确来说是 subtasks ）
-   缓存
-   和其他 TaskManager 交换数据流

#### 2.2.3.2 重要知识点

-   TaskManager 至少一个  
    每个 Job 至少会有一个 TaskManager。
-   最小资源分配单位为 Slot  
    最小资源分配单位为 Slot，他决定了并发能力。
-   TaskManagers 个数  
    Flink on YARN 时，TaskManagers 个数 = `Job 最大并行度 / 每个 TaskManager 分配的 Slot 数`。也就是说 TM 数量由 Flink 根据你的 Job 情况自动推算，`-yn`启动参数失效了。

#### 2.2.3.3 TaskExecutor

在 TM 上实际执行的类为`TaskExecutor`。

### 2.2.4 客户端

客户端虽然不是 Flink 运行时和作业执行时的一部分，但它是被用作准备和提交 dataflow 到 JobManager 。

Job 提交完成之后，客户端可以断开连接 (分离模式)，也可以保持连接来接收 Job 执行进度报告（attach 模式）。

## 2.3 网络通讯架构

### 2.3.1 节点之间 - Akka

Flink 内部节点之间的通信是用 Akka，比如 JobManager、TaskManager、Yarn-ResourceManager 之间的通信，比如状态从 TaskManager 发送到 JobManager。

-   Akka 优点
    -   Akka 是基于协程的，性能不容置疑
    -   基于 scala 的偏函数，易用性良好
-   Akka 缺点
    -   只支持 RPC 通信
    -   Akka 版本可能跟用户 Akka 版本冲突，因为 Akka 不同版本之间无法互相通信，导致用户 Akka 版本无法升级。
    -   Flink 的 Akka 配置是针对 Flink 自身来调优的，可能跟用户自己代码中的 Akka 配置冲突。
    -   如果只是为了通信，没有必要用 Actor Model。Actor Model 重要的一点是解决了并发带来的问题。

可参考：

-   [Akka 介绍](https://blog.csdn.net/hxcaifly/article/details/84998240)
-   [协程和线程的区别](https://blog.csdn.net/jek123456/article/details/80449694)

### 2.3.2 算子之间 - Netty

算子之间基于 Netty 通信。

-   Netty 优点  
    Netty 相比 Akka 而言更加底层，可以为不同的应用层通信协议（RPC，FTP，HTTP 等）提供支持。

可参考：

-   [Netty 介绍](https://blog.csdn.net/hxcaifly/article/details/85336795)

## 2.4 Flink On Yarn 架构

![](https://img-blog.csdnimg.cn/20191031231233738.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

Flink Yarn Session 部署流程如下：

1.  Client 上传依赖 jar 到 HDFS  
    开启一个新的 Flink Yarn Session 时，Client 首先检查申请的资源（AM 所需的内存和 vcore）可用的是否足够，然后将包括 Flink 和配置的 jar 上传到 HDFS。
2.  Client 为 AM 申请资源  
    Client 为 Flink 所用的 AM 向 Yarn RM 申请 Container 资源。
3.  Yarn 启动 AM(JobManager)  
    Yarn 的 RM 收到申请后，为 AM 在 NM 上分配首个 Container，用来启动 Flink AM。具体来说，NM 会先为该 AM 做一些 Container 分配准备工作，如下载资源（就包括刚才 Client 上传的 Flink Jar 等文件）。准备完成后，就启动了 AM。

    JobManager 和 AM 运行在同一个 Container 中，所以 AM 知道 JobManager 的地址。所以在启动完成后，会为随后需要创建的 TaskManager 们生成一个新的 Flink 配置文件，该文件就包含了该 JobManager 的链接地址。该文件也会被上传到 HDFS。

    此时，Flink 的 Web 服务也会在该 Container 开始运行。

    注意，Yarn 为 app 分配的所有端口都是临时端口，可使得用户并行执行多个 Flink Yarn Session。
4.  AM 向 RM 申请 Container 资源以启动 TaskManager  
    RM 向拥有适合 Container 资源的 NM 发送分配指令，NM 接到请求后先从 HDFS 下载相关的 jar 文件、配置文件，然后启动 TaskManager。此时 TaskManager 就能正确运行，并连接到正确的 JobManager。
5.  此时一个 Flink Yarn Session 集群部署完毕，可以开始接受 Job。

更多内容可参考：

-   [how Flink and YARN interact](https://ci.apache.org/projects/flink/flink-docs-master/ops/deployment/yarn_setup.html#start-a-long-running-flink-cluster-on-yarn)
-   [Hadoop-Yarn 学习](https://blog.csdn.net/baichoufei90/article/details/90649077)

## 2.5 Flink TaskManager 内存模型

### 2.5.1 概述

可参考:

-   [Set up Task Executor Memory](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_setup.html#managed-memory)
-   [Detailed Memory Model](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_detail.html)
-   [Flink Memory Configuration](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/config.html#memory-configuration)

**注意：本章节基于 Flink 1.10，该版本有大量修改。** 

Flink 通过严格控制其各种组件的内存使用情况，在 JVM 之上提供有效的工作负载。Flink 开发者已经竭尽可能来设置最优默认配置，但还是有一些时候需要细粒度内存调优。

Flink 内存使用情况：  
![](https://img-blog.csdnimg.cn/20200310133056468.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

-   Total Process Memory（`taskmanager.memory.process.size`）  
    这个就是在容器环境（Yarn、Docker 等）Flink 程序请求的总内存大小。
    -   Total Flink Memory（`taskmanager.memory.flink.size`）  
        不建议同时设定此选项和`Total Process Memory`，否则可能导致内存设定冲突
        -   JVM Heap
            -   Framework Heap  
                JVM 堆内存中 Flink 框架本身专用部分，一般不调整

                相关配置`taskmanager.memory.framework.heap.size`
            -   Task Heap  
                JVM 堆内存中运行算子和用户代码专用部分

                相关配置`taskmanager.memory.task.heap.size`。
        -   Off-Heap Memory
            -   Managed Memory  
                由 Flink 管理的本地内存，`Batch jobs`用来排序、存放 HashTable、中间结果的缓存；`Streaming jobs`的 RocksDB State Backend。这块内存堆皮处理算子提高处理效率很重要，有些操作可直接在原始数据进行而无需序列化过程，所以 Flink 会尽可能在不超过配额前提下分配 Managed Memory，并当该内存不足时，优雅得将数据落盘，避免 OOM。

                相关配置`taskmanager.memory.managed.size`和`taskmanager.memory.managed.fraction`，都设置时 size 会覆盖 fraction，如果都没设置则用[默认值 0.4](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/config.html#taskmanager-memory-managed-fraction)。

                还可参考[https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_tuning.html#configure-memory-for-state-backends](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_tuning.html#configure-memory-for-state-backends)
            -   Direct Memory

                -   Framework Off-heap Memory  
                    运行 Flink 框架的堆外直接（或本地）内存，一般不调整

                    相关配置`taskmanager.memory.managed.size`、`taskmanager.memory.framework.off-heap.size`
                -   Task Off-heap Memory  
                    运行算子的堆外直接、本地的 task 专用内存

                    相关配置`taskmanager.memory.task.off-heap.size`
                -   Network Memory  
                    Flink 用来在 Task 之间交换数据（比如网络传输 Buffer）的直接内存。

                    相关配置：`taskmanager.memory.network.min`、`taskmanager.memory.network.max`、`taskmanager.memory.network.fraction`
    -   其他 Off-Heap Memory
        -   JVM metaspace  
            相关配置`taskmanager.memory.jvm-metaspace.size`
        -   JVM Overhead  
            JVM 其他开销使用的本地内存，比如线程栈、代码缓存、GC 空间等。

            相关配置：`taskmanager.memory.jvm-overhead.min`、`taskmanager.memory.jvm-overhead.max`、`taskmanager.memory.jvm-overhead.fraction`

### 2.5.2 Flink 1.11 中修改部分

#### 2.5.2.1 概述

参考  
[https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/memory/mem\\\_setup.html](https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/memory/mem\_setup.html)

flink 1.11 在 flink 1.10 调整了内存模型基础上，1.11 进一步调整，JM 和 TM 内存模型更统一

必须设定以下某项，因为没有默认值

| TaskManager                     | JobManager                     |
| ------------------------------- | ------------------------------ |
| taskmanager.memory.flink.size   | jobmanager.memory.flink.size   |
| taskmanager.memory.process.size | jobmanager.memory.process.size |

| taskmanager.memory.task.heap.size  
taskmanager.memory.managed.size | jobmanager.memory.heap.size |

如果使用低版本升级到高版本 Flink，可参考[迁移指南](https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/memory/mem_migration.html)  
![](https://img-blog.csdnimg.cn/20200720171104475.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

注意上图中`Off-heap`包括

-   Direct memory
-   Native memory

#### 2.5.2.2 TaskManager 内存模型

![](https://img-blog.csdnimg.cn/20200813114756791.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/20200813120309116.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/20200813120257134.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

#### 2.5.2.3 JobManager 内存模型

![](https://img-blog.csdnimg.cn/20200813114939852.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/2020081314330295.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

Off-heap Memory 用途

-   Flink 框架，如 Akka 网络通信开销
-   在作业提交时（例如一些特殊的批处理 Source）或是 Checkpoint 完成后的回调函数中执行的用户代码

### 2.5.2 内存所有配置

Flink 大多默认配置已经足够好，一般只需要配置`taskmanager.memory.process.size` 或`taskmanager.memory.flink.size`，再配置下`taskmanager.memory.managed.fraction`控制 Jvm 堆和`Managed Memory`比例。

以下配置适用于 TaskManager 和 JobManager:

| Key                                                                                                                     | Default | Type       | Description                                                                                                       |
| ----------------------------------------------------------------------------------------------------------------------- | ------- | ---------- | ----------------------------------------------------------------------------------------------------------------- |
| taskmanager.memory.process.size                                                                                         | (none)  | MemorySize | TaskExecutors 的所有内存大小即`Total Process Memory`.                                                                     |
| 使用 YarnConainter 时应该和 Contianer 内存一致（不设本值，只指定`yjm`或`ytm`已经和 Contianer 内存一致）                                             |         |            |                                                                                                                   |
| taskmanager.memory.flink.size                                                                                           | (none)  | MemorySize | TaskExecutor 的总 Flink 内存大小即`Total Flink Memory`，除去 JVM Metaspace 和 JVM Overhead                                   |
| taskmanager.memory.framework.heap.size                                                                                  | 128 mb  | MemorySize | 堆内 Flink Framework 内存，不会分配给 Task Slot                                                                             |
| taskmanager.memory.framework.off-heap.size                                                                              | 128 mb  | MemorySize | 堆内 Flink Framework 内存，包括 direct 和 native，不会分配给 Task Slot。                                                         |
| Flink 计算 JVM 最大 `direct memory`时会考虑本部分                                                                                  |         |            |                                                                                                                   |
| taskmanager.memory.task.heap.size                                                                                       | (none)  | MemorySize | `Task Heap Memory`，JVM 堆内存中运行算子和用户代码的 task 专用部分.                                                                  |
| 未指定本值时，等于`Total Flink Memory` 减去（`Framework Heap Memory` 加 `Task Off-Heap Memory` 加 `Managed Memory` 加 `Network Memory` |         |            |                                                                                                                   |
| taskmanager.memory.task.off-heap.size                                                                                   | 0 bytes | MemorySize | `Task Off-heap Memory`，JVM 堆外直接、本地内存中运行算子和用户代码的 task 专用部分.                                                        |
| Flink 计算 JVM 最大 `direct memory`时会考虑本部分                                                                                  |         |            |                                                                                                                   |
| taskmanager.memory.managed.size                                                                                         | (none)  | MemorySize | 位于堆外非直接内存中的`Managed Memory`。`Batch jobs`用来排序、存放 HashTable、中间结果的缓存；`Streaming jobs`的 RocksDB State Backend Memory。 |

使用者即可以以 MemorySegments 的形式从内存管理器中分配内存，也可以从内存管理器中保留字节并将其内存使用率保持在该范围内。  
如果未指定，则采用下一项配置来计算。 |
| taskmanager.memory.managed.fraction | 0.4 | Float | 位于堆外非直接内存中的`Managed Memory`比例，如果没有配置具体大小时，该空间大小由`Total Flink Memory`乘以本值得出。 |
| taskmanager.memory.network.fraction | 0.1 | Float | (TaskManager Only) 堆外直接内存中的`Network Memory`，Flink 用来在 Task 之间交换数据（比如网络传输 Buffer），用`Total Flink Memory`乘以本值来计算。但结果如果小于 network.min 就用 min，大于 max 就用 max。 |
| taskmanager.memory.network.max | 1 gb | MemorySize | network memory 上限 |
| taskmanager.memory.network.min | 64 mb | MemorySize | network memory 下限 |
| taskmanager.memory.jvm-metaspace.size | 96 mb | 位于堆外的 MemorySize | JVM Metaspace 内存 |
| taskmanager.memory.jvm-overhead.fraction | 0.1 | Float | 位于堆外的 JVM 其他开销使用的本地内存，比如线程栈、代码缓存、GC 空间等。包括 native 内存但不包括 direct 内存,  
Flink 计算 JVM 最大 `direct memory`时不会考虑本部分.  
jvm-overhead 区具体大小用`Total Process Memory` 乘以本值来计算。但结果如果小于 jvm-overhead.min 就用 min，大于 max 就用 max。 |
| taskmanager.memory.jvm-overhead.max | 1 gb | MemorySize | jvm-overhead 上限 |
| taskmanager.memory.jvm-overhead.min | 192 mb | MemorySize | jvm-overhead 下限 |

### 2.5.3 本地运行内存配置

本地 ide 直接运行且不运行集群时，只需做以下内存配置：

| Memory component               | 配置项                                   | 本地运行时默认值 |
| ------------------------------ | ------------------------------------- | -------- |
| Task heap                      | taskmanager.memory.task.heap.size     | 无限大      |
| Task off-heap                  | taskmanager.memory.task.off-heap.size | 无限大      |
| Managed memory                 | taskmanager.memory.managed.size       | 无限大      |
| Network memory                 | taskmanager.memory.network.min        |          |
| taskmanager.memory.network.max | 无限大                                   |          |

以上配置不是必须的，未设定时采用默认值。

注意： 启动的本地进程的实际 JVM 堆大小不受 Flink 的控制，取决于您如何启动该进程。 如果您想控制 JVM 堆大小，则必须显式传递相应的 JVM 参数，例如 -Xmx，-Xms。

### 2.5.4 上 / 下界内存配置

#### 2.5.4.1 概述

-   Network Memory 可以是 Total Flink Memory 的一小部分
-   JVM overhead 可以是 Total Process Memory 的一小部分

这种内存有三类设置情况，但必须在最大值和最小值之间，否则 Flink 会启动失败。以下以`network`举例。

#### 2.5.4.2 fraction 计算后处于区间

```
total Flink memory = 1000Mb,
network min = 64Mb,
network max = 128Mb,
network fraction = 0.1

```

此时 network memory = 1000Mb x 0.1 = 100Mb，处于 max/min 之间.

#### 2.5.4.3 fraction 计算后处于区间之外

```
total Flink memory = 1000Mb,
network min = 128Mb,
network max = 256Mb,
network fraction = 0.1

```

此时 network memory = 1000Mb x 0.1 = 100Mb &lt; min，所以 network memory = 128Mb

#### 2.5.4.4 定义了 Total Flink memory 和其他组件内存

```
total Flink memory = 1000Mb,
task heap = 100Mb,
network min = 64Mb,
network max = 256Mb,
network fraction = 0.1

```

此时 network memory 是 total Flink memory 的剩余内存，但必须位于 min-max 之间，否则失败。

### 2.5.5 JVM 调优

可参考:

-   [JVM and Logging Options](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/config.html#jvm-and-logging-options)

Flink 进程启动时根据配置或派生来的内存组件大小自动推断设置 JVM 参数

| JVM Arguments           | TaskManager Value                                                         | JobManager Value                                                                                             |
| ----------------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| -Xmx and -Xms           | Framework Heap + Task Heap Memory                                         | JVM Heap Memory                                                                                              |
| -XX:MaxDirectMemorySize | Framework Off-Heap + Task Off-Heap(计算时包括了用户代码使用的本地非直接内存) + Network Memory | Off-heap Memory (计算时包括了用户代码使用的本地非直接内存；jobmanager.memory.enable-jvm-direct-memory-limit 为 true 时才会设置此 JVM 选项) |
| -XX:MaxMetaspaceSize    | JVM Metaspace                                                             | JVM Metaspace                                                                                                |

### 2.5.6 内存调优指南

-   [Memory tuning guide](https://ci.apache.org/projects/flink/flink-docs-release-1.11/en/ops/memory/mem_tuning.html)

#### 2.5.6.1 TaskManager

-   Framework Heap / OffHeap Framework Heap  
    Framework Memory 包括堆内和非堆两部分，一般不需要调优，除非确定 Flink 需要更多内存来存放内部数据结构或算子，以及特殊场景或数据结构，比如高并行度场景。

#### 2.5.6.2 JobManager

-   Java Heap

    -   总内存
    -   jobmanager.memory.heap.size
    -   [-Xms / -Xmx](https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/memory/mem_setup.html#jvm-parameters)
-   OffHeap Heap  
    包括`JVM Direct Memory`和`Native Memory`，当`jobmanager.memory.enable-jvm-direct-memory-limit`为`true`就开启对 JVM Direct Memory 内存大小限制，此时可通过`-XX:MaxDirectMemorySize`/ `jobmanager.memory.off-heap.size`限制。当发生了`OutOfMemoryError: Direct buffer memory`异常时可调整该值。

    如果不设，就从`Total Flink Memory`减去`JVM Heap`来推断该值。

#### 2.5.6.3 TaskManager StateBackend

-   Heap StateBackend  
    运行无状态 Job 或使用 Heap StateBackend（包括 Memory 和 FS），此时可将`managed memory`大小设为 0，把尽可能多的内存分配给 JVM Heap 来运行用户代码。
-   RocksDB StateBackend  
    使用本地内存，并且默认状态下会限制不超过`managed memory`，否则可能导致内存超限而使得应用被 RM 杀掉。详见
    -   [how to tune RocksDB memory](https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/state/large_state_tuning.html#tuning-rocksdb-memory)
    -   [state.backend.rocksdb.memory.managed](https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/config.html#state-backend-rocksdb-memory-managed)

### 2.5.7 [内存相关常见问题](https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/memory/mem_trouble.html)

## 2.6 Task 和 Operator Chain

详见[Tasks and Operator Chains](#task-operator)

## 2.7 Flink Application 与 Flink Job

### 2.7.1 Flink Application Execution

`Flink Application Execution`指一个用户程序，可通过`main()`方法产生一个或多个 Flink Job：

1.  获取一个 \`ExecutionEnvironment\`\`\`\`java
    val env = StreamExecutionEnvironment.getExecutionEnvironment()

    ```

    ```
2.  Load/create 初始数据 \`\`\`java
    val text: DataStream[String] = env.readTextFile("file:///path/to/file")

    ```

    ```
3.  数据转换 \`\`\`java
    val mapped = text.map {x => x.toInt }

    ```

    ```
4.  指定计算结果输出位置，如各种 Sink 等 \`\`\`java
    windowCounts.print()

    ```

    ```
5.  触发程序执行 \`\`\`java
    env.execute("Streaming WordCount")

    ```

    ```

每个 Job 由`ExecutionEnvironment`提供的方法来控制 job 执行（如并行度）。

提交目的地有几个，他们的集群生命周期和资源隔离保证有所不同：

-   Flink Session Cluster  
    一个长期运行的 Flink 集群，可接受若干 Flink Job 运行。

    该模式下的 Flink 集群生命周期与 Job 无关，以前该模式成为 Flink Session Cluster。
-   Flink Job Cluster  
    只运行一个 Flink Job 的专用 Flink Cluster。Flink On Yarn per-job 就是该模式。

    该模式下的 Flink 集群的生命周期和 Job 绑定。

    需要较多时间用来申请资源和启动 Flink Cluster，所以适合长期运行的 Flink Job。
-   Flink Application Cluster  
    只运行一个 Flink Application 上提交的 Job 的专用 Flink Cluster。Zeppelin per-job 使用该模式。

    该模式下的 Flink 集群的生命周期和 Flink Application 绑定。

    该模式中，Flink 应用的`main()`方法在集群中运行，而不是在 Client 中。

    提交 job 时不需要先启动 Flink Session Cluster 再提交 job 到该集群，而是将应用逻辑和依赖打到一个可执行的 jar 中，由`ApplicationClusterEntryPoint`负责调用该 jar 的`main`方法来提取 JobGraph 执行。

### 2.7.2 Flink Job

指 LogicGraph 的运行时表示，通过在 Flink Application 中调用`execute`方法来创建和提交。

## 2.8 执行计划

### 2.8.1 逻辑执行计划 - LogicalGraph(源码中的 JobGraph)

Logical Graph 是一个有向图，用来描述 streaming 程序的高层次逻辑。

图里的节点是 Operator，边代表算子和敌营数据流（或数据集）的 input/output 关系。

逻辑计划也通常被称为`dataflow graphs`（数据流图）

### 2.8.2 物理执行计划 - ExecutionGraphw / PhysicalGraph

PhysicalGraph 是将 LogicalGraph 翻译成在分布式运行时中运行的执行图的结果。

图里的节点是 Task，边代表数据流（或数据集）之间的 input/output 关系或 partition。

## 2.9 Partition

-   注意这个 Partition 不同于 Kafka 的 Partition，这是 Flink Partition 概念！  
    Partition 是整个数据流（或数据集）的独立子集，数据流中的每条 record 会被发到一个或多个 partition。
-   Partition 的消费者是 Task。
-   如果一个算子改变了数据流的 partition 划分方式，则称为`repartitioning`

## 2.10 [调度模型](https://ci.apache.org/projects/flink/flink-docs-release-1.11/cn/internals/job_scheduling.html)

### 2.10.1 Pipeline 与 Slot

#### 2.10.1.1 基础概念

前面提到过，Flink 资源调度单位为`Slot`，每个 TaskManager 有一个或多个 Slot，每个 Slot 可以运行多个不同 JobVertex 的并行 Task 实例组成的 pipeline。一个 pipeline 由多个连续 task 组成，比如并行度为 n 的 `MapFunction` 和 并行度为 n 的 `ReduceFunction`。

Pipeline 内部各算子实例之间通过流水线交互数据，效率很高。

比如有一个 job 包含 DataSource（并行度 4）、MapFunction（并行度 4）、ReduceFunction（并行度 3），此时一个 pipeline 就由`Source - Map - Reduce`序列组成。

如果当前有两个 TM，每个 TM 包含 3 个 Slot，则程序运行状况如下：  
![](https://img-blog.csdnimg.cn/20200810223825644.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

\*\*上图中同一个颜色的就是一个 pipeline！\*\*在一个 Slot 中运行，多个 pipeline 之间是并发运行。

Flink 内部使用`SlotSharingGroup`和`CoLocationGroup`来定义哪些 task 可以共享一个 Slot， 哪些 task 必须严格放到同一个 slot。  
![](https://img-blog.csdnimg.cn/20201108153821502.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

切记，只有不同 JobVertex 的实例才能放到一个 Slot 进行 Share。

#### 2.10.1.2 好处

-   Pipeline 中的算子实例之间数据交换在内存中进行，比通过网络交换数据效率高很多
-   同类型算子均匀分布到各个 Slot，避免较重的算子全部挤到一个 Slot 执行

### 2.10.2 JobManager 数据结构

#### 2.10.2.1 概述

job 运行期间，JM 跟踪分布式 task，以决定何时调度下一个 task，以及处理运行完成的 task 和执行失败。

#### 2.10.2.2 Transformation

我们先提一下一个重要的类`Transformation`：

-   他是一个抽象类，表示创建了一个 DataStream 的特定算子。每个 DataStream 都对应有一个底层的 Transformation，表示该 Stream 的发起者。
-   多个算子 API 会创建一棵 Transformation 树，当 Stream 程序提交运行的时候，这个树形结构图会被`StreamGraphGenerator`翻译为`StreamGraph`。
-   在运行时，一个 Transformation 不一定对应一个物理上的算子，因为某些算子只是逻辑上的，比如`union/split/select`数据流、partition 等。

    比如有以下一个 Transformation 图：  
    ![](https://img-blog.csdnimg.cn/20200827001305919.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

    会在运行时转换为如下物理算子图：  
    ![](https://img-blog.csdnimg.cn/20200827001332700.png#pic_center)

    而分区信息、union、split/select 等信息已经被编码到了 Source 到 Map 算子之间的边里。这些信息会在提交到集群 JM 后，被转为 ExecutionGraph 的 ResultPartition 和 InputGate 时使用。

#### 2.10.2.3 DataStream Job 提交流程

详见[Flink - 作业提交流程](https://blog.csdn.net/baichoufei90/article/details/108274922)

#### 2.10.2.2 JobGraph

JM 接收参数为`JobGraph`，他表示数据流，由作为顶点（JobVertex）的算子和中间结果`IntermediateDataSet`构成。每个算子都有属性，如并行度、运行的代码、依赖的类库等。

#### 2.10.2.3 ExecutionGraph

JM 将 JobGraph 转换为`ExecutionGraph`。

-   ExecutionJobVertex  
    ExecutionGraph 是一个并行版本的 JobGraph，对于 JobGraph 的每个 JobVertex 来说，对应着多个表示每个并行子任务实例的`ExecutionVertex`。比如并行度 100 的某个算子，则 JobGraph 中有 1 个表示该算子的 JobVertex，ExecutionGraph 中有 100 个表示该算子的 ExecutionVertex。

    ExecutionVertex 的作用是跟踪特定子任务的执行状态。
-   ExecutionJobVertex  
    而所有对应一个 JobVertex 的 ExecutionVertex 被封装在一个`ExecutionJobVertex`中，用来追踪该算子整体状态。
-   IntermediateResult  
    追踪`JobGraph.IntermediateDataSet`状态
-   IntermediateResultPartition  
    追踪每个 partition 的 IntermediateDataSet 状态

#### 2.10.2.4 JobGraph 和 ExecutionGraph

![](https://img-blog.csdnimg.cn/20200810230803674.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

每个 ExecutionGraph 都有一个与之相关的 job 状态信息，用来表示当前的 job 执行状态。

#### 2.10.2.5 Job 状态机

![](https://img-blog.csdnimg.cn/20200810231235512.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

![](https://img-blog.csdnimg.cn/20200926004147137.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

-   Job 初始为`Created`
-   Job 开始运行后转为`Running`
-   Job 开始正常结束后转为`Finished`
-   Job 出错时，先转为`Failing`以撤销所有运行中 Task。
-   当所有 JobVertex 转为`final`状态且配置为不可重启，则 Job 转为`Failed`状态
-   如果 Job 重启，则转为`Restarting`，准备好后转为`Created`
-   用户撤销 job 时，变为`Cancelling`，会撤销所有运行中 Task。当所有 Task 转为`final`状态后，转为`Cancelled`
-   `Cancelled`、`Failing`和`Finished`状态会导致全局的终止状态，触发 Job 清理
-   `Suspended`与上面三个状态不同，只会触发本地终止  
    只会在配置了 JM HA 时发生。意味着该 Job 只是在当前 JM 结束，但可由新拉起的 JM 使用持久化 HA 存储来找到该 Job 进行恢复和重启。总之，该状态的 Job 不会被完全清理！

#### 2.10.2.6 并行 Task 实例状态机

![](https://img-blog.csdnimg.cn/20200810232325708.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

![](https://img-blog.csdnimg.cn/20200926004837655.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

注意：由于一个 TasK 可能会被执行多次（比如在异常恢复后），所以`ExecutionVertex`的执行是由`Execution`来跟踪的，每个 ExecutionVertex 有`Current Execution`和`Prior Execution`。

### 2.10.3 ScheduleMode

Flink Job 调度模型相关枚举类为`ScheduleMode`，具体调度策略类为`SchedulingStrategy`，决定执行图中的任务怎么开始。  
![](https://img-blog.csdnimg.cn/20201108154356438.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

-   EAGER  
    立刻调度所有 task。

    主要用于无界流数据。
-   LAZY_FROM_SOURCES  
    从 Source 开始，按拓扑顺序，一旦上游所有输入数据就绪，则下游 task 开始。即前驱任务全部执行完成后，才开始调度后续任务，后续任务会读取上游缓存的输出数据来进行计算。

    适用于批处理作业。
-   LAZY_FROM_SOURCES_WITH_BATCH_SLOT_REQUEST  
    和`LAZY_FROM_SOURCES`类似，不同之处在于本选项会使用 batch slot 请求，这种模式可支持使用比请求的 request 更少的 slot 来执行 job。

    用户需要保证 job 不包含任何 pipelined shuffle
-   （正在开发中的）Pipelined region based  
    以 PipelinedRegion 作为单位进行调度

## 2.11 资源架构

### 2.11.1 概述

![](https://img-blog.csdnimg.cn/20200901164641774.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

为准备好运行的 Task 分配 Slot 资源申请核心为运行在 JobManager 的`SlotProvider`，有两种分配模式：

-   立刻分配  
    立刻满足。可调用`CompletableFuture#getNow(Object)`来获取已分配的 slot
-   排队分配  
    将申请请求排队，并返回一个 future，当有一个 slot 可用时变得可用

### 2.11.2 资源管理组件

-   Flink ResourceManager  
    负责资源分配和回收，内部有`SlotManager`负责管理`TaskManagerSlot`（最小资源分配和调度单位）。

    Flink 在不同运行环境 (Yarn、Mesos、K8S 等) 中实现了不同 ResourceManager，如`YarnResourceManager`。

    主要方法：

    -   registerJobManager(JobMasterId, ResourceID, String, JobID, Time)  
        向 ResourceManager 注册一个 JobMaster
    -   requestSlot(JobMasterId, SlotRequest, Time)  
        向 ResourceManager 申请一个 Slot
-   SlotManager  
    属于 ResourceManager，具体管理集群中已注册到 JM 的所有 TaskManagerSlot 的信息和状态，他们由 TaskExecutor 启动后注册到 RM 时提供。当 JM 为 Task 申请资源时，SlotManager 就会从当前空闲的 Slot 中按一定的匹配规则选择一个空闲的 Slot 分配给 Job 使用。

    内部通过 TaskExecutor 到 Flink ResourceManager 的定时心跳（包括了该 TM 的所有 Slot 状态信息）来更新 Slot 状态
-   SlotPool  
    每个 SlotPool 实例属于某个 Job 的对应的 JobMaster 实例

    -   作用是服务和缓存由`ExecutionGraph`向本 JM 提起的 Slot 请求，当没有足够 Slot 时就向 Flink ResourceManager 申请新的 Slot。
    -   一旦申请到 Slot，SlotPool 也会保存他们，这样即使 ResourceManager 挂了，依然可以分配这里已空闲的 Slot。
    -   申请到 Slot 后，SlotPool 从缓存的 Slot 请求中选择对应请求进行分配和 Job 下发执行。
    -   当 Slot 无人使用时，会释放。
    -   所有 Slot 分配由一个自增的 AllocationID 标记

### 2.11.3 分配流程

![](https://img-blog.csdnimg.cn/20201108150434156.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/2020090116595257.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/2020101412110656.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

提交任务详细流程可见[Flink - 作业提交流程](https://blog.csdn.net/baichoufei90/article/details/108274922)

1.  TM 启动后会连接、注册到 RM，成功后会将本 TM 携带的所有 Slot 信息传递到 RM
2.  Flink RM 收到 TM 的注册消息后，会有一个动态代理`AkkaInvocationHandler`触发`ResourceManager#sendSlotReport`方法将该 TM 注册到 SlotManager，此后该 TM 上的 Slot 资源可由 JM 进行分配。

    实际上就是注册到`SlotManagerImpl`，会将这些新注册的 Slot 先尝试从`SlotManagerImpl.pendingSlots`中匹配已有的 Slot 请求；否则就将这个状态为`TaskManagerSlot.State.FREE`的 Slot 放入`freeSlots`中保存，表示为可分配的空闲 Slot 资源。

    如果 TM 连接断开，则会移除该 TM 注册的 Slot。

    到这里，TM 已经成功连接并注册到 RM，但需要注意的是，TM 与 JM 的连接和注册要等到 RM 向 TM 提起 Slot 资源分配请求阶段。
3.  每个 Job 会对应启动一个 JM，他会连接 RM，然后调度本 Job，向 SlotPool 申请 Slot。  
    当然最开始因为还没连接到 RM 所以无法分配，需要放入`pending`队列等待分配
4.  RM 接收到 JM 的 SlotRequest 请求后，`SlotManagerImpl`尝试为该请求分配 Slot 资源  
    如果`findMatchingSlot(ResourceProfile)`有可用 TaskManagerSlot，就先从 freeSlots 中移除要分配的 Slot，然后使用`SlotManagerImpl#allocateSlot`进行分配，此时就会利用`AkkaInvocationHandler`动态代理发送 RPC 调用`TM#requestSlot`
5.  TM 接收到 requestSlot 后，查询`taskSlotTable`，若目标 Slot 空闲则调用`allocateSlot`开始分配 Slot，会连接、注册到该申请 Slot 的 JM，最后通过 RPC offerSlots 将 Slot 分配信息告知 JM
6.  JM 接收到分配成功的 Slot 信息，此后 Scheduler 组件开始 deploy 所有 ExecutionVertex，最后将 Task 部署描述信息`TaskDeploymentDescriptor`通过 RPC 发送给 TaskExecutor，提交任务  
    注意，一旦 JM 申请到 Slot，其 SlotPool 也会保存他们，这样即使 ResourceManager 挂了，依然可以分配这里已空闲的 Slot；当 Slot 无人使用时，会被自动释放。
7.  TM 接收到 JM 发送的 Task 部署信息后，会初始化 TM 各种服务，利用收到的部署信息组装`Task`，利用 ExecutionGraph 创建运行时图（为每个下游 inputChannel(ExecutionEdge) 生成一个`ResultSubPartition`实例`PipelinedSubpartition`、为每个输入`ExecutionEdge`生成一个 InputChannel 实例，每个 InputChannel 都只从上游单个 ResultSubPartition 消费）  
    ![](https://img-blog.csdnimg.cn/20201014115719220.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)
8.  Task 组装任务完成后，从 taskSlotTable 匹配分配的 Slot 和本 TM 拥有的 Slot，匹配上后启动 executingThread 线程来执行核心工作  
    包括从`PermanentBlobService`下载相关 lib jar、创建拥有这些类的引用的 ClassLoader、为该 Subtask 的每个 ResultPartition 实例分别创建一个`BufferPool`实例（由该 SubTask 的所有`ResultSubPartition`共享，属于 Floating Buffers）、初始化用户代码并创建`RuntimeEnvironment`（提供给执行代码访问，内容如 Task 的 name、并行度、Configuration、数据流 Reader 和 Writer，以及 TM 提供的一系列组件如 MemoryManager、IO Manager 等，最后执行用户定义的 StreamTask

分配 Slot 核心源码可见`SchedulerImpl#internalAllocateSlot`

```java

CompletableFuture<LogicalSlot> allocationFuture = scheduledUnit.getSlotSharingGroupId() == null ?
	allocateSingleSlot(slotRequestId, slotProfile, allocationTimeout) :
	allocateSharedSlot(slotRequestId, scheduledUnit, slotProfile, allocationTimeout);

allocationFuture.whenComplete((LogicalSlot slot, Throwable failure) -> {
	if (failure != null) {
		cancelSlotRequest(
			slotRequestId,
			scheduledUnit.getSlotSharingGroupId(),
			failure);
		allocationResultFuture.completeExceptionally(failure);
	} else {
		allocationResultFuture.complete(slot);
	}
});

```

9.  执行完成后，无论成功或失败，TM 都将 Slot 标记为已占用但未执行任务的状态
10. JM 的 SlotPool 缓存该 Slot，但不立刻释放，以便如果任务异常需要重启时需要重启 Slot 的问题，可以通过延时释放 Slot 实现尽快调度 FailOver task。
11. 超过一定时间如果 Slot 仍未被使用，则 SlotPool 通知 TM 释放 Slot
12. TM 得到通知后，通知 RM Slot 释放

### 2.11.4 YarnResourceManager

这是一种 Flink ResourceManager 实现，具体如下：  
![](https://img-blog.csdnimg.cn/20201014140805732.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

跟上面图的区别是，TaskExecutor 会在 JM 向 Flink RM 申请资源时发现 SlotManager 上的 Slot 不够，Flink RM 才会向 Yarn RM 申请 Container 资源来启动 TE 来注册到 RM，随后就是 RM 向 TE 请求 Slot，最后 TE Offer Slot 给 JM。

### 2.11.5 心跳

TM 与 RM 和 JM 之间会定时发送心跳同步 Slot 状态，以保证分布式系统的一致性恢复。当某组件长时间未收到其他组件的心跳信息时，就会认为对方失效并进入 FailOver 流程。

## 2.12 作业模式

Flink On Yarn 分为:

-   Per-Job
-   Yarn-Session  
    ![](https://img-blog.csdnimg.cn/2020110518064484.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)
-   Application  
    以上两种模式的共同问题是需要在 Client 执行用户代码以编译生成 Job Graph 才提交到集群运行，过程中需要下载相关 Jar 包、上传集群，客户端和网络负载压力容易成为瓶颈，尤其当多个用户共用一个客户端时。

    1.11.0 中引入了 Application 模式（FLIP-85）来解决上述问题，按照 Application 粒度来启动一个集群，属于这个 Application 的所有 Job 在这个集群中运行。核心是 Job Graph 的生成以及作业的提交不再在客户端执行，而是转移到 JM 端执行，这样网络下载上传的负载也会分散到集群中，不再有上述 Client 单点上的瓶颈。

    用户可以通过`bin/flink run-application`来使用 Application 模式，目前 Yarn 和 Kubernetes（K8s）都已经支持这种模式。Yarn application 会在客户端将运行作业需要的依赖都通过 Yarn Local Resource 传递到 JM。K8s Application 允许用户构建包含用户 Jar 与依赖的镜像，同时会根据作业自动创建 TM，并在结束后销毁整个集群，相比 Session 模式具有更好的 隔离性。K8s 不再有严格意义上的 Per-Job 模式，Application 模式相当于 Per-Job 在集群进行提交作业的实现。

## 3.1 处理无界和有界的数据

任何类型的数据都是作为事件流产生的，如信用卡交易、传感器数据、机器日志以及社交网站或者手机上的用户交互信息。数据可以分为无界和有界的。  
![](https://img-blog.csdn.net/20180928001306664?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 3.1.1 无界数据

无界数据有开端但没有定义结束。也就是说无界数据会源源不断的被生产，必须持续进行处理。处理无界数据通常要求以特定顺序（例如事件发生的顺序）摄取事件，以便能够推断出结果的完整性。

Flink 通过时间和状态的精准控制能够在无界流上运行任何类型的应用程序。

### 3.1.2 有界数据

有界数据有明确的开始和结束。可以在处理计算有界数据前先摄取到所有数据。与无界数据不同，处理有界流不需要有序地摄取，因为可以始终对有界数据集进行排序。 有界流的处理也称为批处理。

Flink 通过算法和数据结构来对有界流进行内部处理，这些算法和数据结构专门针对固定大小的数据集而设计，从而产生出色的性能。

## 3.1.3 同时处理有界和无界数据

![](https://img-blog.csdnimg.cn/20200825230327613.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

## 3.2 Dataflow

### 3.2.1 概述

Flink 中应用程序由用户自定义算子转换而来的流式 Dataflow 组成，这些 Dataflow 组成有向无环图 DAG，包含若干 Source 和若干。  
![](https://img-blog.csdnimg.cn/20200825210349649.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

上图展示了一段 Flink DataStream 程序转换为 DataFlow 的情况，图下方黄色圆内为一个算子。

大多数时候程序中的`transformation`和算子一一对应，但也有例外，比如上图，一个 transformation 包含了三个算子。

### 3.2.2 Dataflow 并行度

Flink 程序天生就是并行化和分布式的。在执行时，DataStream 有一个或更多的 StreamPartition，而且每个算子也有一个或多个 Subtask（由不同线程甚至进程、机器节点执行）。

算子 subtask 个数就是该算子的并行度（`operator.setParallelism(N)`），而一个 Job 的不同算子可能有不同并行度。

以下是实例程序的逻辑视图和并行视图：  
![](https://img-blog.csdnimg.cn/20200825231125443.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

## 3.3 Flink 部署方式灵活

和 Spark 一样，Flink 既能以 StandAlone 方式部署，又可以跑在 YARN Mesos Kubernetes 等资源调度器上。

Flink 在这些 RM 上运行时，关于提交和控制一个 Application 的所有通讯手段都是 REST 请求。

Flink 可以根据应用配置的并行性来识别出所需的资源并向 RM 申请。为了预防任务失败，Flink 会在 container 失败时重新申请新的资源。

## 3.4 Flink 可运行任意规模的应用

Flink 被设计来可以高效地运行任意规模的有状态的流式应用。应用程序可以并行拆分为数千个任务分布在集群中，同时执行。理论上来讲应用可以用到无限的资源。而且，Flink 可以轻松维护非常大的应用程序状态。 Flink 异步、增量的检查点 (checkpoint) 算法可以确保对处理延迟的影响最小化，同时能保证精确一次性(`exactly once`) 的状态一致性。

目前 Flink 应用广泛，很多公司使用它来处理巨大规模的应用，如：

-   处理万亿事件 / 天的应用
-   处理多个 TB 级的状态信息的应用
-   运行在数千个核上的应用

## 3.5 有装态流处理

Flink 内的部分算子是有状态的，意味着一个新的事件来到，被处理的逻辑需要依赖于之前累积的事件的结果。这些装态可用来简单计算没短时间内的数量，也可以用在复杂场景，比如欺诈检测特征计算。

前面已经提到过，Flink 算子可在不同线程并行执行。那么有状态算子的并行实例组在存储对应状态时是按 key 进行划分的，每个并行实例负责处理自己的那个 key 分组的事件和在本地维护这些 key 对应的状态。

![](https://img-blog.csdnimg.cn/20200826153052952.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

状态总是在本地访问，可使得 Flink 程序达到高吞吐低延迟，利用内存达到极致性能。![](https://img-blog.csdn.net/20180928210419199?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

有状态的 Flink 应用程序针对本地状态访问进行了特别优化。

任务状态信息始终保存在内存中，或是当状态信息超过可用内存时异步存储在高效的、位于磁盘的数据结构中。这样设计使得任务计算通常都是在内存中进行，延迟非常低。（是不是很熟悉，很 SparkStreaming 相似的思想）

## 3.6 Exactly Once 与容错性

前面说到过，Flink 能保证就算出现故障时也拥有精准一次的状态一致性。

Flink 的处理方式是周期性异步（异步原因是不阻塞正在进行的数据处理逻辑）获取状态并存储的检查点，来将本地状态持久化到存储，在出错时用以恢复，重放流。

状态快照的内容是

-   捕获整个分布式 pipeline 的状态
-   记录消费数据源的 offset
-   记录消费数据到达该点时的整个 JobGraph 中的状态

当发生故障时，就从最后一次成功存储的 Checkpoint 恢复状态，重置数据源，从状态中记录的消费 offset 开始重新消费。

## 3.7 端到端的 Exactly Once

必须要保证每个从 Source 发出的 event 只能精准一次地影响 Sink，要求：

-   Source 端必须是可重放的
-   Sink 必须是事务化或幂等的

## 4.1 Streams

`Stream`流，就是流式处理中的基本概念。虽然流数据分为各种不同特征的类型，但是 Flink 可以巧妙高效的进行处理：

### 4.1.1 有界流和无界流

![](https://img-blog.csdn.net/20180928213705657?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

Flink 的设计哲学最擅长于处理无界数据，但是对于有界数据（其实就是批处理）也提供了高效的操作方式。

### 4.1.2 实时流和记录型流

一般来说处理这类流数据的方式有两种：

-   数据一旦生成就立刻实时处理
-   先将流数据持久化到存储系统（如文件系统等），稍后再处理它们。

### 4.1.3 Record

数据流或数据集的基本组成元素就是 Record。

算子和函数将 Record 作为输入和输出。

## 4.2 State

见[这里](#state)

## 4.3 Time

![](https://img-blog.csdnimg.cn/20181227223905760.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

### 4.3.1 概述

Time 是流式应用状态中的一个重要概念。

大多流式数据本身就有时间语义，因为每个事件都是在特定的时间点上产生的。而且通常流式计算是以时间为基础的。在流式处理中很重要的方面是应用程序如何去测量时间即`event-time` 和`processing-time`的差别。

在流处理中设定使用哪种时间，请参考[Flink 学习 3-API 介绍 - DataStream - 处理时间设定](https://blog.csdn.net/baichoufei90/article/details/82891909#datastream)。这个设定决定了 Source 怎么表现（比如是否分配 timestamp）以及时间窗口算子使用哪种时间作为计算标准。

### 4.3.2 Event Time

EventTime 是每个单独的 Event 在其生产设备上发生的时间，比如 APP 向服务端上报事件时在 APP 内该事件的发生时间，在 APP 内就会被嵌入到该 event 中。服务端可以从该 event 中提取 EventTime 时间戳。

说白了，EventTime 是事件在现实世界中发生的时间，在之前就产生，和后来到达的 Flink 服务端时间无关。Flink 可从每条记录中提取`timestamp`。

处理具有 EventTime 语义的流的应用程序，是基于**事件的时间戳**来计算结果。 因此，无论是处理记录型还是实时的流事件，通过`EventTime`处理都会得到准确和一致的结果。但由于事件往往乱序到达，所以不可能无线等待，只能等待有限时间。只要在有限等待时间内所有数据都达到 Flink 系统，则基于 EventTime 的处理就会产生正确和一致性结果，就算是乱序、事件迟到、重跑历史数据（比如一个月的 Kafka 历史数据，ProcessingTIme 可能就只有几秒区间，这时就适合用 EventTime 处理）等场景。比如有 1 小时的 EventTime 时间窗口，则只要事件的时间戳落入该窗口时间，则不管顺序如何、何时被处理，都能被 Flink 正确处理。

当选择 EventTime 模式时有两个重要概念：

-   水位 WaterMark  
    当使用 EventTime 时，程序中必须使用指定生成 EvnetiTime WaterMark 的方式。Flink 将使用水位来推断 EventTime 应用中的时间，也就是说基于 EventTime 时必须指定如何生成 WaterMark。

    此外，水位也是一种灵活的机制，可以在结果的延迟和完整性间做出权衡。
-   延迟数据处理  
    当使用水位在`event-time`模式下处理流时，可能发生在所有相关事件到达之前就已完成了计算，这类事件称为延迟事件。 Flink 具有多种处理延迟事件的选项，如重新路由它们以及更新此前已经完成的结果。

请注意，有时当 EventTime 程序实时处理在线数据时，它们将使用一些 ProcessingTime 操作，以确保其及时进行。

### 4.3.3 Ingestion Time

数据到达 Flink 系统时间。

具体来说，是在 Source 算子处将每条记录都是用当前时间作为时间戳，后续时间算子就用该时间戳。

对比 Processing Time，Ingestion Time 代价较昂贵，但结果更可预测。因为 Ingestion Time 统一在 source 处理时一次性分配时间戳，因此对事件的时间窗口操作不会再有时间分配，所以不会受到如处理算子所在机器时间不同步或算子传输时网络延时造成计算结果不准确的问题；而 Processing Time 可能会有延时导致不准确的情况。

对比 Event Time，Ingestion Time 不能处理乱序事件和迟到的情况，但优势是可以不指定水印。

**注意，在 Flink 内部实现中其实是将 Ingestion Time 当做类似 Event Time 来处理，但时间戳的分配和水印生成是 Flink 系统自动处理的，对用户透明。** 

### 4.3.4 Processing Time

-   概念  
    Processing Time 指的是数据处理算子所在的机器的系统时钟时间。具体来说，流计算使用 Processing Time 时，所有基于时间的算子（如 time windows）会使用具体算子所在的机器的系统时钟时间。
-   整点时间窗口区间  
    小时级的 Processing Time Window 将包括系统时钟指示**整点小时**之间到达特定算子的所有记录。例如，如果一个应用程序在 9:15 am 开始运行，则第一个每小时处理时间窗口将包括在 9:15 am 和 10:00 am 之间处理的事件，下一个窗口将包括在 10:00 am 和 11:00 am 之间处理的事件，依此类推。
-   优点  
    Processing Time 是 Flink 默认采用的时间，不需要流和机器之间的协调，编码实现最为简单，它提供了最佳的性能和最低的延迟。
-   缺点  
    但也必须容忍不确定性（比如数据乱序、迟到、各节点时间不同步等）以及并不精确、近似的结果。
-   适用场景  
    Processing Time 语义适用于具有严格的低延迟要求，但能容忍一定的不准确结果的应用。

## 4.4 窗口

### 4.4.1 概述

大体来说 ，Flink 的窗口分为两类：

-   Time Window  
    以某一指定的时间长度来衡量窗口大小
-   Count Window  
    以某一指定的 Event 数量来衡量窗口大小  
    ![](https://img-blog.csdnimg.cn/20190227111949915.png)
    Flink 的窗口又能细分为：
-   翻滚窗口（Tumbling Window，无重叠）
-   滑动窗口（Sliding Window，有重叠）
-   会话窗口（Session Window，活动间隙）

而滑动窗口与滚动窗口的最大区别就是滑动窗口有重复的计算部分。

### 4.4.2 Tumbling Window - 滚动窗口

![](https://img-blog.csdnimg.cn/20181227201714577.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

#### 4.4.2.1 概述

-   滚动窗口分配器将每个元素分配给固定大小的窗口
-   每个窗口之间无重叠部分
-   窗口大小可按 Time 或 Event Count 衡量
-   如果滚动窗口的大小指定为 5 分钟，则将每 5 分钟启动一个新窗口，如上图所示

#### 4.4.2.2 源码

见[Flink - 时间窗口源码分析](https://blog.csdn.net/baichoufei90/article/details/105603180)

#### 4.4.2.3 实例

```java
val input: DataStream[T] = ...


input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>)


input
    .keyBy(<key selector>)
    .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>)


input
    .keyBy(<key selector>)
    
    .window(TumblingEventTimeWindows.of(Time.days(1), Time.hours(-8)))
    .<windowed transformation>(<window function>)

```

### 4.4.3 Sliding Window - 滑动窗口

![](https://img-blog.csdnimg.cn/2018122720191638.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

#### 4.4.3.1 概述

-   滑动窗口分配器将所有元素分配给固定大小的窗口
-   滑动窗口有两个参数：
    1.  窗口大小
    2.  滑动间隔（步长）
-   有重叠  
    因此，如果滑动间隔小于窗口大小，那么滑动窗口会有重叠部分。此时，元素会被分配到多个窗口。
-   窗口大小可按 Time 或 Event Count 衡量
-   适合求最近时间的统计，比如 BI 常用这种模式窗口

例如，滑动窗口两个参数为 (10 分钟, 5 分钟)。这样，每 5 分钟会生成(滑动) 一个窗口，包含生成时往前推 10 分钟内到达的事件，每次有 5 分钟时间内的数据重叠，如下图所示。

#### 4.4.3.2 实例

```java
val input: DataStream[T] = ...


input
    .keyBy(<key selector>)
    .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>)


input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>)


input
    .keyBy(<key selector>)
    
    .window(SlidingProcessingTimeWindows.of(Time.hours(12), Time.hours(1), Time.hours(-8)))
    .<windowed transformation>(<window function>)

```

### 4.4.4 Session Window - 会话窗口

![](https://img-blog.csdnimg.cn/20181227201928599.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

#### 4.4.4.1 概述

-   Session Window 分配器通过`activity session`来对事件进行分组。
-   无固定的时间窗口大小对齐
-   起点和终点不固定  
    没有固定的开始和结束时间。相反，当会话窗口在一段时间内没有接收到元素时会关闭，例如，不活动的间隙时。
-   无重叠  
    与滚动窗口和滑动窗口相比，会话窗口不会重叠
-   Gap 可配  
    会话窗口分配器配置会话间隙，定义所需的不活动时间长度 (defines how long is the required period of inactivity)。当此时间段到期时，当前会话关闭，后续元素被分配到新的会话窗口。
-   Gap 可配静态固定的或动态提取的  
    用来定义 gap 周期长度。当 gap 过期后，当前 session 关闭，后来的事件分配到下一个 SessionWindow.
-   实现原理  
    SessionWindow 算子为每个到来的事件都创建一个新的 Window，只要符合规则就 merge 到一个窗口。一个 SessionWindow 算子为了被 merge，需要一个[Merge Trigger](https://ci.apache.org/projects/flink/flink-docs-release-1.11/zh/dev/stream/operators/windows.html#triggers)和[Window Function](https://ci.apache.org/projects/flink/flink-docs-release-1.11/zh/dev/stream/operators/windows.html#window-functions)（如 ReduceFunction, AggregateFunction, ProcessWindowFunction ，而 FoldFunction 不能被合并.)
-   适合线上用户行为分析  
    如分析某个登录用户一段时间内的行为统计

#### 4.4.4.2 实例

```java
val input: DataStream[T] = ...


input
    .keyBy(<key selector>)
    
    .window(EventTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>)


input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withDynamicGap(new SessionWindowTimeGapExtractor[String] {
      override def extract(element: String): Long = {
        
      }
    }))
    .<windowed transformation>(<window function>)


input
    .keyBy(<key selector>)
    .window(ProcessingTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>)



input
    .keyBy(<key selector>)
    .window(DynamicProcessingTimeSessionWindows.withDynamicGap(new SessionWindowTimeGapExtractor[String] {
      override def extract(element: String): Long = {
        
      }
    }))
    .<windowed transformation>(<window function>)

```

### 4.4.5 Global Window - 全局窗口

![](https://img-blog.csdnimg.cn/20200822001721140.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

#### 4.4.5.1 概述

使用全局窗口时，会将所有拥有相同 key 的元素分配到相同的单个全局窗口中。

仅当您还指定自定义触发器时，使用全局窗口才有意义；否则，将不会执行任何计算，因为全局窗口没有可以处理聚合元素的自然尾端。

#### 4.4.5.2 实例

```java
val input: DataStream[T] = ...

input
    .keyBy(<key selector>)
    .window(GlobalWindows.create())
    .<windowed transformation>(<window function>)

```

## 4.5 Watermarker(水印 / 水位) 和 EventTime

请点击[Flink - 水位](https://blog.csdn.net/baichoufei90/article/details/108164777)

## 4.6 触发器

触发器决定在窗口的什么时间点上启动用户定义的数据处理任务。

触发器意义是解决水位迟到、早到引起的问题。

## 4.7 Flink 数据类型

-   [Flink 的类型分类](https://www.cnblogs.com/qcloud1001/p/9626462.html)
-   [Flink 的数据类型和序列化](https://www.jianshu.com/p/e8bc484fa4c5)
-   [flink 类型系统 TypeIinformation](https://www.jianshu.com/p/e62ee7f4fbb1)

## 4.8 转换

## 4.9 [Streaming 程序容错和一致性语义](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/connectors/guarantees.html)

根据 Source 是否参与 Checkpoint 快照机制，不同 Source 时更新用户定义的 State 有不同语义：  
![](https://img-blog.csdnimg.cn/20200804102231894.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

而若需要在消费端 exactly-once 基础上进一步实现端到端的 exactly-once，那就需要 Sink 端也参与到 Checkpoint，在已保证 Source 状态 exactly-once 语义的前提下，不同 Sink 的交付保证如下：  
![](https://img-blog.csdnimg.cn/20200804102926380.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

-   比如 HDFS Sink 就是两阶段提交，开始 Checkpoint 快照就将当前写入的文件变为 PENDING，待通知 Checkpoint 成功后就变为 FINISHED 对读可见，这样就能实现精确一次的语义了。

## 4.10 [Batch 程序容错](https://ci.apache.org/projects/flink/flink-docs-release-1.11/concepts/stateful-stream-processing.html#state-and-fault-tolerance-in-batch-programs)

可参考:

-   [stateful-stream-processing](https://ci.apache.org/projects/flink/flink-docs-release-1.11/concepts/stateful-stream-processing.html)
-   [Checkpointing](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/stream/state/checkpointing.html)
-   [Checkpointing - 中文版](https://ci.apache.org/projects/flink/flink-docs-release-1.10/zh/dev/stream/state/checkpointing.html)
-   [Data Streaming Fault Tolerance-Flink 流计算容错机制内部的技术原理](https://ci.apache.org/projects/flink/flink-docs-release-1.10/internals/stream_checkpointing.html)
-   [Queryable State Beta](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/stream/state/queryable_state.html)
-   [Flink’s state backends](https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/state/state_backends.html)

## 5.1 状态

### 5.1.1 概述

可参考

-   [Flink State 误用之痛，竟然 90% 以上的 Flink 开发都不懂](https://mp.weixin.qq.com/s/0mWNi45_bs2lIC2LsQ_wUA)  
    从性能和 TTL 两个维度来描述 ValueState 中存 Map 与 MapState 有什么区别？

    如果不懂这两者的区别，而且使用 ValueState 中存大对象，生产环境很可能会出现以下问题：

    -   CPU 被打满
    -   吞吐上不去
-   [Flink 源码：从 KeyGroup 到 Rescale](https://mp.weixin.qq.com/s?__biz=MzkxOTE3MDU5MQ==&mid=2247484339&idx=1&sn=c83b5fc85f6abadaa7ac94f08c571b31&scene=21#wechat_redirect)  
    阅读本文你能 get 到以下点：

    -   KeyGroup、KeyGroupRange 介绍
    -   maxParallelism 介绍及采坑记
    -   数据如何映射到每个 subtask 上？
    -   任务改并发时，KeyGroup rescale 的过程

有状态和无状态：

-   每个有价值的流式应用一般都是有状态的。Flink 中的状态是指中间状态，如算子的状态。**该状态托管到 Flink 系统内部**。  
    有状态算子需要跨多个事件记录信息用于后续计算，比如 Window 相关算子  
    ![](https://img-blog.csdnimg.cn/20201109180002773.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)
-   对应无状态的就比如函数，输入到输出无状态。

可举例使用状态场景如下：  
![](https://img-blog.csdnimg.cn/20201109180633529.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

Flink 状态核心概念如下：

-   ManagedState 和 RawState  
    ![](https://img-blog.csdnimg.cn/20201109180856943.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)
-   Operator State  
    算子状态，一般存于内存
-   Keyed State  
    KeyedState 保存在内嵌的类 KeyValue 存储内，它是一种由 Flink 管理的分片式 key value 存储。  
    ![](https://img-blog.csdnimg.cn/20201109181335790.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)
-   State Backend  
    决定状态怎样去存、存在哪，可配置。

有状态计算：  
![](https://img-blog.csdnimg.cn/20200924230101928.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

-   Spark 的做法是，在调用算子之前就提取状态，然后和数据一起调用无状态算子，最大的好处就是使用无状态算子就能实现有状态计算。
-   Flink 的做法是，某些需要状态的算子，可将状态保存在 StateBackend，数据调用算子时可以使用算子含有的状态，对计算结果造成影响。

运行基本业务逻辑的任何应用程序都需要记下事件或中间结果，以便在以后的时间点访问它们：例如在收到下一个事件时或在特定持续时间之后。  
![](https://img-blog.csdn.net/20180928215143852?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

应用程序的 State(状态) 是很重要的一个概念，Flink 中有很多 feature 来处理状态。

-   多种多样的状态元语  
    Flink 有多种数据结构来提供状态原语，例如原子值、列表或映射，我们可以根据这个 function 访问方式来选择合适的状态元语类型。
-   可插拔的`StateBackend`  
    应用的状态是由一个可插拔的 StateBackend 服务管理和设置检查点的，我们可以选择用内存或`RocksDB`（一个高效的嵌入式磁盘数据存储）甚至是自定义的状态后端插件来存储应用状态。
-   精准一次 ExactlyOnce 的状态一致性  
    Flink 的检查点和恢复算法可以保证在失败时应用状态的一致性。所以，这些失败对应用来说是透明的，不会影响正确性。
-   巨大的状态信息维护  
    Flink 通过异步和增量检查点算法可维护 TB 级别的应用状态。
-   可扩展的应用  
    Flink 通过对应用弹性分配 worker 数量来实现应用可扩展。
-   维护跨并行 Task 实例的状态
-   Managed State  
    Managed State 描述了已在 Flink 注册的应用程序的托管状态。

    Apache Flink 会负责 Managed State 的持久化和重伸缩等工作。

### 5.1.2 Flink State 分类

![](https://img-blog.csdnimg.cn/20200925103131497.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

-   KeyedState 是一种由 Flink 管理的分片式 key value 存储，详见 5.1.3
    -   ValueState  
        就是一个 Key 对应的一个 State，可向其包装的任何变量，以此添加容错功能。

        ValueState 是一个包装类型，有三个重要方法：

        -   update  
            设置状态
        -   value  
            获取当前值。在初始或 clear 后该方法返回`null`
        -   clear  
            删除内容
    -   ListState  
        就是一个 Key 对应的一个 State List，是状态多值 ListState。

        可参考`org.apache.flink.streaming.api.functions.sink.filesystem.StreamingFileSink`的`ListState<byte[]> bucketStates` 和 `ListState<Long> maxPartCountersState`
    -   MapState  
        一个 Key 对应的一个 Map State
-   OperatorState

### 5.1.3 Keyed State

#### 5.1.3.1 概念

KeyedState 保存在内嵌的类 KeyValue 存储内，它是一种由 Flink 管理的分片式 key value 存储，有两个特点：

-   只能应用于 KeyedStream 的函数与操作中，例如 Keyed UDF, window state  
    Flink 严格地将 KeyedState 与有 KeyedState 的运算符读取的流一起进行分区和分发。因此，只有 KeyedStream 才能访问 KeyValue 的 Keyed State，并且仅限于与当前事件的键关联的值，即在整个程序中没有`keyBy`的过程就没有办法使用 KeyedStream。
-   keyed state 是已经分区 / 划分好的，每一个 key 只能属于某一个 keyed state  
    每个 Key 对应一个 State，即一个 Operator 实例处理多个 Key，访问相应的多个 State，并由此就衍生了 Keyed State。

对齐流键和状态键可确保所有状态更新都是本地操作，从而确保了一致性而没有事务开销。 这种对齐方式还允许 Flink 重新分配状态并透明地调整流分区。  
![](https://img-blog.csdnimg.cn/2020080323444463.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

-   每个 Key 映射到一个有状态算子实例保存
-   每个 Operator 一般有多个并行实例，同 key 数据由同一个 Operator 实例处理
-   一个 KeyedState 只会对应一个 Operator 实例，由该实例所在 TM 负责保存，但一个 Operator 实例会有多个 Keyed state（因为一个 Operator 实例有多个 Key，每个 Key 一个 Keyed State）
-   KeyedState 又被组织为`KeyGroup`，是 Flink 重分布 KeyedState 的原子单元。  
    KeyGroup 的个数和 Job 的 task 最大并行度相同。每个 KeyedOperator 实例会处理一个或多个 KeyGroup。

#### 5.1.3.2 分类

![](https://img-blog.csdnimg.cn/2020110918285688.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/20201109200539935.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

-   ValueState  
    存 Key 对应的状态单值。

    可以通过 update 方法更新状态值，通过 value() 方法获取状态值。

    如 WordCount 可用 Word 当 Key，Count 存为该 Word 对应的 State。
-   MapState  
    类似 Java 中的 Map，由键值对状态组成。

    **需要注意的是在 MapState 中的 key 和 Keyed state 中的 key 不是同一个。** 
-   ListState  
    类似 Java 中的 List，一个 Key 含有多个状态值。

    可以通过 add 方法往列表中附加值；也可以通过 get() 方法返回一个 Iterable 来遍历状态值。
-   ReducingState  
    状态为单值，存储用户传入的 reduceFunction 的 reduce 计算结果状态。

    每次调用`add`方法添加值的时候，会调用 reduceFunction，最后合并到一个单一的状态值。
-   AggregatingState  
    `AggregatingState`与`ReducingState`的区别是：ReducingState 中 `add(T)`和 `T get()`的泛型元素为同类型，但在 AggregatingState 输入的 `IN`，输出的是 `OUT`。

#### 5.1.3.3 并行度改变时

Keyed State 随 Key 在实例间迁移 Redistribute。

![](https://img-blog.csdnimg.cn/20201119151925288.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

#### 5.1.3.4 API 使用

##### 5.1.3.4.1 例子 1

![](https://img-blog.csdnimg.cn/20201207112922540.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

-   `keyBy`创建 keyed stream 对 key 进行划分，这是使用 keyed state 的基本前提！
-   sum 方法会调用内置的 StreamGroupedReduce 实现

![](https://img-blog.csdnimg.cn/20201207115948545.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

##### 5.1.3.4.2 官方反欺诈例子

1.  main 方法：

```java

val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment





val transactions: DataStream[Transaction] = env
  .addSource(new TransactionSource)
  
  .name("transactions")


val alerts: DataStream[Alert] = transactions
  
  .keyBy(transaction => transaction.getAccountId)
  
  
  .process(new FraudDetectorV2)
  .name("fraud-detector")


alerts
  
  .addSink(new AlertSink)
  .name("send-alerts")

```

![](https://img-blog.csdnimg.cn/20201102172119836.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

3.  KeyedState 只能用于 \`RichFunction\`\`\`\`java

    class FraudDetectorV2 extends KeyedProcessFunction[Long, Transaction, Alert]

    ```

    ```
4.  将 State 声明为实例级别变量 \`\`\`java
    @transient private var flagState: ValueState[java.lang.Boolean] = \_

    ```

    ```
5.  在实现自`RichFunction`的`open`方法中创建 State 描述符，并为 State 赋值 \`\`\`java

     @throws[Exception]
     override def open(parameters: Configuration): Unit = {

       val flagDescriptor = new ValueStateDescriptor("flag", Types.BOOLEAN)

       flagState = getRuntimeContext.getState(flagDescriptor)

       val timerDescriptor = new ValueStateDescriptor("timer-state", Types.LONG)
       timerState = getRuntimeContext.getState(timerDescriptor)
     }

    ```

    ```
6.  读写 State\`\`\`java

    val lastTransactionWasSmall = flagState.value

    flagState.update(true)

    flagState.clear()

    ```

    ```

### 5.1.4 Operator State

#### 5.1.4.1 概述

-   Operator State 又称为`non-keyed state`，每一个 operator state 都仅与一个 operator 的实例绑定。
-   常见的 operator state 是`source state`，例如记录当前 KafkaSource consumer 的 offset

Operator State 可用于所有算子，常用于 Source，例如 `FlinkKafkaConsumer`.

```java
public abstract class FlinkKafkaConsumerBase<T> extends RichParallelSourceFunction<T> implements
		CheckpointListener,
		ResultTypeQueryable<T>,
		CheckpointedFunction

```

#### 5.1.4.2 分类

-   ListState
-   UnionList State
-   BroadState State

#### 5.1.4.3 并行度改变时

#### 5.1.4.4 API 使用

需要自己实现 `CheckpointedFunction` 或 `ListCheckpointed` 接口.

##### 5.1.4.4.1 例 1

![](https://img-blog.csdnimg.cn/20201207115817805.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

`fromElements`会调用`FromElementsFunction`的类，其中就使用了类型为 list state 的 operator state  
![](https://img-blog.csdnimg.cn/20201207115846132.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

##### 5.1.4.4.1 例 2

```java
public abstract class FlinkKafkaConsumerBase<T> extends RichParallelSourceFunction<T> implements
		CheckpointListener,
		ResultTypeQueryable<T>,
		CheckpointedFunction {
	
	@Override
	public final void initializeState(FunctionInitializationContext context) throws Exception {

		OperatorStateStore stateStore = context.getOperatorStateStore();

		this.unionOffsetStates = stateStore.getUnionListState(new ListStateDescriptor<>(OFFSETS_STATE_NAME,
			createStateSerializer(getRuntimeContext().getExecutionConfig())));
	
		if (context.isRestored()) {
			
			restoredState = new TreeMap<>(new KafkaTopicPartition.Comparator());

			
			for (Tuple2<KafkaTopicPartition, Long> kafkaOffset : unionOffsetStates.get()) {
				restoredState.put(kafkaOffset.f0, kafkaOffset.f1);
			}

			LOG.info("Consumer subtask {} restored state: {}.", getRuntimeContext().getIndexOfThisSubtask(), restoredState);
		} else {
			LOG.info("Consumer subtask {} has no restore state.", getRuntimeContext().getIndexOfThisSubtask());
		}
	}
	
	@Override
	public final void snapshotState(FunctionSnapshotContext context) throws Exception {
		if (!running) {
			LOG.debug("snapshotState() called on closed source");
		} else {
			
			unionOffsetStates.clear();
			
			final AbstractFetcher<?, ?> fetcher = this.kafkaFetcher;
			if (fetcher == null) {
				
				
				for (Map.Entry<KafkaTopicPartition, Long> subscribedPartition : subscribedPartitionsToStartOffsets.entrySet()) {
					unionOffsetStates.add(Tuple2.of(subscribedPartition.getKey(), subscribedPartition.getValue()));
				}

				if (offsetCommitMode == OffsetCommitMode.ON_CHECKPOINTS) {
					
					
					pendingOffsetsToCommit.put(context.getCheckpointId(), restoredState);
				}
			} else {
				
				HashMap<KafkaTopicPartition, Long> currentOffsets = fetcher.snapshotCurrentState();

				if (offsetCommitMode == OffsetCommitMode.ON_CHECKPOINTS) {
					
					
					pendingOffsetsToCommit.put(context.getCheckpointId(), currentOffsets);
				}
				
				
				for (Map.Entry<KafkaTopicPartition, Long> kafkaTopicPartitionLongEntry : currentOffsets.entrySet()) {
					unionOffsetStates.add(
							Tuple2.of(kafkaTopicPartitionLongEntry.getKey(), kafkaTopicPartitionLongEntry.getValue()));
				}
			}

			if (offsetCommitMode == OffsetCommitMode.ON_CHECKPOINTS) {
				
				while (pendingOffsetsToCommit.size() > MAX_NUM_PENDING_CHECKPOINTS) {
					pendingOffsetsToCommit.remove(0);
				}
			}
		}
	}
	
	
	@Override
	public final void notifyCheckpointComplete(long checkpointId) throws Exception {
		if (!running) {
			LOG.debug("notifyCheckpointComplete() called on closed source");
			return;
		}

		final AbstractFetcher<?, ?> fetcher = this.kafkaFetcher;
		if (fetcher == null) {
			LOG.debug("notifyCheckpointComplete() called on uninitialized source");
			return;
		}

		if (offsetCommitMode == OffsetCommitMode.ON_CHECKPOINTS) {
			
			if (LOG.isDebugEnabled()) {
				LOG.debug("Consumer subtask {} committing offsets to Kafka/ZooKeeper for checkpoint {}.",
					getRuntimeContext().getIndexOfThisSubtask(), checkpointId);
			}

			try {
				final int posInMap = pendingOffsetsToCommit.indexOf(checkpointId);
				if (posInMap == -1) {
					LOG.warn("Consumer subtask {} received confirmation for unknown checkpoint id {}",
						getRuntimeContext().getIndexOfThisSubtask(), checkpointId);
					return;
				}
				
				
				@SuppressWarnings("unchecked")
				Map<KafkaTopicPartition, Long> offsets =
					(Map<KafkaTopicPartition, Long>) pendingOffsetsToCommit.remove(posInMap);

				
				for (int i = 0; i < posInMap; i++) {
					pendingOffsetsToCommit.remove(0);
				}

				if (offsets == null || offsets.size() == 0) {
					LOG.debug("Consumer subtask {} has empty checkpoint state.", getRuntimeContext().getIndexOfThisSubtask());
					return;
				}
				
				fetcher.commitInternalOffsetsToKafka(offsets, offsetCommitCallback);
			} catch (Exception e) {
				if (running) {
					throw e;
				}
				
			}
		}
	}
}

```

### 5.1.5 State 持久化

#### 5.1.5.1 概述

Flink 通过联合使用`StreamReplay`和`Checkpoint`来实现容错。Checkpoint 需要使用持久化存储来保存状态：

-   可回放重复消费记录的数据源  
    比如消息队列（如 Kafka）、文件系统（如 HDFS）
-   可存放状态的持久化存储  
    HDFS 等分布式文件系统

其中每次 Checkpoint 就是标记每个输入流中的特定点以及每个运算符的对应状态。通过恢复算子状态以及从 Checkpoint 开始重放数据记录，可以恢复数据流以及同时保持一致性（ExactlyOnce 处理语义）。

Flink 容错机制的具体做法就是不断给分布式数据流做 Checkpoint 快照，存放到可配的地方（通常是分布式文件系统中）。当由于如机器、网络、软件等错误导致程序出错时，就停止该数据流，并从最近一次成功的 Checkpoint 恢复和重启算子，输入流也被重置到该快照点。而在恢复过程中被重复消费的数据会被保证不造成影响。

而 Checkpoint 的间隔时间，就是一种在执行中用于 Checkpoint 的开销和恢复时间的权衡方法：

-   间隔越长，恢复时的记录数越多，恢复耗时更长
-   间隔越短，一定时间内用于 Checkpoint 的时间百分比越长，恢复耗时更短

#### 5.1.5.2 KeyedState 和 OperatorState Checkpoint 区别

状态从本质上来说，是 Flink 算子子任务的一种本地数据，为了保证数据可恢复性，使用 Checkpoint 机制来将状态数据持久化输出到存储空间上。状态相关的主要逻辑有两项：

-   将算子子任务本地内存数据在 Checkpoint 时将状态快照后持久化到状态后端 StateBackend；
-   初始化或重启应用时，以一定的逻辑从 StateBackend 中读出对应状态数据，并转为算子子任务的本地内存数据。

注意：

-   Keyed State 对这两项内容做了更完善的封装，开发者可以开箱即用，也就是说只需要注册和使用这类状态，不需要管 Checkpoint 快照和恢复，Flink 已经帮你做了。
-   对于 Operator State 来说，每个算子子任务管理自己的 Operator State，或者说每个算子子任务上的数据流共享同一个状态，可以访问和修改该状态。

    但 Flink 的算子子任务上的数据在程序重启、横向伸缩等场景下不能保证百分百的一致性。换句话说，重启 Flink 应用后，某个数据流元素不一定会和上次一样，还能流入该算子子任务上。因此，我们需要根据自己的业务场景来设计 snapshot 和 restore 的逻辑。为了实现这两个步骤，Flink 提供了最为基础的 CheckpointedFunction 接口类。

```java
public interface CheckpointedFunction {

  
    void snapshotState(FunctionSnapshotContext context) throws Exception;

  
    void initializeState(FunctionInitializationContext context) throws Exception;

}

```

## 5.2 Checkpoint

可参考：

-   [Checkpointing 使用](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/stream/state/checkpointing.html)
-   [Checkpoints 部署和恢复](https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/state/checkpoints.html)

### 5.2.1 概述

#### 5.2.1.1 Chandy Lamport

![](https://img-blog.csdnimg.cn/2020120718070850.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

Chandy Lamport 算法，可以想象就是对水管吹气，把里面的水全部吹出来，此时就是管道水排空时系统状态。

对应到 Flink，就是 CheckpointCoordinator 定时对 Source 发送 Checkpoint Barr

#### 5.2.1.2 怎么保证精确一次容错？

#### 5.2.1.3 到底是什么的状态？

由于 Flink 中每个函数和算子都可以是有状态的，有状态函数可以在处理每个事件过程中存储数据，这使状态成为任何类型更复杂操作的关键构成。

Flink 容错性的核心部分就是为分布式数据流和算子状态制作 Checkpoint 一致性快照，可以在出错时从 Checkpoint 中恢复状态和执行位置等。

相关详细内容还可参考

-   [论文 - Lightweight Asynchronous Snapshots for Distributed Dataflows](https://arxiv.org/pdf/1506.08603.pdf)  
    描述 Flink 快照机制实现原理
-   [Chandy-Lamport algorithm](http://research.microsoft.com/en-us/um/people/lamport/pubs/chandy.pdf)  
    Flink 快照实现参照的算法

#### 5.2.1.4 Checkpoint 持久化了什么？

Checkpoint 需要使用持久化存储来保存快照状态：

-   可回放重复消费记录的数据源  
    比如消息队列（如 Kafka）、文件系统（如 HDFS）
-   可存放状态的持久化存储  
    HDFS 等分布式文件系统

状态快照是指 Flink Job 的全局一致性镜像，一个快照包括：

-   一个指向每个 DataSource 的指针（如 Kafka 的 offset）
-   每个有状态算子的状态副本，该副本是处理了 sources offset 之前所有的事件后而生成的状态。

### 5.2.2 Checkpoint 异步性

Checkpoint 为异步执行的，Checkpoint 屏障不会在 lock 步骤中传播，并且算子可以异步地为其状态制作快照。

Flink 的 StateBackend 利用写时复制（copy-on-write）机制允许当异步生成旧版本的状态快照时，能够不受影响地继续流处理。

只有当快照被持久保存到 StateBackend 后，这些旧版本的状态才会被当做垃圾回收。

### 5.2.3 Checkpoint Aligned 和 Unaligned

Flink1.11 以前只支持对齐的 Checkpoint，从 1.11 开始也可启用不对齐的 Checkpoint 了。

-   对齐的 Checkpoint  
    每个算子需要等到已接收所有上游发送的 Barrier 对齐后才可以进行 本算子状态的 Snapshot ，完成后继续向后发送 Barrier。这样，在出现反压的情况下，Barrier 从上游算子传送到下游算子可能需要很长的时间，从而导致 Checkpoint 超时的问题。
-   非对齐的 Checkpoint  
    针对这一问题，Flink 1.11 新增了 `Unaligned Checkpoint`机制，开启后一旦收到第一个上游 Barrier 就可以开始执行 Checkpoint，并把上下游之间正在传输的数据也作为状态保存到快照中，这样 Checkpoint 的完成时间大大缩短，不再依赖于算子的处理能力，解决了反压场景下 Checkpoint 可能超时的问题。

    可以通过 `env.getCheckpointConfig().enableUnalignedCheckpoints();`开启`unaligned Checkpoint`。

### 5.2.4 Checkpoint 配置

```java
val env = StreamExecutionEnvironment.getExecutionEnvironment()

```

-   开启 Checkpoint  
    Checkpoint 默认关闭。需要用`env.enableCheckpointing(n 毫秒)`来开启，这里的 n 指开启两次 checkpoint 之间的间隔毫秒数时间。

    ```java
    env.enableCheckpointing(1000)

    ```
-   exactly once / at least once 语义  
    默认为`exactly once`:

    ```java
    env.getCheckpointConfig.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE)

    ```
-   Checkpoint 超时时间  
    如果某次 Checkpoint 超过此阈值还没完成，则将进行中的 Checkpoint 干掉作废，单位毫秒

    ```java
    env.getCheckpointConfig.setCheckpointTimeout(60000)

    ```
-   下次 Checkpoint 距离上一次 Checkpoint 完成后的最小时间间隔毫秒数。  
    注意，此时不会管已设置的 Checkpoint 时间间隔和每次 Checkpoint 持续时间。也就是说，此时间隔时间永不会比本参数时间还小了。**还有一点就是，此时并发 Checkpoint 数设置失效，强制设为 1。** 

    ```java
    env.getCheckpointConfig.setMinPauseBetweenCheckpoints(1500);

    ```

    比如，我们通过`env.enableCheckpointing`设定了是 2 个两次 Checkpoint 之间的间隔毫秒数时间为 1000ms。那么如果一个 Checkpoint 耗时 900ms，本来过 100ms 就应做下一个 Checkpoint，导致 checkpoint 过于频繁。这个时候，本设置让 Checkpoint 完成之后最少要等 500ms 才开始下一个，可以防止 Checkpoint 太过于频繁而导致业务处理的速度下降。

    所以有时候因为某次 Checkpoint 时间过长，可能导致采用时间间隔方式受影响导致上一个 Checkpoint 刚完成没多久又开始下一个 Checkpoint，可采用本方式。
-   Checkpoint 并发数  
    默认为 1，即只能有 1 个 Checkpoint 同时进行。一个没完另一个不会触发。减少对 Chcekpoint 开销，不会对系统产生过多影响。

    ```java
    env.getCheckpointConfig.setMaxConcurrentCheckpoints(1)

    ```

    对于这种场景可以用并发 Checkpoint：有确定的处理延迟（比如经常调用较好时的外部服务接口），但仍希望频繁 Checkpoint，以期当发生错误时恢复很少的数据开始处理。
-   Checkpoint 失败后是否导致 flink 任务失败  
    默认为 true。

    ```java
    env.getCheckpointConfig.setFailOnCheckpointingErrors(false)

    ```

    注：本方法已经废弃，建议改用以下方法，默认 0 即不容忍任何一个 Checkpoint 失败：

    ```java
    env.getCheckpointConfig.setTolerableCheckpointFailureNumber(int)

    ```
-   是否开启外部检查点  
    默认情况下，Checkpoint 在默认的情况下仅用于恢复失败的作业，而并不会被保留，即当程序 Cancel 时 Checkpoint 就会被删除。当然，你可以通过配置来保留 checkpoint，这些被保留的 checkpoint 在作业 Fail 或 Cancel 时不会被清除。

    我们可以将周期性的 Checkpoint 配置为保存到外部存储。这种检查点会将元数据持久化到外部存储，而且即使 job 失败也不会自动清除。这样，当 job 失败时可直接从这个现成的 Checkpoint 恢复。详细参考:[Externalized checkpoints 的部署文档](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/state/checkpoints.html#externalized-checkpoints)

    有两种选择：

    -   ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION  
        当 Cancel 该 job 时也保留 Checkpoint。也就是说，我们必须在 Cancel 后需要手动删除 Checkpoint 文件。
    -   ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION  
        当 Cancel 该 job 时删除 Checkpoint。仅当 Job 失败时，Checkpoint 才会被保留。

        为单个 job 设置 Externalized 方法：

    ```java
    env.getCheckpointConfig().enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);

    ```

    在 flink-conf.yaml 中全局设置方法：

    ```xml
    execution.checkpointing.externalized-checkpoint-retention:RETAIN_ON_CANCELLATION

    ```
-   优先从 Checkpoint 恢复而不是 Savepoint  
    默认为 false。该属性确定 job 是否在最新的 checkpoint 回退，即使有更近的可以潜在地减少恢复时间的 savepoint 可用（因为 checkpoint 恢复比 savepoint 恢复更快）

    ```java
    env.getCheckpointConfig().setPreferCheckpointForRecovery(true);

    ```

### 5.2.5 Checkpoint 原理

Checkpoint 由运行在 JM 的组件`CheckpointCoordinator`定时触发：

```java

long baseInterval = chkConfig.getCheckpointInterval();
if (baseInterval < minPauseBetweenCheckpoints) {
	baseInterval = minPauseBetweenCheckpoints;
}

private ScheduledFuture<?> scheduleTriggerWithDelay(long initDelay) {
	return timer.scheduleAtFixedRate(
		new ScheduledTrigger(),
		
		initDelay, baseInterval, TimeUnit.MILLISECONDS);
}

private final class ScheduledTrigger implements Runnable {

	@Override
	public void run() {
		try {
			triggerCheckpoint(true);
		}
		catch (Exception e) {
			LOG.error("Exception while triggering checkpoint for job {}.", job, e);
		}
	}
}

```

#### 5.2.5.1 CheckpointBarrier

分布式快照的核心组件被称为`stream barrier`，他们会被注入到数据流中和数据记录一起在算子之间流动，并且他们永不会超过记录，因为他们是按严格线性流动的。

Barrier 将记录分为流入当前快照和下一个快照的两部分，且 Barrier 会记录下当前快照 ID。`Checkpoint n` 将包含每个算子的状态，这些状态是对应的算子消费了严格在`Checkpoint barrier n`之前的所有事件，并且不包含在此后的任何事件后而生成的状态。  
![](https://img-blog.csdnimg.cn/20200804120647453.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

1.  JM 的 CheckpointCoordinator 定时触发 Checkpoint 流程
2.  CheckpointBarrier 被注入到 Source 阶段的并行数据流中，第 n 个快照的注入点 Sn 就是 Source 覆盖数据  
    比如在 Kafka 中，注入点就是该 partition 最后一条记录的 Offset。 第 n 个快照的注入点位置 Sn 会被报告给 JobManager 的组件 CheckpointCoordinator。

    当 Job 开始做 Checkpoint barrier N 的时候可以理解为逐步将状态信息填充如下图左下角的表格。  
    ![](https://img-blog.csdnimg.cn/20201109165949649.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

    比如上图 Source 收到 Barrier N 后将 partition Offset（比如 Source 当前消费 Kafka 分区 Offset）放入 Source 状态表格中。
3.  Barrier 在注入后，和其他数据一起向数据流下游流动。  
    ![](https://img-blog.csdnimg.cn/20201209112400261.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
4.  当一个中间算子收到来自所有输入流的 Snapshot N 的对应 Barrier N 后，会将自己的状态异步写入持久化存储，并将 Barrier N 插入输出流发送给下游。

    Barrier N 流动到 Operator 1 时，就将属于 Checkpoint N 到 Checkpoint N-1 之间的所有数据反映到当前状态快照中。  
    ![](https://img-blog.csdnimg.cn/20201109171400295.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)
5.  一旦作为流程序 DAG 终点的某个 Sink 算子接收到所有输出流发来的 Barrier N 以后先填充自己的状态表格，随后就会认为快照 N 已经完成，此时会发送一个快照 N 已完成的 ACK 给 CheckpointCoordinator。  
    ![](https://img-blog.csdnimg.cn/20201109172606518.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)
6.  当 CheckpointCoordinator 收到所有 Sink 算子发出的 ACK 后，快照 N 就被认为已经成功执行完成。  
    此后就不会再访问 Sn 以前的数据了，因为认为之前的数据已经走完整个数据拓扑。
7.  最后 CheckPointCoordinator 会把整个 StateHandle 封装成 completed CheckPoint Meta，写入到 hdfs。
8.  而如果直到 Checkpoint 超时，CheckPointCoordinator 仍未收集完所有的 State Handle，CheckPointCoordinator 会认为本次 CheckPoint 失败，将这次 CheckPoint 产生的所有状态数据全部删除。

#### 5.2.5.2 Checkpoint Barrier 对齐过程

当有多个输入流时，需要对齐输入流的同 ID 的 Barrier。

当 job graph 中的每个 operator 都接收到该 Barrier 时，就会记录下自己的状态。

拥有两个输入流的 Operators（例如`CoProcessFunction`）会执行 Checkpoint Barrier 对齐（barrier alignment） ，以便当前快照能包含已消费两个输入流 barrier 之前（但不超过）的所有 event 而产生的状态。  
![](https://img-blog.csdnimg.cn/20200804131254134.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

1.  一旦算子从某个输入流收到 BarrierN，那就不能再处理任何该输入流的数据，直到从其他所有输入流收到 BarrierN。否则会导致将快照 N 和快照 N+1 的数据记录搞混。

    暂时停止处理的流的数据继续收到放入 InputBuffer 等待处理。
2.  一旦算子收集齐了 BarrerN，就会立刻发送所有 OutputBuffer 中的 PENDING 状态的记录给下游，然后发送 BarrierN。
3.  算子发送 BarrierN 完毕后，在内存中将状态制作快照，然后恢复从输入流获取、处理数据  
    注意，会优先将 InputBuffer 中的数据处理完毕，然后再从输入流处理
4.  最后，算子将内存中制作好的状态异步写入 StateBackend

注意，所有拥有多路输入流和消费来自多个上游子任务输出流的 shuffle 后的算子都需要使用 Checkpoint 对齐。

特点：

-   当若干快照并发执行时，数据流中会同时存在多个 Barrier。
-   需要注意的是，其实 Barrier 不会中断数据流，所以非常轻量级。

#### 5.2.5.3 算子状态快照

![](https://img-blog.csdnimg.cn/20200804135459676.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

-   算子状态属于快照一部分  
    只要你的算子包含任意格式的 State，那就必须作为快照一部分保存。
-   算子状态快照时间点  
    当算子接收到所有输出流的 BarrierN 后，就在内存中将状态制作快照，然后恢复从输入流获取、处理数据，最后算子将内存中制作好的状态异步写入 StateBackend。

    在这个时间点上，所有在 BarrierN 之前的对 State 的更新已经做完，而在 BarrierN 之后对 State 的更新都还没做。
-   StateBackend  
    因为状态快照可能很大，不可能全放在内存，所以被存放到可配的`StateBackend`。

    默认在 JM 内存中，但生产中一般存放在分布式存储中，比如 HDFS。

    当状态快照被存储后，算子会确认该次 Checkpoint 完成，发送该次快照的 Barrier 到输出到下游的流中。
-   状态快照包含内容

    -   对于每个并行流 Source，是快照启动时流中的 offset / 位置
    -   对于每个算子实例，是一个指向作为快照一部分的状态的指针

#### 5.2.5.4 普通 Checkpoint 恢复

当失败时，直接从最近一次成功的 Checkpoint 恢复。

恢复时：

1.  重新部署整个分布式数据流
2.  将 Checkpoint 中包含的状态给 Operator
3.  Source 会从 Checkpoint 中包含的位置 K 开始读取数据流  
    比如 Kafka 中就是记录下的偏移量 K。

#### 5.2.5.5 增量 Checkpoint 恢复

先从最新的全量快照启动算子，然后将后续的增量快照更新到算子状态上，得到最新状态。

### 5.2.6 Unaligned Checkpoint

#### 5.2.6.1 概述

Flink1.11 以前只支持对齐的 Checkpoint，从 1.11 开始也可启用不对齐的 Checkpoint 了。

-   对齐的 Checkpoint  
    每个算子需要等到已接收所有上游发送的 Barrier 对齐后才可以进行 本算子状态的 Snapshot ，完成后继续向后发送 Barrier。这样，在出现反压的情况下，Barrier 从上游算子传送到下游算子可能需要很长的时间，从而导致 Checkpoint 超时的问题。
-   非对齐的 Checkpoint  
    针对这一问题，Flink 1.11 新增了 `Unaligned Checkpoint`机制，开启后一旦收到第一个上游 Barrier 就可以开始执行 Checkpoint，并把上下游之间正在传输的数据也作为状态保存到快照中，这样 Checkpoint 的完成时间大大缩短，不再依赖于算子的处理能力，解决了反压场景下 Checkpoint 可能超时的问题。

    可以通过 `env.getCheckpointConfig().enableUnalignedCheckpoints();`开启`unaligned Checkpoint`。

特点：

-   优点  
    CheckpointBarrier 可以直接越过 in-flight 数据，所以与吞吐不再有关联
-   限制
    -   不能直接对非对齐 Checkpoint Job 进行 rescale 或修改 JobGraph，必须在 rescale 前进行 Savepoint(会对齐)
    -   目前不支持并发非对齐 Checkpoint（本模式时间更短，所以其实不太需要并发）
    -   在恢复过程中，未对齐的检查点会中断有关水印的隐含保证，参考[Unaligned checkpoints](https://ci.apache.org/projects/flink/flink-docs-release-1.12/ops/state/checkpoints.html)

#### 5.2.6.2 原理

非对齐 Checkpoint 核心思想就是只要 in-flight 缓存数据会成为 OperatorState 的一部分，那 Checkpoint 就可以超越这些数据。  
![](https://img-blog.csdnimg.cn/20200804144241223.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

1.  算子收到第一个 Barrier 后，存到输入缓存 InputBuffer 中
2.  算子对缓存中的 Barrier 立刻处理，将该 Barrier 放入输出 OutputBuffer 末尾以立刻发送给下游算子
3.  算子将所有已超越的记录（上图中超过了 3、2、1、d、c、b、a、z、y、x）标记为异步存储，并据此创建其自身状态的快照。

因此，非对齐方式的 Checkpoint 算子仅短暂停止输入流处理以标记缓冲区、发送 Barrier、创建其他状态的快照，不需要再像对齐 Checkpoint 那样等待所有所有输入流的 Barrier。

非对齐 Checkpoint 可保证 Barrier 尽快到达 Sink，所以特别适合某个流路径进展缓慢的情况，如果采用对齐 Checkpoint 甚至或延迟小时级别。

#### 5.2.6.3 Unaligned Checkpoint 恢复

1.  算子首先恢复 in-flight 数据
2.  然后再开始从上游算子消费处理数据，重新部署整个分布式数据流
3.  将 Checkpoint 中包含的状态给 Operator
4.  Source 会从 Checkpoint 中包含的位置 K 开始读取数据流  
    比如 Kafka 中就是记录下的偏移量 K。

#### 5.2.6.4 小结

-   使用场景
    -   多上游中的一个路径进展很慢  
        非对齐方式的 Checkpoint 确保 Barrier 尽快到达 Sink，特别适合至少有一个数据移动很缓慢的路径的应用，这种场景下对齐等待时间可能达到小时级。
    -   背压导致 Checkpoint 持续时间过长  
        采用本方式后，大多数情况下 Checkpoint 持续时间和端到端延迟无关了
-   不适用场景  
    但问题是会增加额外的 IO 压力，在写入 StateBackend 的 IO 是瓶颈时不适用。
-   Savepoint 只能对齐 Checkpoint，不能非对齐 Checkpoint。

### 5.2.7 Exactly Once / At Least Once / At Most Once

根据用户配置以及使用的集群，Flink 有三种语义：

-   Exactly Once  
    数据既不丢失也不重复。注意这里的含义是保证引擎管理的状态更新只提交一次到持久的后端存储，而不是引擎只处理一次该数据。

    -   Checkpoint  
        Flink 精确一次是由 Checkpoint Barrier 对齐实现的，通过这种机制，流应用程序中每个算子的所有状态都会定期一致性地做 checkpoint。一旦程序在运行中发生失败，就把所有算子都统一回滚到最后一次成功执行的全局一致 Checkpoint。回滚期间，会 STW。回滚后，Source 源也回滚到了这个 Checkpoint 时的 offset。也就是说，整个流应用程序基本上是回到最近一次的一致状态，然后程序可以从该状态重新启动。  
        ![](https://img-blog.csdnimg.cn/20201208113326460.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
        可以看到 T3 回滚后，Source offset 已经重置，`5,3`会被重新处理，但他们仍然只会精确影响状态 sum 一次，所以结果依然正确。

    如果不需要可以设置`CheckpointingMode.AT_LEAST_ONCE`，此时可以提高性能，但不再是精确一次了。

    ```
      注意：Flink通过回退和重放Source数据流，Exactly Once并不意味着每个event只被精确处理一次，而是指每个event只会精确一次地影响由Flink管理的状态。

    ```

    -   至少一次事件传递和对重复数据去重  
        对每个算子实现至少一次事件传递 + 数据去重，要求为每个算子维护一个事务日志  
        ![](https://img-blog.csdnimg.cn/20201208114139973.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
-   At Least Once  
    数据不丢，但可能重复

    具体做法是失败即从源头重试执行，但事件可能被处理多次。比如 CheckpointBarrier 不对齐时就进行快照，则可能导致先到达的那个流的数据被处理多次。  
    ![](https://img-blog.csdnimg.cn/20201208111811504.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
-   At Most Once  
    保证数据或事件最多由应用程序中的所有算子处理一次

    具体做法是不尝试从错误中恢复  
    ![](https://img-blog.csdnimg.cn/20201207231211752.png)

### 5.2.8 Checkpoint Barrier 对齐与 Exactly Once 、 At Least Once

-   At Least Once  
    Checkpoint 对齐有时候可能会增加大量耗时，对于需要超级低延时、但又要求一致性的程序，可以关掉对齐。也就是说，算子一旦看到 Checkpoint Barrier 就开始生成 Checkpoint 快照。

    当跳过 Checkpoint 对齐后，当一些输入流的 BarrierN 来到后，此时算子标记完成后可以继续处理所有输入，这就造成算子可能在 CheckpointN 完成之前又同时在处理属于 CheckpointN+1 的数据，因为这些后来的数据虽然在 Checkpoint N Barrier 之后，但会将其包含在这次 Checkpoint 备份的状态之中。

    这就造成在恢复 CheckpointN 时这些记录重复出现，因为他们既在 CheckpointN 的状态快照之中（已经影响了 Checkpoint N 记录的状态）又会在恢复时作为 Checkpoint N 之后数据的一部分被重放导致重复计算！
-   Exactly Once  
    所以要想`Exactly Once`，就要开启对齐，使用 InputBuffer 将对齐阶段继续接收的数据缓存，等待对齐完成后继续处理，此时如果从 CheckpointN 恢复时，自然其保存的状态就不会有 CheckpointN 后来的数据的干扰了。

注意：Checkpoint 对齐只能用于拥有多个前驱结点的算子（如 Join）和拥有多个后置节点的算子（repartitioning/shuaffle）。  
所以只有一个后置或前驱结点的单并行算子（如 map、flatMap、filter 等）即使在`at least once`模式也会表现为`exactly once`。

Checkpoint Barrier 对齐与不对齐例子可参考[一文搞懂 Flink 的 Exactly Once 和 At Least Once](https://ververica.cn/developers/flink-exactly-once-and-at-least-once/)

### 5.2.9 QA

转自 [一文搞懂 Flink 的 Exactly Once 和 At Least Once](https://ververica.cn/developers/flink-exactly-once-and-at-least-once/)，作者范瑞  
![](https://img-blog.csdnimg.cn/20201209135027912.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

## 5.3 Savepoint

![](https://img-blog.csdnimg.cn/20201102175415251.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

### 5.3.1 概述

Savepoint 其实就是手动触发的 Checkpoint，它依靠常规的 checkpoint 机制获取程序的快照并将其写入 StateBackend 保存。

Savepoints 允许在不丢失任何状态的情况下升级程序和 Flink 集群。

### 5.3.2 Operator ID 与 Savepoint 状态

Savepoint 内部有一个类似 map 结构，为每个有状态算子保存了`Operator ID -> State`的映射。

所以我们使用 DataStreamApi 时，尽量为每个有状态算子声明 ID，方便恢复。如果不指定会自动生成 ID，只要这些 ID 不变就能手动从 savepoint 恢复。这些 ID 取决于程序，对程序改动很敏感。

正例如下：

```java
DataStream<String> stream = env.
  
  .addSource(new StatefulSource())
  .uid("source-id") 
  .shuffle()
  
  .map(new StatefulMapper())
  .uid("mapper-id") 
  
  .print(); 

```

此时 savepoint 生成的映射如下:

```
Operator ID | State
------------+------------------------
source-id   | State of StatefulSource
mapper-id   | State of StatefulMapper

```

可以看到，只生成了有状态算子的映射。而`print`这样的无状态算子被忽略，不会成为状态一部分。

### 5.3.3 Savepoint 与 Operator 改动

-   Savepoint 后，在 Job 内增加了新有状态算子，再从 Savepoint 恢复  
    可以，心有状态算子在没有状态情况下初始化
-   Savepoint 后，在 Job 内删除了有状态算子，再从 Savepoint 恢复  
    此时会因为 Savepoint 默认会恢复状态内的所有算子给新 Job，此时会导致无法恢复，只能加上`allowNonRestoredState`跳过无法恢复的状态
-   Savepoint 后，在 Job 内对有状态算子重新排序，再从 Savepoint 恢复  
    如果有状态算子指定了 uid，则不受影响；如果没有，则可能导致有状态算子自动生成的 ID 发生变化，导致无法恢复从以前的 Savepoint 恢复
-   Savepoint 后，在 Job 内对无状态算子添加、删除或重排序，再从 Savepoint 恢复  
    如果有状态算子指定了 uid，则不受影响；如果没有，则可能导致有状态算子自动生成的 ID 发生变化，导致无法恢复从以前的 Savepoint 恢复
-   Savepoint 后，改变程序并行度，再从 Savepoint 恢复  
    没问题，恢复后使用新的并行度即可
-   可以移动 savepoint 内的文件吗？  
    目前不行，因为 metadata 文件内使用的是绝对路径。非要改的话，就编辑该文件内容。

### 5.3.4 Savepoint 对比 Checkpoint

![](https://img-blog.csdnimg.cn/20201111151929928.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

-   Checkpoint 在 Flink Job 执行期间定期在 TaskManager 节点上创建快照并生成 Checkpoint，所以在恢复时 Flink 仅需要最后成功完成的 Checkpoint 即可。也就是说，一旦成功完成了新的 Checkpoint，旧的就可以被丢弃。
-   管理者不同
    -   Savepoints 由用户触发、管理、删除，并在新的 checkpoint 完成时不会自动过期，可通过命令行或在 Cancel 一个 Job 的同时通过 REST API 来创建 Savepoint。
    -   Checkpoint 由 Flink 负责管理、删除等。
-   Checkpoint 可以使用 RocksDB 增量模式，比 Savepoint 更轻量级
-   Savepoint 的数据以标准格式输出存储，允许版本升级或配置变更时使用

### 5.3.5 原理

将 Checkpoint Barrier 手动插入到所有 Pipeline 中从而产生分布式快照。  
![](https://img-blog.csdnimg.cn/2020110917374372.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/2020110917385717.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

使用 EventTime，所以每个事件总是放入同一个窗口，保证结果一致性。

## 5.4 State Backend 状态后端

可参考：

-   [State Backends](https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/state/state_backends.html)

### 5.4.1 概述

![](https://img-blog.csdnimg.cn/20201109172833209.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/20200630160406955.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

Checkpoint 会将 timer 以及有状态算子中的状态进行一致性快照保存， 包括 Connector（比如 KafkaConnector、HDFSConnecotr 等），Window 以及任何用户自定义 State。

DataStream API 涉及到的 State 有：

-   Window 在触发计算前收集、聚合的元素
-   转换函数可能使用 key/value State 来存储值
-   转换继承`CheckpointedFunction`来 对存入 State 的值实现容错。  
    比如`StreamingFileSink`实现该接口，使用状态存储了 Bucket、Part-File，且实现了容错。

State Backend 管理的状态包括:

-   `org.apache.flink.streaming.api.datastream.KeyedStream`使用的`Keyed State`
-   实现`org.apache.flink.streaming.api.checkpoint.CheckpointedFunction`来直接管理的 State

开启 Checkpoint 后，可以将上述 State Checkpoint 持久化，防止数据丢失，出错时可一致性恢复。而具体选择的`StateBackend`就决定了 State 内部如何组织表现，以及如何、在哪进行 Checkpoint。

Checkpoint 存储的可能位置包括 JobManager 内存、文件系统、数据库等（State 默认存储在 TaskManager 内存中，而 Checkpoint 保存在 JobManager 内存中），具体取决于所配置的`State Backend`（比如持久化巨型状态就不适合放在内存），当前有以下 State Backend 可用于存储 Checkpoint State：

Job 级别的 StateBackend 配置方式如下：

```java
StreamExecutionEnvironment.setStateBackend()

```

这定义了该 Job 保存 State 的数据结构以及 Checkpoint 数据存储位置。

全局 StateBackend 配置方式如下：

-   MemoryStateBackend 为不配置时的默认选项。如果要修改默认配置，请修改`flink-conf.yaml`文件中的`state.backend`：\`\`\`java
    state.backend: filesystem

    state.checkpoints.dir: hdfs://namenode:40010/flink/checkpoints

    ````

    其中：
    *   state.backend  
        可选内置的`jobmanager` (MemoryStateBackend), `filesystem` (FsStateBackend), `rocksdb` (RocksDBStateBackend)，或自己实现自`StateBackendFactory`的类全限定名，如`org.apache.flink.contrib.streaming.state.RocksDBStateBackendFactory`。
    *   state.checkpoints.dir  
        定义了StateBackend写Checkpoint数据和元数据文件路径。目录结构如下：```
        /user-defined-checkpoint-dir
            /{job-id}
                |
                + --shared/
                + --taskowned/
                + --chk-1/
                + --chk-2/
                + --chk-3/
                ...
        
        ```
        
        `shared`目录保存了可能被多个Checkpoint引用的文件；`taskowned`存储JM不能删除的文件；`chk`开头的是每次checkpoint使用的文件，数字表示checkpointId。
    ````

### 5.4.2 MemoryStateBackend

![](https://img-blog.csdnimg.cn/2020110917290858.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/20201111152152802.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

-   MemoryStateBackend 为不配置时的默认选项。
-   存储位置

    -   平时  
        以 Java Object 形式放在 TaskManager 的 JVM 内存中。具体来说， Key/value state 和 Window 算子持有 HashTable 来存储 State 值和触发器等。
    -   Checkpoint 时  
        将 State 进行快照，然后将它作为 Checkpoint 消息一部分发送给 JobManager，然后放在 JobManager 的 JVM 内存中。
-   异步快照  
    支持，且默认开启，强烈建议使用异步快照来防止数据流阻塞。

    要关闭异步采用同步方式（官方建议只在 Debug 时关闭异步 Checkpoint）：

    ```java
    new MemoryStateBackend(MAX_MEM_STATE_SIZE, false);

    ```
-   使用限制

    -   每个单独的 State 默认最大 5MB，可用 MemoryStateBackend 构造函数设置

    ```java
    env.setStateBackend(new MemoryStateBackend(MAX_MEM_STATE_SIZE, false))

    ```

    但是必须注意该参数谨慎调大，因为 State Checkpoint 时 TaskManager 会将 State Checkpoint 后的数据通过限制了大小的 RPC 方式发送给 JobManager，而 JobManager 需要在内存保存来自各个 TaskManager 的状态数据。太大了可能导致 OOM！

    -   同时由于 Flink 节点间采用 Akka 通信，所以单状态大小受限于 Akka Frame Size 限制。
    -   JobManager 的 JVM 内存必须能存下所有聚合后的 State
-   特点

    -   这种方式轻量级无需其他依赖，但受 JVM 内存限制，只能对小型 State 做 Checkpoint，适用于
        -   本地开发测试
        -   计数器
        -   几乎无状态的 Job，比如仅由每次只处理一条记录的算子（Map、FlatMap、Filter 等）组成
        -   KafkaConsumer，只需要很少的 State 用于记录消费 offset 情况
    -   每次运算值需要读取状态时使用 Java object 读写，代价较小；
-   调优  
    `managed memory`内存设为 0，因为 RocksDB StateBackend 才会使用该部分内存。这样设定可以使得用户代码可以使用 JVM 能提供的最大值来执行。

### 5.4.3 FsStateBackend

![](https://img-blog.csdnimg.cn/20201102172953620.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/20201111152234289.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

-   存储位置

    -   平时  
        以 Java Object 形式放在 TaskManager 的 JVM 内存中，读写代价小
    -   Checkpoint 发生时  
        以 Java Object 形式将状态的 Checkpoint 序列化后保存到分布式文件系统，还有极小的 State 元数据存储在 JobManager 内存中（或是在高可用模式下将元数据存放在元数据 Checkpoint 中）。
-   异步快照  
    支持，且默认开启异步快照 Checkpoint 来防止数据流阻塞。

    如果要关闭异步采用同步方式：

    ```java
    new FsStateBackend(path, false);

    ```

    这里的`Path`是 Checkpoint 持久化目录，如`hdfs://namenode:40010/flink/checkpoints` 或`file:///data/flink/checkpoints`
-   特点

    -   当使用分布式文件系统如 HDFS 或 Alluxio 时，可保证单节点故障时也不会丢状态的数据，使得 Flink 程序高可用、强一致性。
    -   读写本地 State 时直接使用 Java Object 读写，开销较小
    -   当 Checkpoint 发生时需要将状态序列化再保存到 HDFS，有一定开销。
-   适用场景

    -   大型 State，很大的 key/value State
    -   很宽的 Window
    -   需要高可用
    -   同时对读写性能也有较高要求
    -   注意，不支持增量 Checkpoint
-   调优  
    `managed memory`内存设为 0，因为 RocksDB StateBackend 才会使用该部分内存。这样设定可以使得用户代码可以使用 JVM 能提供的最大值来执行。

### 5.4.4 RocksDBStateBackend

可参考：

-   [RocksDB State Backend Details](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/state/state_backends.html#setting-the-per-job-state-backend)
-   [task executor memory configuration](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_tuning.html#rocksdb-state-backend)
-   [Tuning Checkpoints and Large State](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/state/large_state_tuning.html)
-   [RocksDB Native Metrics](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/config.html#rocksdb-native-metrics)
-   [每个 Slot 中 RocksDB 实例占用的总内存大小配置](https://ci.apache.org/projects/flink/flink-docs-release-1.11/zh/ops/state/large_state_tuning.html#bounding-rocksdb-memory-usage)

#### 5.4.4.1 概述

![](https://img-blog.csdnimg.cn/20201102173224843.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/20201111152245570.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

-   简介  
    RocksDB 是一个以日志合并树 ( LSM 树，Kudu、HBase 都有使用）作为索引结构的 KV 存储引擎。当用于在 Flink 中存储 kv 状态时，Key 由`<Keygroup，Key，Namespace>` 的序列化字节串组成，而 Value 由状态的序列化字节组成。

    每次注册 kv 状态时，它都会映射到列族（column-family），并将键值对以字节存储在 RocksDB 中。**这意味着每次读写（READ or WRITE）操作都必须对数据进行反序列化或者序列化，与 Flink 内置的 in-memory 状态后端相比，会有一些性能开销。** 
-   存储位置

    -   平时  
        将运行时 State 数据保存在[RocksDB](http://rocksdb.org/)数据库中，RocksDB 数据库默认将数据存储在 TaskManager 节点磁盘上的的数据目录。
    -   Checkpoint 发生时  
        将整个 RocksDB 数据库 Checkpoint 到配置的文件系统目录。同样的，还有极小的 State 元数据存储在 JobManager 内存中（或是在高可用模式下将元数据存放在元数据 Checkpoint 中）。
-   仅支持异步快照  
    RocksDBStateBackend 仅支持异步快照，且默认开启异步快照 Checkpoint 来防止数据流阻塞。
-   可支持增量 Checkpoint  
    RocksDB 是唯一可支持的
-   使用限制  
    由于 RocksDB 的 JNI API 构建在 byte\[] 数据结构之上, 所以每个 key 和 value 最大支持`2^31` 字节。 **注**意: RocksDB 使用 merge 算子的状态（例如 ListState）累积数据量大小可能悄悄超过 `2^31` 字节，会在下一次获取数据时失败。这是当前 RocksDB JNI 的限制。
-   特点

    -   可保留的 State 大小仅受可用磁盘空间量的限制。

        与平时将 State 保留在内存中的 FsStateBackend 相比，RocksDBStateBackend 可以保留非常大的状态。 但这也意味着若使用 RocksDBStateBackend，则可以实现的最大吞吐量将降低。
    -   每次读取状态都有序列化 / 反序列化开销  
        从 RocksDBStateBackend 进行的所有读 / 写都必须经过序列化 / 反序列化以检索和存储 State 对象，这也比基于堆的 StateBackend 开销大很多。
    -   可使用增量快照
    -   不受 Java 垃圾回收的影响，与 heap 对象相比，它的内存开销更低
    -   RocksDB 是唯一可支持增量 Checkpoint 的
-   适用场景

    -   巨型 State，巨大的 key/value State
    -   很宽的 Window，如天级窗口聚合计算
    -   需要高可用
    -   对计算性能要求较低，因为 RocksDB 读写有序列化 / 反序列化开销

#### 5.4.4.2 [内存管理](https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/state/state_backends.html#memory-management)

Flink 花了很多功夫在内存管理上，以使得 TM 很好得利用内存，不至于在容器环境因为内存超限被杀掉，也不会因为内存利用率过低导致大量内存数据落盘或缓存命中率太低。

默认，RocksDB 的可用内存配置为 TM 的一个 Slot 的内存量，一般用户无需调整细节，只需在不够时增加整体内存大小即可。`state.backend.rocksdb.memory.managed`就开启了 RocksDB 使用`Managed Memory`，Flink 通过配置 RocksDB 来确保其使用的内存正好与 Flink 的 Managed Memory 预算相同，计算粒度是`Per-Slot`。也就是说，Flink 会为一个 Slot 上的所有 RocksDB 实例使用共享的 `RocksDB cache`和 `RocksDB write buffer manager`

调整缓冲区：

-   写入缓冲内存不够时（现象为频繁 flush）可调整`state.backend.rocksdb.memory.write-buffer-ratio`即分配给写入缓存的比例，默认 0.5 即 50%。
-   读取缓冲命中率低时可调整`state.backend.rocksdb.memory.high-prio-pool-ratio`即分配给写入缓存的比例，默认 0.1 即 10%，这部分内存优先分配给 RocksDB 的索引和过滤器。

但专业用户也可以手动为 RocksDB 每个列族（每个算子的每个 State 就对应了一个列族）分配内存，用户自己来确保总内存不会超限。

#### 5.4.4.3 Timer（Heap vs RocksDB）

这里说的 Timer 是指用来触发窗口或回调 ProcessFunction 的类似操作，可基于事件时间或处理时间。

RocksDBStateBackend 中 Timer 默认存在 RocksDB，需要一定成本维护，所以也可以将 Timer 存在 Java 堆。

Timer 较少时，可将`state.backend.rocksdb.timer-service.factory`设为`heap`，可获得更好的性能。但这样设置后，Timer 状态就不能被异步快照存储了。

#### 5.4.4.4 列族（ColumnFamily）级别的预定义选项

#### 5.4.4.5 通过 RocksDBOptionsFactory 配置 RocksDB 选项

### 5.4.5 存储原理

#### 5.4.5.1 概述

![](https://img-blog.csdnimg.cn/20201207121634485.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

#### 5.4.5.2 HeapKeyedStateBackend

对于 HeapKeyedStateBackend，有两种实现：

-   支持异步 Checkpoint（默认）：存储格式`CopyOnWriteStateMap`  
    ![](https://img-blog.csdnimg.cn/20201207122216806.png)
-   仅支持同步 Checkpoint：存储格式 `NestedStateMap`  
    ![](https://img-blog.csdnimg.cn/2020120712223267.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
-   namespace 用来标注如属于哪个 window
-   `MemoryStateBackend` 内使用`HeapKeyedStateBackend`时，Checkpoint 序列化数据阶段默认有最大 5 MB 数据的限制  
    ![](https://img-blog.csdnimg.cn/20201207180056837.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

#### 5.4.5.3 RocksDBKeyedStateBackend

-   `RocksDBKeyedStateBackend`的每个 state 都存储在一个单独的 column family 内，使用了基于 LSM 树的磁盘、内存混合型 DB  
    ![](https://img-blog.csdnimg.cn/2020120712335899.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

写入相关操作如下图  
![](https://img-blog.csdnimg.cn/20201209144840297.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

1.  将数据写入内存中的 Active MemTable
2.  MemTable 会被后台线程周期性 Flush 到磁盘，生成按 Key 排序的只读的不可变文件`SSTable`  
    当 MemTable 写满时，它将成为 READ ONLY MemTable，并被一个新申请的 MemTable 替换。
3.  刷到磁盘的 SSTable 文件会被后台线程多路归，并实现进一步的整合为大文件，提升读取效率  
    合并后的 sstable 包含所有的键值对，RocksDB 会删除合并前的 sstable。
4.  **每个注册状态都是一个列族，这意味着每个状态都包含独享的 MemTables 和 SSTables 集。** 

读取时：

-   首先访问`Active MemTable`。
-   若找不到，则访问`SSTable`
    -   优先查 RocksDB BlockCache
    -   没有则查操作系统缓存 PageCache
    -   在没有就查本地磁盘
    -   在此过程中可使用`bloom filter`减少大量磁盘访问，进行过滤

增量时（可参考[Managing Large State in Apache Flink: An Intro to Incremental Checkpointing](https://flink.apache.org/features/2018/01/30/incremental-checkpointing.html)以及 [翻译版](https://ververica.cn/developers/manage-large-state-incremental-checkpoint/)）：

-   Flink 将所有新生成的 sstable 备份到持久化存储（比如 HDFS，S3），并在新的 checkpoint 中引用。Flink 并不备份前一个 checkpoint 中已经存在的 sstable，而是在需要时仅引用他们，并记录下引用次数。Flink 还能够保证所有的 checkpoint 都不会引用已经删除的文件，因为 RocksDB 中文件删除是由压缩完成的，压缩后会将原来的内容合并写成一个新的 sstable。因此，Flink 增量 checkpoint 能够切断 checkpoint 历史。

![](https://img-blog.csdnimg.cn/20201209152831996.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

以下转自[Apache Flink 管理大型状态之增量 Checkpoint 详解](https://ververica.cn/developers/manage-large-state-incremental-checkpoint/)，作者 Stefan Ricther & Chris Ward，翻译 邱从贤（山智）  
![](https://img-blog.csdnimg.cn/20201209154051783.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

![](https://img-blog.csdnimg.cn/20201209154227370.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

关于控制便捷性与性能之间平衡的策略可以参考此文档：

-   [Tuning Checkpoints and Large State](https://ci.apache.org/projects/flink/flink-docs-release-1.12/ops/state/large_state_tuning.html)

### 5.4.6 高级用法

-   KeyedStateCheckpointOutputStream
-   OperatorStateCheckpointOutputStream

### 5.4.7 性能对比

### 5.4.8 更多配置

可参考：

-   [Flink 完整配置参数](https://ci.apache.org/projects/flink/flink-docs-release-1.10/zh/ops/config.html)

以下配置在`conf/flink-conf.yaml`中配置，改变后需要重启 Flink 应用。

| Key                                | Default | Type    | Description                                                                                                                       |
| ---------------------------------- | ------- | ------- | --------------------------------------------------------------------------------------------------------------------------------- |
| state.backend                      | (none)  | String  | 保存和 Checkpoint State 的位置                                                                                                          |
| state.backend.async                | true    | Boolean | StateBackend 是否应在可能且可配置的情况下使用异步快照方法。 某 StateBackend 可能不支持 / 仅支持异步快照，会忽略此选项。                                                       |
| state.backend.fs.memory-threshold  | 1024    | Integer | State 数据文件大小的最小值。 所有大小小于该值的 State Chunk 都以内联方式存储在根 Checkpoint 元数据文件中。                                                             |
| state.backend.fs.write-buffer-size | 4096    | Intege  | 写入文件系统的 Checkpoint 流的写缓冲区的默认大小。 实际的写缓冲区大小为本选项和`state.backend.fs.memory-threshold`的最大值                                             |
| state.backend.incremental          | false   | Boolean | StateBackend 是否应创建增量 Checkpoint（如果可能）。 对于增量 Checkpoint，仅存储与前一个 Checkpoint 的差异，而不存储完整的 Checkpoint 状态。 某些 StateBackend 可能不支持，会忽略此选项 |
| state.backend.local-recovery       | false   | Boolean | 默认禁用。为此 StateBackend 配置本地恢复。 当前，本地恢复仅涵盖 Keyed-StateBackend。 注意，目前 MemoryStateBackend 不支持本地恢复，请忽略此选项。                              |
| state.checkpoints.dir              | (none)  | String  | 用于在 Flink 支持的文件系统中存储 Checkpoint 的数据文件和元数据的默认目录。 必须从所有参与的进程 / 节点（即所有 TaskManager 和 JobManager 节点）访问存储路径                            |
| state.checkpoints.num-retained     | 1       | Integer | 要保留的最大已完成 Checkpoint 数                                                                                                            |
| state.savepoints.dir               | (none)  | String  | Savepoint 的默认存储目录。 由 StateBackend 用于将 Savepoint 写入文件系统（适用于 MemoryStateBackend，FsStateBackend，RocksDBStateBackend）                 |
| taskmanager.state.local.root-dirs  | (none)  | String  | 定义用于存储基于文件的 State 以进行本地恢复的根目录。 当前，本地恢复仅涵盖 Keyed-StateBackend。 当前，MemoryStateBackend 不支持本地恢复，请忽略此选项                                |

## 5.5 [巨大状态的调优](https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/state/large_state_tuning.html)

### 5.5.1 概述

为了使 Flink 应用程序在规模很大时仍能可靠运行，必须满足两个条件：

-   应用程序必须能够可靠地运行 Checkpoint
-   发生故障后，程序重启后的资源必须足够赶上输入数据流

### 5.5.2 监控状态和 Checkpoint

可参考[Monitoring Checkpointing](https://ci.apache.org/projects/flink/flink-docs-release-1.11/monitoring/checkpoint_monitoring.html)

Checkpoint 关键指标：

-   checkpoint_start_delay  
    表示从 Checkpoint 被 Checkpoint 发起到算子真正开始执行 Checkpoint 时间 ，计算公式如下：  
    `checkpoint_start_delay = end_to_end_duration - synchronous_duration - asynchronous_duration`

    如果该值持续高值，说明`checkpoint barrier`从 Source 流到算子的时间过长，一般就意味着出现了长期背压的情况。
-   CheckpointBarrier 对齐期间的数据缓存量  
    在开启`exactly-once`语义后，多路 input 的算子收到一个 input 的 Barrier 就需要做对齐操作，在此期间就需要缓存数据。理想情况下缓存数据应该比较少，否则说明多个 input 的 Bairrer 长期不对齐。

### 5.5.3 Checkpoint 调优

当 Checkpoint 耗时过长，意味着 Checkpoint 耗费太多资源，算子进展太少，虽然是异步 Checkpoint，但依然会对整个应用性能产生影响。

此时就应该使用`StreamExecutionEnvironment.getCheckpointConfig().setMinPauseBetweenCheckpoints`调大 Checkpoint 之间最小间隔时间，该配置是定义了`当前 Checkpoint 的结束与下一个 Checkpoint 的开始`之间必须经过的最短时间间隔，官网图例如下：  
![](https://img-blog.csdnimg.cn/20200813180603561.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

-   图一为正常 Checkpoint
-   图二为 Checkpoint 执行时间超过了设定的间隔，导致每次 Checkpoint 刚结束就开始下一个 Checkpoint
-   图三表示采用了 minPause 限制，即使 Checkpoint 执行时间过久，也必须等到 minPause 后才能跑下一个 Checkpoint

### 5.5.4 RocksDB 调优

-   开启增量 Checkpoint
-   将默认存在 RocksDB 中的 Timer 改为存在 Java 堆，谨慎使用！  
    Timer 较少时，可将`state.backend.rocksdb.timer-service.factory`设为`heap`，可获得更好的性能。但这样设置后，Timer 状态就不能被异步快照存储了。而且可能增加 Checkpoint 时间，大小也不能超过能存限制\*\*，谨慎使用！\*\*
-   Tuning RocksDB Memory
    1.  最直接的方式就是调大`managed memory`，可以显著提高性能。  
        默认只有 0.4 比例，除非程序逻辑本身需要很多 JavaHeap 内存，否则可尽量多给`managed memory`
    2.  因为每个状态对应了一个 RocksDB 列族，每个列族都需要独有的 write buffer，所以拥有很多状态的应用通常需要更多内存
    3.  可设`state.backend.rocksdb.memory.managed：false`来对比性能  
        非托管模式中，除非使用了`ColumnFamily`，否则上线公式为`140MB * num-states-across-all-tasks * num-slots`（State 包括 Timer）
    4.  如果拥有很多状态的应用中观察到频繁地`MemTable flushes`，说明写入有瓶颈了，如果你不能给 RocksDB 更多内存了，那此时可以配置`state.backend.rocksdb.memory.write-buffer-ratio`将更多内存分配给 write buffer。
    5.  专家可以尝试使用`RocksDBOptionsFactory`来调整`arena block size`， `max background flush threads`等，以减少 \`MemTable flushes\`\`\`\`java
        	public class MyOptionsFactory implements ConfigurableRocksDBOptionsFactory {

            @Override
            public DBOptions createDBOptions(DBOptions currentOptions, Collection<AutoCloseable> handlesToClose) {
                
                
                return currentOptions.setMaxBackgroundFlushes(4);
            }

            @Override
            public ColumnFamilyOptions createColumnOptions(
                ColumnFamilyOptions currentOptions, Collection<AutoCloseable> handlesToClose) {
                
                return currentOptions.setArenaBlockSize(1024 * 1024);
            }

            @Override
            public OptionsFactory configure(Configuration configuration) {
                return this;
            }

        }

        ```

        ```

### 5.5.5 资源规划

要想是的 Flink Job 运行可靠，一般可遵循以下步骤规划资源：

1.  正常运行应具有足够的容量，但应在恒定的背压下无法运行。
2.  在无故障且无背压运行程序所需的资源的基础上，还需提供一些额外的资源，以备在发生错误恢复时有足够资源来追上恢复任务期间累积的未处理数据。  
    具体取决于任务恢复速度以及具体场景需要多快恢复
3.  短暂的背压是没问题的，他是负载高峰或要写入的外部系统突然变慢时的必要流控手段。
4.  一些算子（如巨大的 window）会在每次触发时给下游算子带来负载高峰，所以下游算子并行度规划时需要考虑 window 大小、频率以及需要处理完成的速度
5.  Job 最大并行度应该设置足够，决定了使用`savepoint`来重设并行度时的上限

### 5.5.6 Checkpoint 压缩

默认关闭。

目前只支持[snappy compression algorithm (version 1.1.4)](https://github.com/xerial/snappy-java):

```java
ExecutionConfig executionConfig = new ExecutionConfig();
executionConfig.setUseSnapshotCompression(true);

```

注意，此选项不适用于增量快照，因为增量快照使用的是 RocksDB 的内部格式，始终使用开箱即用的 snappy compression 。

Checkpoint 压缩工作粒度为`keyed state`下的`key-group`，解压时也是此粒度，方便扩缩容。

### 5.5.7 Task 本地恢复

#### 5.5.7.1 现存问题 - 大状态任务恢复慢

Checkpoint 时，每个 Task 都会生产一个状态快照，然后异步写入 StateBackend。随后，每个 Task 会发送一个 ACK 给 JM 告知状态写入成功，该 ACK 是一个句柄，带有 state 在 StateBackend 中的位置。JM 在收集完所有 task 的 ACK 后，将他们封装到一个 Checkpoint 对象中。

在恢复时，JM 打开最近的 Checkpoint，然后将其中的 state 文件句柄发回给各个对应的 task 以进行恢复。

使用分布式 StateBackend 的好处是容错性和方便扩缩容；坏处是必须通过网络远程读取访问，可能在大型状态场景导致很长的恢复时间，尽管可能只是因为很小的错误发生。

#### 5.5.7.2 Task 本地恢复

##### 5.5.7.2.1 概述

![](https://img-blog.csdnimg.cn/20200814112150427.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

核心思想就是每次 Checkpoint 时，写一份主副本到远程 StateBackend，再写一份副本到本地（磁盘或内存）。

这样，在恢复时大多数 task 只需要找到之前本地 state 就能从本地恢复无需从远程下载。

##### 5.5.7.2.2 两个副本间的关系

Checkpoint 的 StateBackend 副本始终被认为是主副本，而本地副本是第二副本。而且他们格式不一定相同，比如存放在内存的本地状态就是 Java 对象。

Checkpoint 时，**主副本必须成功此次 Checkpoint 才能成功**，而如果此时本地副本写入失败不会导致此次 checkpoint 失败。

主副本被 JM 确认和管理，而第二副本是由 TM 管理，生命周期可独立于主副本。比如保留 3 个最新主副本，而只保留一个本地最新副本。

恢复时，总是先尝试本地副本，如果失败就找远程主副本，如果还是失败就根据配置可找上一个 checkpoint 继续恢复。

更巧妙的是，即使 Flink 因为错误只写入部分状态到本地副本，也会尝试恢复这部分状态，然后再去远程副本恢复其他状态，因为远程状态总是完整的。

但如果 TM 挂了，则关联的本地状态也会全部丢失。

##### 5.5.7.2.3 本地 Checkpoint 副本配置

默认关闭。

| 配置                           | 默认    | 类型      | 说明                           |
| ---------------------------- | ----- | ------- | ---------------------------- |
| state.backend.local-recovery | false | Boolean | 默认禁用。为此 StateBackend 配置本地恢复。 |

当前，本地恢复仅涵盖 Keyed-StateBackend。  
注意，目前 MemoryStateBackend 不支持本地恢复，请忽略此选项。 |

注意，目前非对齐 Checkpoint 不支持本地 Checkpoint 副本恢复。

##### 5.5.7.2.4 不同 StateBackend 的本地 Checkpoint 副本

当前只支持 KeyedStateBackend，未来会支持算子和 timer 的状态。

目前支持的 StateBackend 如下：

-   FsStateBackend  
    将状态复制到一个本地文件，会带来一定写入和磁盘空间开销。未来也许会做一个内存副本版本。
-   RocksDBStateBackend
    -   增量 Checkpoint  
        根据 RocksDB 本地 checkpoint 机制。详解[Details on task-local recovery for different state backends](https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/state/large_state_tuning.html#details-on-task-local-recovery-for-different-state-backends)
    -   全量 Checkpoint  
        将状态复制到一个本地文件，会带来一定写入和磁盘空间开销。

##### 5.5.7.2.5 保留分配的 task 调度

Task 本地恢复模式中假定在错误后保留 task 调度，工作机制如下：

-   每个 task 记住之前的分配，请求同样的 slot 来开始恢复
-   如果该 slot 不可用，则 task 向 RM 请求一个全新的 slot  
    这样，即使一个 TM 不再可用，一个不能回归到之前位置的 task 不会再驱动其他正在恢复的任务从之前的 slot 中移出
-   这样做的好处是使得从本地恢复状态的 task 数量最大化，避免了夺取其他 task 之前的 slot 带来的级联效应（互相夺取 slot）。

## 5.6 增量 Checkpoint

### 5.6.1 概述

在减少 Checkpoint 花费的时间时，可首先考虑开启增量 Checkpoint。 与完整 Checkpoint 相比，增量 Checkpoint 可以大大减少 Checkpoint 的时间，因为增量 Checkpoint 仅记录与先前完成的 Checkpoint 相比发生变化的部分，而不是生成完整数据。

一个增量 checkpoint 依赖之前的若干 checkpoint 构建。由于 RocksDB 内部存在 compaction 机制对 sst 文件进行合并，Flink 的增量快照也会定期重新设立起点（rebase），因此增量链条不会一直增长，旧快照包含的文件也会逐渐过期并被自动清理。

如果网络为瓶颈，则从增量 Checkpoint 恢复可能会较慢，因为需要读取更多 delta 增量文件；而如果 CPU 或 IO 为瓶颈，则从增量 Checkpoint 恢复更快，因为这种方式不会像全量或 savepoint 那样使用并解析 Flink Key/Value 快照格式数据来重建本地 RocksDB 表。

增量 Checkpoint 开启方式：

```java
new RocksDBStateBackend(path, TernaryBoolean.TRUE);

```

这里的`Path`是 Checkpoint 持久化目录，如`hdfs://namenode:40010/flink/checkpoints` 或`file:///data/flink/checkpoints`。  
或使用配置`state.backend.incremental: true`开启

需要注意的是，只要开启了增量 Checkpoint，则在 WebUI 上显示的`Checkpointed Data Size`就只代表增量 Checkpoint 数据大小，而不是整个 State 大小了。

### 5.6.2 原理

![](https://img-blog.csdnimg.cn/2020120717070431.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

### 5.6.3 调优

![](https://img-blog.csdnimg.cn/20201205163515776.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

## 5.7 状态最佳实践

-   [Flink State 最佳实践](https://ververica.cn/developers/flink-state-best-practices/)
-   [如何在 Flink 中规划 RocksDB 内存容量？](https://ververica.cn/developers/how-to-plan-the-memory-capacity-of-rocksdb-in-flink/)

## 6.1 Task

### 6.1.1 概述

是 PhysicalGraph 的节点，一个 task 是 Job 的基本组成单元，在 Flink 运行时被执行。

Task 精确封装了一个 Operator 或者 Operator Chain 的并行实例。

### 6.1.1 Sub-Task

一个 Sub-Task 是指处理数据流的一个 partition 的 task。

该术语强调同一个 Operator 或者 Operator Chain 拥有多个并行实例由多个 Task 执行。

## 6.2 Operator

### 6.2.1 概述

Operator 即算子，是 LogicalGraph 的节点。

每个算子执行特定操作。

### 6.2.2 Operator 与 Task

分布式计算中，Flink 将算子（operator）的 subtask 按一定规则链接成若干 task，每个 task 由 TaskManager 上的一个线程执行。

把算子链接成 task 能够有效减少线程间切换和缓冲的开销，在降低延迟的同时提高了整体吞吐量。

下图的 dataflow 由 5 个 subtasks 执行，因此具有 5 个并行线程运行：  
![](https://img-blog.csdnimg.cn/20200630160252555.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

-   上方那个 Operator Chain 是整个算子链的概要图  
    可以看到 source 和 map 组成了一个 chain 在一个 task 执行；keyBy 是一个单独的 task；sink 也是一个单独 task
-   下方是加入了并行概念后的算子链视图  
    因为 source/map/keyBy 并行度都为 2，所以共有 4 个 Task；sink 并行度为 1；所以总共有 5 个 task。

Operator Chain 配置详情可参考：6.2.2

### 6.2.3 Operator Chain

#### 6.2.3.1 概述

将两个连续的算子（即不会造成 repatitioning 的算子）连接即将他们放在同一个线程执行，同一个算子链内的算子直接交换数据，而不会进行序列化 / 反序列化或 Flink 网络相关操作，以获得更好的性能，优点：

-   减少线程切换
-   内部传输过程不会进行序列化 / 反序列化或 Flink 网络相关操作
-   减少数据在缓冲区的交换
-   减少了延迟的同时提高整体的吞吐量

Flink 默认会尽可能将算子连接（比如两个 map 算组），但也可以自主控制：判断是否构成 Chain 的条件很苛刻，需要以下提交件全部都为 true:

-   下游 StreamNode 的入边数为 1，即只有一个 input，而不是多个输入
-   上游 StreamNode 和下游 StreamNode 同在一个`SlotSharingGroup`
-   上游的`ChainingStrategy`不能是`NEVER`  
    即只能是`ALWAYS`（可参与上下游连接，如 FlatMap、Filter、Map）或`HEAD`（只能作为上游与下游连接，但不能与下游连接，如 Source）
-   下游必须是`ALWAYS`
-   该 StreamEdge 的 Partitioner 必须是`ForwardPartitioner`，即发送和接收分区一一对应
-   该 StreamEdge 的 ShuffleMode 不能为 BATCH，该模式下生产者先生产完整数据，然后消费者才开始消费数据；还有个模式是 PIPELINED，此时生产者消费者同时完成生产消费；UNDEFINED 的含义是由 Flink 推断到底采用以上哪种模式。
-   上下游并行度相同
-   StreamGraph 允许 Chain 机制。默认 true，可通过`pipeline.operator-chaining`设置。

下面这幅图，展示了 Source 并行度为 1，FlatMap、KeyAggregation、Sink 并行度均为 2，最终以 5 个并行的 Task 线程来执行:  
![](https://img-blog.csdnimg.cn/2020090115030331.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

可以看到，`Key Agg`和 Sink 因为符合 Chain 的条件被连接到一起，作为一个 Task 执行。

#### 6.2.3.2 配置

可在 job 级别禁用 Operator Chain：

```java
StreamExecutionEnvironment.disableOperatorChaining()

```

细粒度控制：

-   开启新的 Operator Chain  
    注意`startNewChain`必须用在算子之间，不能直接放在 stream 后面。

    以下例子 filter 是单独的 task，但第一个 map 和第二个 map 组成了一个 Operator Chain

    ```java
    someStream.filter(...).map(...).startNewChain().map(...)

    ```
-   算子级别禁用 Operator Chain

    ```java
    someStream.map(...).disableChaining();

    ```

#### 6.2.3.3 实现原理

以下为入度为 1，出度为 2 的一个 OperatorChain 示例。  
![](https://img-blog.csdnimg.cn/20200901152045552.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

-   `OperatorChain`为一个算子链的呈现类
-   OperatorChain 对外部来说就是一个整体，只需要将 input 数据传输给该算子链的 HeadOperator 即可  
    比如上图的算子链就可以看作是入度为 1，出度为 2 的一个算子整体
-   上图中的实线就对应了 JobEdge
-   上图中的虚线就是 OperatorChain 内部数据传输`ChainingOutput`，不会经过序列化 / 反序列化、网络传输，而是直接通过方法传递处理。

    OperatorChain.ChainingOutput 如下

    ```java
    static class ChainingOutput<T> implements WatermarkGaugeExposingOutput<StreamRecord<T>> {
    	
    	protected final OneInputStreamOperator<T, ?> operator;
    	protected final Counter numRecordsIn;
    	protected final WatermarkGauge watermarkGauge = new WatermarkGauge();

    	protected final StreamStatusProvider streamStatusProvider;

    	
    	@Nullable
    	protected final OutputTag<T> outputTag;

    	public ChainingOutput(
    			OneInputStreamOperator<T, ?> operator,
    			StreamStatusProvider streamStatusProvider,
    			@Nullable OutputTag<T> outputTag) {
    		
    		this.operator = operator;
    		...
    		this.streamStatusProvider = streamStatusProvider;
    		this.outputTag = outputTag;
    	}

    	@Override
    	public void collect(StreamRecord<T> record) {
    		if (this.outputTag != null) {
    			
    			return;
    		}

    		pushToOperator(record);
    	}
    	...
    	protected <X> void pushToOperator(StreamRecord<X> record) {
    		try {
    			
    			
    			@SuppressWarnings("unchecked")
    			StreamRecord<T> castRecord = (StreamRecord<T>) record;

    			numRecordsIn.inc();
    			operator.setKeyContextElement1(castRecord);
    			
    			operator.processElement(castRecord);
    		}
    		catch (Exception e) {
    			throw new ExceptionInChainedOperatorException(e);
    		}
    	}
    	...
    }

    ```

### 6.2.4 Flink 自带 Operator

首先，Source 和 Sink 是特殊的算子，用来数据摄取和输出。

### 6.2.5 Operator 并行度

Operator 并行度配置可参考[这里](https://blog.csdn.net/baichoufei90/article/details/82884554#job-parallel)。

### 6.2.6 Operator 数据交换与分区策略

![](https://img-blog.csdnimg.cn/20200825233037308.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

DataStream 可以在两个算子间传输数据，有以下两种模式：

-   一对一  
    例如上图中`Source` 和 `map()`算子之间。

    可保留元素的分区和排序信息（也就是说`map()`算子的 1 号实例可以相同顺序看到跟`Source`算子的 1 号实例生产顺序相同的元素）。
-   重分发 - 类似 MR Shuffle  
    例如上图中的 `map()`和 `keyBy/window`算子 之间，以及 `keyBy/window`和 `Sink` 之间。

    会更改数据所在 Stream 分区。注意此时只能保证一个算子 subtask 发到一个下游算子 subtask 的元素顺序性。如上图 keyBy/window 的 subtask\[2] 接收到的 map() 的 subtask\[1]的数据有序，但发送到 Sink 的所有数据中，无法确定不同 key 的聚合结果的到达顺序。

    每个算子 subtask 发送数据到不同的下游算子 subtask，分发依据是具体的`transformation`(相关方法在`org.apache.flink.streaming.api.datastream.DataStream`)：

    -   keyBy  
        按照 key 的值 hash 后重分区到某个下游算子实例
    -   broadcast  
        广播到所有下游算子实例分区
    -   rebalance  
        轮询分配到下游算子实例分区
    -   global  
        全部分配到第一个下游算子实例分区
    -   shuffle  
        随机均匀分配到下游算子实例分区
    -   forward  
        上下游并行度一致时，发送到对应的位于本地的下游算子分区
    -   rescale  
        轮询方式将输出的元素均匀分发到下游分区的子集。

        子集构建依赖于上游和下游算子的并行度。

        -   比如上游算子并行度 2，下游为 4，此时每个上游算子轮询各自分发到下游的两个算子。
        -   如果上游并行度 4，下游为 2，此时每两个上游算子分发到一个下游算子。
        -   如果不是倍数，则下游分发的源头数目不一致

## 6.3 TaskSlot 和 资源

### 6.3.1 概述

每个 TaskManager 都是一个 JVM 进程，并且可以在不同的 Slot 线程中执行多个 subtasks，Slot 控制 TaskManager 并发执行 task 的数量，每个 TaskManager 至少一个 Slot。他们的关系示例如下图，此时作业基本并行度为 2，一个 taskManager 的 slot 数量为 3：  
![](https://img-blog.csdnimg.cn/20200707002801417.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

### 6.3.2 Slot 与资源隔离

每个 Slot 是 TaskManager 资源分配和调度最小单元，代表一份固定资源子集。例如，具有三个 slot 的 TaskManager 会将其管理的内存资源平均分成三等份给每个 slot。 划分资源意味着 subtask 之间不会竞争资源，但这也意味着它们只能拥有固定的划分了的资源。

注意 Slot 并没有实现 CPU 隔离，当前 Slot 之间只实现了内存资源隔离划分。

通过调整 slot 的数量，用户可以决定 subtask 的隔离方式:

-   每个 TaskManager 只有一个 slot  
    ~ 意味着每个 task 在一个单独的 JVM 中运行（即每个 task 独享一个 TM 进程）~ 。**但可以注意到，目前 Flink1.11 版本已经支持多个 Task 线程共享一个 Slot，所以以上结论已经不再适用。** 
-   每个 TaskManager 多个 slot  
    意味着多个 subtask 共享同一个 JVM，分别在各自 slot 线程运行。

    在同一个 JVM 中运行的 Task 会共享 TCP 连接（通过 IO 多路复用）和心跳信息，可以减少数据的网络传输。它们还可能共享数据集和数据结构，从而降低每个 task 的性能开销。

### 6.3.3 Subtask 共享 slot

默认情况下，Flink 允许 subtask 们 共享 slot，即使它们是不同 task 的 subtasks，只要它们来自同一个 job 就行。因此，一个 slot 在极端场景下甚至可能会负责这个 job 的整个执行 pipeline！

允许 slot sharing 有两个好处：

-   Flink 集群的 Slot 数量总数需要与 job 中使用的最高并行度 (`highest parallelism`) 完全相同。这样不需要计算作业总共包含多少个 tasks（具有不同并行度）。
-   更好的资源利用率  
    如果不能共享 slot ，则简单的 subtask（比如`source`/ `map`等）将会占用和复杂的 subtask （如`window`）一样多的资源。

    而通过 slot 共享，将之前示例中的 job 最大并行度从 2 增加到 6 就可以完全利用 按 slot 分隔 的资源，同时确保开销大的 subtask 在 所有 TaskManager 上均匀分布：  
    ![](https://img-blog.csdnimg.cn/20200707004053553.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

### 6.3.4 slot 与线程

一个 Slot 并不一定只有一个线程运行，比如上图中一个 Task Slot 内部就运行了 2-3 个线程！

这些线程共同分享 Slot 获得的分隔内存资源。

不同算子是否能运行在一个 Slot 取决于 SlotSharingGroup

### 6.3.5 slot 数量设置

根据经验，合理的 slots 数量应该和 CPU 核数相同。在使用超线程（hyper-threading）时，每个 slot 将会占用 2 个或更多的硬件线程上下文（hardware thread contexts）。

### 6.3.6 SlotSharingGroup

#### 6.3.6.1 概述

Flink 会将相同`SlotSharingGroup`的算子放到相同的 slot 执行，而将 SlotSharingGroup 不同的算子放到其他 slot，这可用来隔离 slot。

如果所有 input 算子都在相同 SlotSharingGroup，则下游算子的 SlotSharingGroup 继承自 input 算子。

请记住，SlotSharingGroup**软定义**了不同的 task(JobVertex) 是否可在一个 Slot 中运行

#### 6.3.6.2 配置

默认的`slot sharing group`的名字是`default`，算子也可以显示地放入指定名字的`slot sharing group`：

```java
someStream.filter(...).slotSharingGroup("name");

```

### 6.3.7 CoLocationGroup

`CoLocationGroup`是 JobVertex 组，**硬性**定义了某个 JobVertex 的第 i 个子任务必须和其他所有 JobVertex 的第 i 个子任务运行在相同 TaskManager 上。

## 6.4 Task 运行模型

### 6.4.1 概述

-   Task 运行在一个独占线程中，多个 Task 线程可以共享一份 Slot 资源
-   Source Task 使用 SourceFunction 生产数据
-   运算 Task 负责从 InputGate 读取数据，调用 OperatorChain，最终结果输出到 ResultSubPartition，通过 Netty 发送给下游算子的 InputChannel。
-   最后是 Sink 的 InputChannel 接收到数据后，通过调用`SinkFunction`相关方法发送到目的地。

![](https://img-blog.csdnimg.cn/20201013173852405.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

### 6.4.2 概述

## 7.1 概述

当 Flink 某个任务失败挂掉时，会对失败任务和其他受影响任务重启来恢复整个作业。

Flink 中主要有两类策略来控制任务重启：

-   重启策略  
    决定了是否重启、何时重启失败和受影响任务
-   Failover 策略  
    决定了 job 中的哪些任务应该被重启。

可参考

-   [任务失败恢复](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/task_failure_recovery.html)

如果 Flink 部署在 yan 上，则需要依赖 Yarn 来实现高可用，Yarn 会对失败挂掉的 JobManager(AM) 重启，最大重试次数的配置是`yarn-site.xml`的`yarn.resourcemanager.am.max-attempts`。

另一方面，Flink 还支持高可用配置。

## 7.2 重启策略

优先级最高的是 job 自己指定的，然后才是 flink-conf.xml 中指定的：

采用`restart-strategy`配置。

### 7.2.1 不允许 Checkpoint 时默认采用 none

当不允许 checkpoint 时，采用的是`none`(也可以填`off`、`disable`) 策略，即不重启

可用配置项：

-   restart-strategy: none

代码：

```java
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.noRestart())

```

### 7.2.2 开启 Checkpoint 时默认 fixeddelay

开启 checkpoint 时，不配置重启策略时默认采用`fixeddelay`(也可以填`fixed-delay`、`failure-rate`)，是一种固定间隔重启的策略，默认会 1 秒间隔来进行 Integer.MAX_VALUE 次重启尝试，超过最大次数就会导致 job 最终失败了。

可用配置项：

-   restart-strategy  
    默认`fixed-delay`
-   restart-strategy.fixed-delay.attempts  
    默认 1，即当 job 失败时重试一次
-   restart-strategy.fixed-delay.delay  
    重试间隔，默认`1 s`

代码设置：

```java
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, 
  Time.of(10, TimeUnit.SECONDS) 
))

```

### 7.2.3 开启 Checkpoint 时还可选`failurerate`

开启 checkpoint 时 还可选`failurerate`, (也可以填`failure-rate`）。按失败率重启。

失败率是指每个时间间隔内发生的失败次数。

当失败率超过设定阈值，则 job 最终失败了。

可用配置项：

-   restart-strategy  
    failure-rate
-   restart-strategy.failure-rate.delay  
    默认 `1 s`，指定重启间隔时间，比如`20 s`
-   restart-strategy.failure-rate.failure-rate-interval  
    默认`1 min`，指定计算失败率的时间间隔
-   restart-strategy.failure-rate.max-failures-per-interval  
    默认 1， 失败率最大值

代码设置：

```java
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.failureRateRestart(
  3, 
  Time.of(5, TimeUnit.MINUTES), 
  Time.of(10, TimeUnit.SECONDS) 
))

```

## 7.3 Failover 策略

配置项为`jobmanager.execution.failover-strategy`，Failover 策略 相关接口为`FailoverStrategy`

-   可选值`full`，重启 job 所有 task。  
    RestartAllFailoverStrategy
-   默认值`region`，当 task 失败时，重启所有可能被该出错 task 影响的所有 task。  
    RestartPipelinedRegionFailoverStrategy

    该策略将 task 分组为不同 region，当任务失败时就计算必须重启以恢复 job 的最小范围的 region set。

    一个 Region 是指一个 pipeline，是一些 task 的集合，这些 task 通过 pipeline 交换数据通讯：

    -   DataStream 和 流式 Table/SQL job 的所有数据交换都是 Pipelined 形式的。
    -   批处理式 Table/SQL job 的所有数据交换默认都是 Batch 形式的。
    -   DataSet job 中的数据交换形式会根据 `ExecutionConfig` 中配置的 `ExecutionMode`决定。

    需要重启的 Region 的判断逻辑如下：

    -   出错 Task 所在 Region 需要重启。
    -   如果要重启的 Region 需要消费的数据有部分无法访问（丢失或损坏），产出该部分数据的 Region 也需要重启。
    -   需要重启的 Region 的下游 Region 也需要重启。这是出于保障数据一致性的考虑，因为一些非确定性的计算或者记录分发可导致同一个 ResultPartition 每次产生时包含的数据都不相同。
-   单点重启  
    可能无法保证一致性，但资源开销最小。

## 7.4 TaskManager 和 JobManager 高可用

可参考

-   [JobManager High Availability (HA)](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/jobmanager_high_availability.html)
-   [FlinkWiki-JobManager High Availability](https://cwiki.apache.org/confluence/display/FLINK/JobManager+High+Availability)
-   [Apache Flink 官方文档–作业管理器 (JM, JobManager) 高可用（HA）](https://blog.51cto.com/1196740/2349626)

### 7.4.1 TaskManager HA

由 JM 保证 TM 恢复期间的跨 TM 一致性。

### 7.4.2 JobManager 单点问题与 HA

因为 JM 用来调度任务、管理资源，所以如果他挂了就导致整个程序失败了，这就是 JM 单点问题。

我们可以配置 JM HA 高可用，即当 JM 失败后进行恢复，适用于 Standalone 和 Yarn Cluster 模式，这里我们主要分析 Yarn Cluster 模式。

### 7.4.3 Yarn Cluster 模式 JM ZK HA

#### 7.4.3.1 ZK HA 概述

相关源码在`ZooKeeperHaServices`。

ZK 存储的 Flink 相关数据结构如下：  
![](https://img-blog.csdnimg.cn/2020063016375952.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

-   flink  
    ZK 存放 Flink 数据的根目录，通过`high-availability.zookeeper.path.root`配置
-   cluster_id  
    某个 Flink 集群的数据根节点。包括 Standalone 模式和 Flink 集群模式的 Cluster
-   job-id  
    Flink 集群上运行的 job 信息，包括 Checkpoint 信息
-   checkpoints  
    触发 Checkpoint 快照后，每个 TM 会生成 Snapshot，并把对应的句柄传递给 JM，JM 会创建全局 Checkpoint 并持久化，最后将文件句柄写入 ZK。
-   persisted_job_graph  
    JM 接收到客户端提交的 JobGraph 后会生成并持久化 ExecutionGraph。具体来说，JM 会先将 ExecutionGraph 持久化到 ZK，再开始调度 Task 到 TM 执行。

#### 7.4.3.2 配置`yarn-site.xml`

配置`yarn-site.xml`内的`yarn.resourcemanager.am.max-attempts`

即 am 重试次数，要算上初次启动，默认值为 2，即当设置了 JM HA 后允许 JM 出错重启一次

#### 7.4.3.3 配置 `flink-conf.yaml` ZK

要保证 JM 服务自身在恢复期间的一致性，就必须使用第三方服务来存储少量的恢复用的元数据（比如最后提交的 Checkpoint 等），以及帮助选举哪个备选 JM 当 Leader，避免脑裂。

-   high-availability  
    定义了 HA 模式，可选项如下。

    -   默认`NONE`，不启用 HA
    -   `ZOOKEEPER`，ZK 模式的 HA
    -   恢复工厂类的全限定名的 HA
-   high-availability.cluster-id  
    定义了 Flink 集群 ID，用来隔离不同 Flink 集群。

    需要为 Standalone 模式设置，会在 YARN 和 Mesos 中自动推断。
-   high-availability.storageDir  
    Flink 持久化元数据的文件系统路径 (URL)
-   high-availability.zookeeper.path.root  
    Flink 存放集群节点元数据信息的 ZK 上的根目录路径，ZK 上存了指向该数据的指针信息。
-   high-availability.zookeeper.quorum  
    Flink HA 模式时使用的 ZK 法定节点数 (quorum)
-   high-availability.jobmanager.port

更多 Flink HA Zk 高级配置可参考[Advanced High-availability ZooKeeper Options](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/config.html#advanced-high-availability-zookeeper-options)

#### 7.4.3.4 配置`flink-conf.yaml` AM/JM 重启

配置`flink-conf.yaml`内的`yarn.application-attempts`和`yarn.application-attempt-failures-validity-interval`。

Flink On Yarn 时需要依赖 Yarn 来实现高可用，Yarn 会对失败挂掉的 JobManager(AM) 重启，最大重试次数的配置是 yarn-site.xml 的`yarn.resourcemanager.am.max-attempts`。

Flink 的 yarn-client 有一些配置可以控制在 container 失败的情况下的行为，也可通过`$FLINK_HOME/conf/flink-conf.yaml`或启动`yarn-session.sh`时以`-D`参数指定：

-   yarn.application-attempts  
    ApplicationMaster（运行着 JobManager）及其 TaskManager Container 的最大失败重试次数。

    当没有设置 HA 时默认值为 1，此时若 AM 挂掉就直接导致整个 flink yarn session 失败了。

    如果设为较高的值，使得可在失败时 Yarn 可多次尝试重启 AM，但会导致整个 Flink 集群重启，而 Yarn Client 会丢失到该 Flink Cluster 的连接，JobManager 的地址也会改变，所以需要在重启后手动设置 JobManager 的 host:port，所以推荐保留该值为 1。

    如果超过该阈值，不会再重启 AM，该 yarn session 上提交的任务也会全部停止。比如设为 5 时，代表初始化 1 次、最大重试 4 次。

    如果 Yarn 有一些额外操作（如抢占、硬件故障或 reboot、NM resync 等）需要重启 Application，则不会计算在次数内，参考[Jian Fang’s blog](http://johnjianfang.blogspot.de/2015/04/the-number-of-maximum-attempts-of-yarn.html)

    注意该值不能超过 yarn 中`yarn.resourcemanager.am.max-attempts`配置的最大重启次数。
-   yarn.application-attempt-failures-validity-interval  
    默认为 10000，单位毫秒

    定义`yarn.application-attempts`统计的时间窗口。如果设为 - 1 则为全局统计。

    也就是说，如果定义的时间窗口内统计到的 attempts 达到了阈值，则会停止尝试重启，导致 Job 失败！

    注意 Yarn Container 在不同版本中的行为不太相同，在大于等于 2.6.0 后应将`yarn.application-attempt-failures-validity-interval`设置为 Flink 的 Akka 超时值。

    相关连接：

    -   [PR](https://github.com/apache/flink/pull/8400)
    -   [实例](https://blog.csdn.net/weixin_43990680/article/details/104252477)

#### 7.4.3.5 Flink HA ZK Mode 配置实例

`flink-conf.yaml`:

```yaml
high-availability: zookeeper
high-availability.zookeeper.quorum: localhost:2181
high-availability.storageDir: hdfs:///flink/recovery
high-availability.zookeeper.path.root: /flink
yarn.application-attempts: 10

```

## 8.1 概述

如果在 Flink WebUI 上看到某个 task 发生`back pressure warning`，那么通俗地说，这就意味着数据生产速度大于下游算子消费速度，俗称反压 / 背压。

数据在 Flink 中从 Source 到 Sink 向下流动，而反压是反向传播的。

就拿最简单的`Source -> Sink`来说，如果观察到 Source 出现反压，则说明 Sink 消费速度已经跟不上 Source 生产速度了，所以向上游 Source 算子产生反压。

## 8.2 反压指标采样

Flink JM 会周期性地调用`Task.isBackPressured()`方法，以从运行中的 task 中采样，监控反压指标。

默认每次采样会为每个 task 每 50ms 采样 100 次，可在 WebUI 观察该指标（60 秒刷新一次，避免 TM 过载），比如 0.01 表示百分之 1 的样本发生反压。

该指标有几种情况：

-   OK: 0 &lt;= Ratio &lt;= 0.10
-   LOW: 0.10 &lt; Ratio &lt;= 0.5
-   HIGH: 0.5 &lt; Ratio &lt;= 1

```java
@Override
public boolean isBackPressured() {
	if (invokable == null || consumableNotifyingPartitionWriters.length == 0 || !isRunning()) {
		return false;
	}
	final CompletableFuture<?>[] outputFutures = new CompletableFuture[consumableNotifyingPartitionWriters.length];
	for (int i = 0; i < outputFutures.length; ++i) {
		outputFutures[i] = consumableNotifyingPartitionWriters[i].getAvailableFuture();
	}
	return !CompletableFuture.allOf(outputFutures).isDone();
}

```

目前判断是否产生背压是通过`output buffer`可用性，如果没有足够的 buffer 可用于输出说明该 Taskb 受到了来自下游的反压。

## 8.3 JM 配置

-   web.backpressure.refresh-interval  
    有效的反压结果被废弃并重新进行采样的时间间隔 (默认: 60000ms, 即 1 min)。
-   web.backpressure.num-samples  
    用于确定反压采样的样本数 (默认: 100)。
-   web.backpressure.delay-between-samples  
    用于确定反压采样的间隔时间 (默认: 50 (ms))。

本章主要介绍了 Flink 的一些基本概念，下一章讲 Flink 的下载、安装和启动，请点击[Flink 系列 2 - 安装和启动](https://blog.csdn.net/baichoufei90/article/details/82884554)

-   [Apache Flink](https://yuzhouwan.com/posts/20644/)

-   [Flink 在唯品会的实践应用](http://blog.itpub.net/31077337/viewspace-2200070/)

-   [大数据处理引擎 Apache Flink 全梳理](http://www.sohu.com/a/249855232_472869)

-   [Flink 架构、原理与部署测试](https://www.cnblogs.com/fanzhidongyzby/p/6297723.html)

-   [Apache Flink - 架构和拓扑](https://www.cnblogs.com/ooffff/p/9476032.html)

-   [Flink 原理及架构](https://blog.csdn.net/u013368491/article/details/81750650)

-   [Flink-StreamWindow](https://www.jianshu.com/p/ec7d20aa9836?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)

-   [Apache Flink](https://flink.apache.org/)

-   《深入理解 Flink - 实时大数据处理时间》

-   [Flink 的水印机制到底是怎么回事](https://bbs.csdn.net/topics/392567642)

-   [Flink 内部通信组件 Netty 和 Akka 的区别](https://blog.csdn.net/hxcaifly/article/details/88364907)

-   [为什么 spark 2.0 底层通信不用 Akka 而转用 netty ？](https://www.zhihu.com/question/61638635)

-   [Flink 的迁移之路，2 年处理效果提升 5 倍](https://www.sohu.com/a/337284991_617676)

-   [Flink 原理与实现：理解 Flink 中的计算资源](http://wuchong.me/blog/2016/05/09/flink-internals-understanding-execution-resources/)

-   [Flink 架构分析之资源分配](https://www.cnblogs.com/andyhe/p/10633692.html)

-   《Flink Runtime Architecture》
    -   作者：朱翥
    -   出处：阿里巴巴 - Flink 极客训练营

-   [Apache Flink 零基础入门（一 & 二）：基础概念解析](https://ververica.cn/developers/flink-basic-tutorial-1-basic-concept/)
    -   作者：陈守元、戴资力
    -   出处：Ververica

-   [Apache Flink 进阶教程（一）：Runtime 核心机制剖析](https://ververica.cn/developers/advanced-tutorial-1-analysis-of-the-core-mechanism-of-runtime/)
    -   作者：高赟
    -   出处：Ververica

-   [Apache Flink 零基础入门（七）：状态管理及容错机制](https://ververica.cn/developers/state-management/)
    -   作者：孙梦瑶
    -   出处：Ververica

-   [Flink 从 checkpoint 恢复，并行度改变后，状态重分配](https://blog.csdn.net/lvwenyuan_1/article/details/98511963)
    -   作者：lvwenyuan_1

-   [日均万亿条数据如何处理？爱奇艺实时计算平台这样做](https://mp.weixin.qq.com/s/IayYmMJaXorjzSVYDMjo1w)
    -   作者：梁建煌

-   [大数据框架：Spark 生态实时流计算](https://zhuanlan.zhihu.com/p/302901103)
    -   作者：加米谷大数据

-   [Structured Streaming VS Flink](https://mp.weixin.qq.com/s/F7jHlcc-91bUbCNx50hXww)
    -   作者：浪尖

-   [Apache Flink 进阶教程（三）：Checkpoint 的应用实践](https://ververica.cn/developers/advanced-tutorial-2-checkpoint-application-practice/)
    -   作者：唐云 (茶干)
    -   出处：Ververica

-   [Apache Flink 状态管理和容错机制介绍](https://ververica.cn/developers/introduction-to-state-management-and-fault-tolerance/)
    -   作者：施晓罡
    -   出处：Ververica

-   [流计算框架 Flink 与 Storm 的性能对比](https://ververica.cn/developers/stream-computing-framework/)
    -   作者：孙梦瑶
    -   出处：Ververica

-   [谈谈流计算中的『Exactly Once』特性](https://ververica.cn/developers/exactly-once-2/)
    -   作者：周凯波（宝牛）
    -   出处：Ververica

-   [Flink 状态管理详解：Keyed State 和 Operator List State 深度解析](https://www.cnblogs.com/felixzh/p/13167665.html)
    -   作者：大数据从业者
    -   出处：cnblogs

-   [一文搞懂 Flink 的 Exactly Once 和 At Least Once](https://ververica.cn/developers/flink-exactly-once-and-at-least-once/)
    -   作者：范瑞
    -   出处：Ververica

-   [如何在 Flink 中规划 RocksDB 内存容量？](https://ververica.cn/developers/how-to-plan-the-memory-capacity-of-rocksdb-in-flink/)
    -   作者：Stefan Richter
    -   翻译：毛家琦
    -   校对：胡争
    -   出处：Ververica

-   [Apache Flink 管理大型状态之增量 Checkpoint 详解](https://ververica.cn/developers/manage-large-state-incremental-checkpoint/)
    -   作者：Stefan Ricther & Chris Ward
    -   翻译：邱从贤（山智） 
        [https://blog.csdn.net/baichoufei90/article/details/82875496](https://blog.csdn.net/baichoufei90/article/details/82875496)
