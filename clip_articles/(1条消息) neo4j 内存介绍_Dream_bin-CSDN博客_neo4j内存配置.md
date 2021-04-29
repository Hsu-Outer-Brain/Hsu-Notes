# (1条消息) neo4j 内存介绍_Dream_bin-CSDN博客_neo4j内存配置
_描述 Neo4j 内存配置和使用的不同方面_

内容翻译 neo4j 操作手册

### 1. 总览

![](https://img-blog.csdnimg.cn/20200112170011656.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RyZWFtX2Jpbg==,size_16,color_FFFFFF,t_70)

**1.1 操作系统内存**

必须保留一些内存以运行操作系统本身的进程。不可能显式配置应为操作系统保留的 RAM 数量，因为这是在配置页面缓存和堆空间之后仍保持可用的 RAM。但是，如果我们没有为操作系统留出足够的空间，它将开始交换到磁盘，这将严重影响性能。

1GB 是专用于运行 Neo4j 的服务器的良好起点。但是，在某些情况下，为 OS 保留的容量会大大超过 1GB，例如具有极大 RAM 的服务器。

**1.2 Lucene 索引缓存**

Neo4j 使用[Apache Lucene](https://lucene.apache.org/)为其某些索引功能。通过确保将尽可能多的索引缓存到内存中来优化索引查找性能。与 OS 内存类似，无法显式配置 Lucene 索引缓存。相反，我们估计了所需的内存，并确保在分配了页面缓存和堆缓存之后，还有足够的空间用于 Lucene 索引。

**1.3 页面缓存**

页面缓存用于缓存 Neo4j 数据和本机索引。将图形数据和索引缓存到内存中将有助于避免昂贵的磁盘访问并获得最佳性能。

用于指定 Neo4j 允许用于页面缓存的内存量的参数为：`dbms.memory.pagecache.size`。

**1.4 堆大小**

堆空间用于查询执行，事务状态，图形管理等。堆所需的大小非常取决于 Neo4j 用法的性质。例如，长时间运行的查询或非常复杂的查询可能需要比简单查询更大的堆。

一般来说，为了提高性能，我们要配置足够大的堆以支持并发操作。

在性能问题的情况下，我们可能必须调整查询并监视其内存使用情况，以确定是否需要增加堆。

堆内存大小由参数`dbms.memory.heap.initial_size`和决定`dbms.memory.heap.max_size`。建议将这两个参数设置为相同的值。这将有助于避免不必要的完整垃圾收集暂停

**1.5 交易状态**

事务状态是保存更新数据库中记录的事务中的数据和中间结果所需的内存。仅读取数据的查询不需要事务状态内存分配。默认情况下，事务状态是从堆或_on-heap_分配的。请注意，在堆内部配置事务状态时，不能指定其最大大小。

事务状态也可以被配置为被从堆中单独分配，或_关断堆_，通过使用参数`dbms.tx_state.memory_allocation`。当事务状态分配为堆外时，可以使用参数定义事务状态的最大大小 `dbms.tx_state.max_off_heap_memory`。请注意，事务状态内存未预先分配；它会根据数据库中活动的需要而增长和收缩。保持事务状态处于堆外状态对于以大型，写密集型事务为特征的应用程序特别有利。

### 2. 注意事项

#### 2.1 始终使用显式配置

为了更好地控制系统的行为，建议始终在[_neo4j.conf 中_](https://neo4j.com/docs/operations-manual/3.5/configuration/file-locations/)显式定义页面缓存和堆大小参数。如果未明确定义这些参数，则将在启动时根据可用的系统资源来计算一些启发式值。

#### 2.2 初始记忆建议

使用该`neo4j-admin memrec`命令可获得有关如何分配一定数量的内存的初步建议。可能需要调整值以适应每个特定的用例。

#### 2.3 检查数据库的内存设置

使用`neo4j-admin memrec`检查数据库的内存设置

我们希望估计数据库文件的总大小。

```
$neo4j-home> bin/neo4j-admin memrec --database=graph.db
...
...
...
# Lucene indexes: 6690m
# Data volume and native indexes: 17050m

```

我们可以看到，Lucene 索引占用了大约 6.7GB 的数据，而数据量和本机索引相加则占用了大约 17GB 的数据。

使用此信息，我们可以对内存配置进行完整性检查：

-   将数据量和本机索引的值与的值进行比较`dbms.memory.pagecache.size`。
-   比较 Lucene 索引的值与分配`dbms.memory.pagecache.size`和后剩余的内存量`dbms.memory.heap.initial_size`。

请注意，即使我们努力争取尽可能多地缓存数据和索引，但在某些生产系统中，对内存的访问受到限制，并且必须在不同区域之间进行协商。然后将进行一定数量的测试和调整，以找出可用内存的最佳划分。

**2.4 索引提供程序对内存使用量的影响**

在从 Neo4j 的早期版本升级之后，重建某些索引以利用新的索引功能是有利的。有关详细信息，请参见[第 11.2 节 “索引配置”](https://neo4j.com/docs/operations-manual/3.5/performance/index-configuration/)。索引的重建将改变内存利用率的分布。在具有许多索引的数据库中，可能已为 Lucene 保留了大量内存。重建后，可能有必要将某些内存分配给页面缓存。用于`neo4j-admin memrec --database`在重建索引之前和之后检查数据库。

### 3. 容量规划

在许多使用情况下，尝试缓存尽可能多的数据和索引是有利的。以下示例说明了根据我们是否已经在生产中运行或计划将来的部署来估计页面缓存大小的方法：

示例 11.2 估计现有 Neo4j 数据库的页面缓存

首先估算数据和索引的总大小，然后乘以某个因素（例如 20％）以允许增长。

```
$neo4j-home> bin/neo4j-admin memrec --database=graph.db
...
...
...
# Lucene indexes: 6690m
# Data volume and native indexes: 35050m

```

我们可以看到，数据量和本机索引总共占用了约 35GB。在我们的特定用例中，我们估计 20％将提供足够的增长空间。

`dbms.memory.pagecache.size` = 1.2 \*（35GB）= 42GB

我们通过将以下内容添加到_neo4j.conf_来配置页面缓存：

```
dbms.memory.pagecache.size=42GB

```

示例 11.3 估计新 Neo4j 数据库的页面缓存

在计划将来的数据库时，使用一部分数据进行导入非常有用，然后将结果存储大小乘以该部分再加上一定百分比的增长量。例如，导入数据的 1/100，并测量其数据量和本机索引。然后将该数字乘以 120 即可确定结果的大小，并允许 20％的增长。

假设我们已经将 1/100 的数据导入到测试数据库中。

```
$neo4j-home> bin/neo4j-admin memrec --database=graph.db
...
...
...
# Lucene indexes: 425.0
# Data volume and native indexes: 251100k

```

我们可以看到，数据量和本机索引合起来大约占用 250MB。我们估算结果，并预留 20％的增长空间：

`dbms.memory.pagecache.size` = 120 \*（250MB）= 30GB

我们通过将以下内容添加到_neo4j.conf_来配置页面缓存：

```
dbms.memory.pagecache.size=30G

```

 [https://blog.csdn.net/Dream_bin/article/details/103947355](https://blog.csdn.net/Dream_bin/article/details/103947355)
