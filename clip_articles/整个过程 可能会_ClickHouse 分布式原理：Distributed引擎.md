# 整个过程|可能会_ClickHouse 分布式原理：Distributed引擎
[整个过程 | 可能会\_ClickHouse 分布式原理：Distributed 引擎](https://it.cha138.com/shida/show-483434.html) 

 _Posted 2023-01-29 [凌桓丶](https://it.cha138.com/au/show-228842.html)_

tags: [$.getjson 原理](https://it.cha138.com/zhuanti/show-991273.html)    [$.getjson 原理](https://it.cha138.com/zhuanti/show-991773.html)    [$http.post 上传原理](https://it.cha138.com/zhuanti/show-992237.html)    [%left 在编译原理是什么意思](https://it.cha138.com/zhuanti/show-993093.html)    [@Cacheable 原理](https://it.cha138.com/zhuanti/show-993800.html)   

篇首语：本文由小常识网 (cha138.com) 小编为大家整理，主要介绍了 ClickHouse 分布式原理：Distributed 引擎相关的知识，希望对你有一定的参考价值。

### 文章目录

-   [Distributed 引擎](#Distributed_3)
-   -   [分布式写入流程](#_18)
    -   -   [数据写入分片](#_29)
        -   [副本复制数据](#_59)
    -   [分布式查询流程](#_73)
    -   -   [多副本的路由规则](#_79)
        -   [多分片查询的流程](#_92)
        -   [使用 Global 优化分布式子查询](#Global_119)

* * *

Distributed 表引擎是分布式表的代名词，**它自身不存储任何数据，而是作为数据分片的透明代理，能够自动路由数据至集群中的各个节点**，所以 Distributed 表引擎需要和其他数据表引擎一起协同工作。

ClickHouse 并不像其他分布式系统那样，拥有高度自动化的分片功能。ClickHouse 提供了**本地表（Local Table）** 与 **分布式表（Distributed Table）** 的概念

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2010-55-20/6f34831e-95d0-44b0-8077-a81ed0ee32a4.jpeg?raw=true)

由 Distributed 表将数据写入多个分片

-   **本地表**：通常以\_local 为后缀进行命名。本地表是承接数据的载体，可以使用非 Distributed 的任意表引擎，一张本地表对应了一个数据分片。
-   **分布式表**：通常以\_all 为后缀进行命名。分布式表只能使用 Distributed 表引擎，它与本地表形成一对多的映射关系，日后将通过分布式表代理操作多张本地表。

## 分布式写入流程

在向集群内的分片写入数据时，通常有两种思路

-   **借助外部计算系统，事先将数据均匀分片，再借由计算系统直接将数据写入 ClickHouse 集群的各个本地表。** 
-   **通过 Distributed 表引擎代理写入分片数据。** 

第一种方案通常拥有更好的写入性能，因为分片数据是被并行点对点写入的。但是这种方案的实现主要依赖于外部系统，而不在于 ClickHouse 自身，所以这里主要会介绍第二种思路。为了便于理解整个过程，这里会将分片写入、副本复制拆分成两个部分进行讲解。

### 数据写入分片

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2010-55-20/abb48b2d-4cec-4429-bbaf-e21e583db46c.jpeg?raw=true)

由 Distributed 表将数据写入多个分片

1.  **在第一个分片节点写入本地分片数据**：首先在 CH5 节点，对分布式表 test_shard_2_all 执行`INSERT`查询，尝试写入 10、30、200 和 55 四行数据。执行之后分布式表主要会做两件事情：

    -   根据分片规则划分数据
    -   将属于当前分片的数据直接写入本地表 test_shard_2_local。
2.  **第一个分片建立远端连接，准备发送远端分片数据**：将归至远端分片的数据以分区为单位，分别写入 / test_shard_2_all 存储目录下的临时 bin 文件，接着，会尝试与远端分片节点建立连接。
3.  **第一个分片向远端分片发送数据**：此时，会有另一组监听任务负责监听 / test_shard_2_all 目录下的文件变化，这些任务负责将目录数据发送至远端分片，其中，每份目录将会由独立的线程负责发送，数据在传输之前会被压缩。
4.  **第二个分片接收数据并写入本地**：CH6 分片节点确认建立与 CH5 的连接，在接收到来自 CH5 发送的数据后，将它们写入本地表。
5.  **由第一个分片确认完成写入**：最后，还是由 CH5 分片确认所有的数据发送完毕。

可以看到，在整个过程中，Distributed 表负责所有分片的写入工作。本着谁执行谁负责的原则，在这个示例中，由 CH5 节点的分布式表负责切分数据，并向所有其他分片节点发送数据。

在由 Distributed 表负责向远端分片发送数据时，有**异步写**和**同步写**两种模式：

-   如果是异步写，则在 Distributed 表写完本地分片之后，`INSERT`查询就会返回成功写入的信息；
-   如果是同步写，则在执行`INSERT`查询之后，会等待所有分片完成写入。

### 副本复制数据

如果在集群的配置中包含了副本，那么除了刚才的分片写入流程之外，还会触发副本数据的复制流程。数据在多个副本之间，有两种复制实现方式：

-   **Distributed 表引擎**：副本数据的写入流程与分片逻辑相同，所以 Distributed 会同时负责分片和副本的数据写入工作。但在这种实现方案下，它很有可能会成为写入的单点瓶颈，所以就有了接下来将要说明的第二种方案。
-   **ReplicatedMergeTree 表引擎**：如果使用 ReplicatedMergeTree 作为本地表的引擎，则在该分片内，多个副本之间的数据复制会交由 ReplicatedMergeTree 自己处理，不再由 Distributed 负责，从而为其减负。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2010-55-20/7a8e03c4-6a9d-4a6f-81b5-8411cc3f02b1.jpeg?raw=true)

使用 Distributed 与 ReplicatedMergeTree 分发副本数据的对比示意图

## 分布式查询流程

与数据写入有所不同，在面向集群查询数据的时候，**只能**通过 Distributed 表引擎实现。当 Distributed 表接收到`SELECT`查询的时候，它会依次查询每个分片的数据，再合并汇总返回，流程如下：

### 多副本的路由规则

在查询数据的时候，如果集群中的某一个分片有多个副本，此时 Distributed 引擎就会通过负载均衡算法从众多的副本中选取一个，负载均衡算法有以下四种。

在 ClickHouse 的服务节点中，拥有一个全局计数器`errors_count`，当服务发生任何异常时，该计数累积加 1。

1.  **random（默认）**：random 算法会选择`errors_coun`t 错误数量最少的副本，如果多个副本的`errors_count`计数相同，则在它们之中**随机**选择一个。
2.  **nearest_hostname**：nearest_hostname 可以看作 random 算法的变种，首先它会选择`errors_count`错误数量最少的副本，如果多个副本的`errors_count`计数相同，**则选择集群配置中 host 名称与当前 host 最相似的一个。而相似的规则是以当前 host 名称为基准按字节逐位比较，找出不同字节数最少的一个**。
3.  **in_order**：in_order 同样可以看作 random 算法的变种，首先它会选择`errors_count`错误数量最少的副本，如果多个副本的`errors_count`计数相同，则**按照集群配置中 replica 的定义顺序逐个选择**。
4.  **first_or_random**：first_or_random 可以看作 in_order 算法的变种，首先它会选择`errors_count`错误数量最少的副本，如果多个副本的`errors_count`计数相同，**它首先会选择集群配置中第一个定义的副本，如果该副本不可用，则进一步随机选择一个其他的副本**。

### 多分片查询的流程

分布式查询与分布式写入类似，同样本着谁执行谁负责的原则，它会由接收`SELECT`查询的 Distributed 表，并负责串联起整个过程。

首先它会将针对分布式表的 SQL 语句，按照分片数量将查询拆分成若干个针对本地表的子查询，然后向各个分片发起查询，最后再汇总各个分片的返回结果。

```sql

SELECT * FROM distributor_table


SELECT * FROM local_table

```

如下图

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2010-55-20/f942f986-33b2-45c6-a2cf-56db8032805a.jpeg?raw=true)

对分布式表执行查询的执行计划

1.  **查询各个分片数据**：One 和 Remote 步骤是并行执行的，它们分别负责了本地和远端分片的查询动作。
2.  **合并返回结果**：多个分片数据均查询返回后，在执行节点将所有数据`union`合并

### 使用 Global 优化分布式子查询

如果现在有一项查询需求，例如要求找到同时拥有两个仓库的用户，应该如何实现？对于这类交集查询的需求，可以使用`IN`子查询，此时你会面临两难的选择：**`IN`查询的子句应该使用本地表还是分布式表？（使用`JOIN`面临的情形与`IN`类似）。** 

> **使用本地表的问题（可能查询不到结果）**

如果在 IN 查询中使用本地表时，如下列语句

```sql
SELECT 
	uniq(id) 
FROM 
	distributed_table 
WHERE 
	repo = 100 AND id IN (SELECT id FROM local_table WHERE repo = 200)

```

在实际执行时，分布式表在接收到查询后会将上述 SQL 替换成本地表的形式，再发送到每个分片进行执行，此时，每个分片上实际执行的是以下语句

```sql
SELECT 
	uniq(id) 
FROM 
	local_table 
WHERE 
	repo = 100 AND id IN (SELECT id FROM local_table WHERE repo = 200)

```

**那么此时查询的最终结果就有可能是错误的，因为在单个分片上只保存了部分的数据，这就导致该 SQL 语句可能没有匹配到任何数据**，如下图

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2010-55-20/340c7ae6-f4d7-410a-bf6b-3967cfc9f4af.jpeg?raw=true)

使用本地表作为 IN 查询子句的执行逻辑

> **使用分布式表的问题（查询请求被放大 N^2 倍，N 为节点数量）**

如果在 IN 查询中使用本地表时，如下列语句

```sql
SELECT 
	uniq(id) 
FROM 
	distributed_table 
WHERE 
	repo = 100 AND id IN (SELECT id FROM distributed_table WHERE repo = 200)

```

对于此次查询，每个分片节点不仅需要查询本地表，还需要再次向其他的分片节点再次发起远端查询，如下图

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2010-55-20/583999e5-78c0-4b51-b5cb-8619176ca23b.jpeg?raw=true)

IN 查询子句查询放大原因示意

因此可以得出结论，**在`IN`查询子句使用分布式表的时候，虽然查询的结果得到了保证，但是查询请求会被放大 N 的平方倍**，其中 N 等于集群内分片节点的数量，假如集群内有 10 个分片节点，则在一次查询的过程中，会最终导致 100 次的查询请求，这显然是不可接受的。

**使用 GLOBAL 优化查询**

为了解决查询放大的问题，我们可以使用`GLOBAL IN`或`GLOBAL JOIN`进行优化，下面就简单介绍一下 GLOBAL 的执行流程

```sql
SELECT 
	uniq(id) 
FROM 
	distributed_table 
WHERE 
	repo = 100 AND id GLOBAL IN (SELECT id FROM distributed_table WHERE repo = 200)

```

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2010-55-20/d927e0c6-0b7d-41fb-98df-e8d945febcc4.jpeg?raw=true)

使用 GLOBAL IN 查询的流程示意图

如上图，主要有以下五个步骤

1.  将`IN`子句单独提出，发起了一次分布式查询。
2.  将分布式表转 local 本地表后，分别在本地和远端分片执行查询。
3.  将`IN`子句查询的结果进行汇总，并放入一张临时的内存表进行保存。
4.  将内存表发送到远端分片节点。
5.  将分布式表转为本地表后，开始执行完整的 SQL 语句，`IN`子句直接使用临时内存表的数据。

**在使用 GLOBAL 修饰符之后，ClickHouse 使用内存表临时保存了`IN`子句查询到的数据，并将其发送到远端分片节点，以此到达了数据共享的目的，从而避免了查询放大的问题**。由于数据会在网络间分发，所以需要特别注意临时表的大小，`IN`或者`JOIN`子句返回的数据不宜过大。如果表内存在重复数据，也可以事先在子句 SQL 中增加`DISTINCT`以实现去重。

以上是关于 ClickHouse 分布式原理：Distributed 引擎的主要内容，如果未能解决你的问题，请参考以下文章

[Colocate Join ：ClickHouse 的一种高性能分布式 join 查询模型](https://it.cha138.com/python/show-81450.html)

[Deep learning III - II Machine Learning Strategy 2 - Bias and Variance with mismatched data distribu](https://it.cha138.com/javascript/show-92226.html)

[ClickHouse 分布式集群搭建指南](https://it.cha138.com/ios/show-23579.html)

[ClickHouse 分布式集群搭建指南](https://it.cha138.com/ios/show-17439.html)

[Clickhouse 系列 - 第二节 - 基本原理](https://it.cha138.com/ios/show-13023.html)

[大数据 ClickHouse 进阶：ClickHouse 使用场景和集群安装](https://it.cha138.com/android/show-47531.html)
