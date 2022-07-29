# Apache Arrow - 大数据在数据湖后的下一个风向标 - 腾讯云开发者社区-腾讯云
## 介绍

根据[官方文档](https://arrow.apache.org/overview/)介绍，Arrow 是

> A language-independent columnar memory format for flat and hierarchical data, organized for efficient analytic operations on modern hardware like CPUs and GPUs. The Arrow memory format also supports zero-copy reads for lightning-fast data access without serialization overhead.

一句话概括，Arrow 用于系统间高效交互数据的组件。

### Arrow 的核心能力

Arrow 本身不是一个存储、执行引擎，它只是一个交互数据的基础库。比如可以用于以下组件

-   SQL 执行引擎 (e.g., Drill and Impala)
-   数据分析系统 (e.g., Pandas and Spark)
-   流和队列系统 (e.g., Kafka and Storm)
-   存储系统 (e.g., Parquet, Kudu, Cassandra and HBase)

## 背景

> 每个事物的产生发展都有其历史原因，如果抛开目的去 “学习”，犹如竹篮子打水 - 一场空
>
> <p align="right">- 我说的 ;)</p>

让我们回到 2008 年，故事从那开始...

### 起因

Wes McKinney 在 2008 年开启了 Pandas 项目，这个 python 中分析、操作数据的瑞士军刀。紧接着在 2014 年，Wes 加入 Cloudera 公司，并着手研究如何让 python 可以 “插入” 所有的大数据组件和数据库，但是每个系统都有自己操作数据的方式，于是：

> "Oh my gosh, I'm going to have to write a dozen different data converters to marshal data, convert data between Pandas each of these data processing systems from Spark, to Impala, to different file formats and HDFS, so it was basically this overwhelming problem."
>
> <p align="right">- Wes McKinney</p>

除此之外，在大数据科学领域，dataframe 的概念随处可见，每个框架都将 datafrme 作为高层定义，代表一个表、一系列 API... 但是其底层的实现天壤悬隔，Wes 完全无法复用代码。

**无法共享数据**、**无法共享代码**这两个大难题暂时困住了 Wes。

### 发展

Wes 开始设计一种 table middleware，作为不同组件交换数据的中间层，一种表接口的标准 (standardized table interface)。

接着来到 2015 年，Wes 团队遇到了 Jacques 和 Apache Drill 社区的小伙伴们，两伙人不谋而合，开始了合作。

由于业界没有统一规范的定义，他们合作的首个项目就是设计出了一个内存表视图的标准，并在不同语言都给出实现以证明可以在不同语言中共享数据，也就是说，你可以高效地将数据从 Java 到 C++，或者 Python。

自此，arrow 项目创立。

在项目早期，最重要的是设计出一套与语言无关的内存表结构，并一定要方便分析处理。除此之外，还需要将各种格式、类型的数据转换、转出为这个标准格式的库。最后，还需要一个计算处理的库，以便于直接基于 arrow 进行快速数据分析处理。

> "An important thing to remember about the project is that it's front-end agnostic. So it's not a new data frame library, like Pandas. It's not a new database."
>
> <p align="right">- Wes McKinney</p>

此外，Wes 在和 Apache Impala 团队合作的时候，发现 Impala 的代码中有大量和 pandas 做相似事情的片段，比如 CSV 序列化、反序列化的，I/O 子系统，自己的查询引擎，甚至自己的前端。在有了这样一个语言无关的内存数据格式，他们开始思考如何避免重复代码。

## 实现

故事讲完了，现在让我们一起来探索下 arrow 的设计。

面对不同语言、不同大数据组件之间的差异，首先我们肯定需要一个中间的表示来避免我们的后端直面差异，也就是前文提到的语言无关的内存表视图，这里就有一个必须挖掘的点，为了批量数据分析，我们应当选择**列式存储**。

![](https://ask.qcloudimg.com/http-save/5414999/e4bbfc4ec59f3743b9e901648175efff.png?imageView2/2/w/1620)

列存表查询

使用列存的方式不仅减少了扫描内存的 page 数，还可以利用现在计算机 SIMD(Single Instruction, Multiple Data) 指令进行加速。

* * *

扩展阅读 - [Daniel Abadi 的实验](http://dbmsmusings.blogspot.com/2017/10/apache-arrow-vs-parquet-and-orc-do-we.html)

Daniel 在亚马逊的 EC2 t2.medium 机器上创建了一个有 60,000,000 行数据的内存表。表由 6 个 int32 列组成，整个表大概由 1.5GB。他创建了行表和列表两个实例，并对两种表进行简单地 filter 某个值。

在未开 CPU 优化的情况下，得到结果：

无 SIMD

行表和列表查询耗时相差无几。对于行表，每行都需要扫描，即使只使用到第一列；对于列表则只需要扫描第一列，按理说列表应该是行表的 6 倍快，但是在这个实验中由于 CPU 是瓶颈，而不是内存发往 CPU 的数据。

但是开启 SIMD 后，结果如下：

开 SIMD

SIMD 可以同时比较多个数值（这里是 4 个数，差不多 3 倍快），减少打乱流水线的情况

* * *

现在我们可以继续考虑如何设计语言无关的内存表结构了

直接 IPC

Arrow 需要作为通用的传输结构

通过 arrow 交互

可是代码共享该如何实现呢？Arrow 不应该是 json、protobuf 之流，后者适用于磁盘层面的[数据存储](https://cloud.tencent.com/product/cdcs?from=10680)交互。Arrow 应当作为各个语言、组件中的一种数据格式库，应该是运行时的数据存储交互！直接可以操作数据，存取、计算：

数据操作

## Arrow 列格式

> :construction: 本节内容翻译整理自 apache/arrow 代码仓库中[Arrow Columnar Format 规范](https://github.com/apache/arrow/blob/master/docs/source/format/Columnar.rst)。

Arrow 列格式包含三部分：与语言无关的内存数据结构规范、元数据序列化以及一个用于序列化和通用数据传输的协议。

该列格式支持：

-   顺序访问的数据
-   O(1) 的随机读写
-   支持 SIMD，向量化操作友好
-   可重新定位而无 “pointer swizzling” 问题，允许在共享内存中 zero-copy

* * *

扩展阅读 - [pointer swizzling](https://en.wikipedia.org/wiki/Pointer_swizzling)

简单来说，内存中指针所指向的地址在写入磁盘（序列化）和从磁盘载入指针数据（反序列化）时，需要通过某种方式（swizzling 和 unswizzling）来使得指针存储的地址信息有效。

扩展阅读 - [零拷贝](https://en.wikipedia.org/wiki/Zero-copy)

zero-copy（零拷贝）不是指真的没有拷贝了，而是说减少了不必要的数据拷贝与上下文切换（系统调用）。比如正常情况下用户态进程希望从磁盘中读取数据并写入 socket，此时需要数据流经过磁盘 ->系统态内存 ->用户态内存 ->系统态内存 ->socket，发生了两次系统调用 (磁盘的 read() 和写入 socket 的 write())。使用系统提供的零拷贝函数 (比如 sendfile()) 则可以缩减为磁盘 ->系统态内存 ->socket。

* * *

在 Arrow 中，最基本的结构是 array(或者叫 vector，是由一列相同类型的值组成，长度必须已知，且有上限；换个常见的叫法是 field，字段)，每个 array 都有如下几个部分组成：

-   逻辑上的数据类型（记录 array 类型）
-   一列缓冲区（存放具体数字、null）
-   一个长度为 64 位带符号的整数（记录 array 长度，也可以是 32 位）
-   另一个长度为 64 位的带符号的整数（记录 null 值的数量）
-   （可选）字典（用于字典编码的 array）

Arrow 还支持嵌套 array 类型，其实就是一列 array 组成，它们叫做子 array(child arrays)。

### 物理内存布局

每一个逻辑类型都有一个定义明确的物理布局，Arrow 定义了如下物理布局：

-   Primitive(fixed-size)：用于存放具有相同长度的数值
-   Variable-size Binary：用于存放长度可变的数值。支持 32 位和 64 位的长度编码
-   Fixed-size List：嵌套类型，但是每个子 array 长度必须相同
-   Variable-size List：嵌套类型，每个子 array 长度可以不一致。支持 32 位和 64 位的长度编码
-   Struct：嵌套类型，由一组长度相同的命名子字段组成，但子字段的类型可以不一致。
-   Spare 和 Dense Union：嵌套类型，但是只有一组 array，每个数值的类型是子类型集合之一
-   Null：存放一组 null 值，逻辑类型只能是 null

#### 布局例子

本小节以 Fixed-size Primitive Layout 为例子讲述 Arrow 最基础的内存布局。

如前文所述，Primitive 类型的数值槽长度相同，只能存放固定长度的数值，可以是字节或者比特。

放到具体内存布局上，本类型包含一个连续的内存缓冲区，总大小则是槽宽\*长度（对于比特的槽宽，则需要四舍五入到字节）。

给出文档中一个 Int32 Array 的例子：

会这样表示：

    * Length: 5, Null count: 1
    * Validity bitmap buffer:

      |Byte 0 (validity bitmap) | Bytes 1-63            |
      |-------------------------|-----------------------|
      | 00011101                | 0 (padding)           |

    * Value Buffer:

      |Bytes 0-3   | Bytes 4-7   | Bytes 8-11  | Bytes 12-15 | Bytes 16-19 | Bytes 20-63 |
      |------------|-------------|-------------|-------------|-------------|-------------|
      | 1          | unspecified | 2           | 4           | 8           | unspecified |

其中有效性位图是用于记录每个值槽是否为空的。具体看[规范](https://arrow.apache.org/docs/format/Columnar.html#validity-bitmaps)。

剩下的布局都在 Primitive 布局上变化而来，具体看规范。

#### 布局使用的缓冲区

Arrow 的几种物理布局用到的缓冲区如下表所示：

\| 

Primitive

 \| 

validity

 \| 

data

 \|
\| 

Variable Binary

 \| 

validity

 \| 

offsets

 \| 

data

 \|
\| 

List

 \| 

validity

 \| 

offsets

 \|
\| 

Fixed-size List

 \| 

validity

 \| 

 \|
\| 

Struct

 \| 

validity

 \| 

 \|
\| 

Sparse Union

 \| 

type ids

 \| 

 \|
\| 

Dense Union

 \| 

type ids

 \| 

offsets

 \|
\| 

Null

 \| 

 \| 

 \|
\| 

Dictionary-encoded

 \| 

validity

 \| 

data (indices)

 \|

Arrow 如何实现 O(1) 读写的呢？

所有的物理布局底层都是用数组存储数据，并且会根据层级嵌套建立 offsets bitmap，当然就实现了 O(1) 的读写速度了。

### 逻辑类型

[Schema.fbs](https://github.com/apache/arrow/blob/master/format/Schema.fbs)定义了 Arrow 支持的逻辑类型，每种逻辑类型都会对应到一种物理布局。

### 序列化与 IPC

列式格式序列化时最原始的单位是 "record batch"(也就是一个表，table 啦)。一个 record batch 是一组有序的 array 的集合，被称为 record batch 的字段 (fields)。每个字段(field) 有相同的长度，但是字段的数据类型可以不一样。record batch 的字段名、类型构成了它的 schema。

本节描述一个协议，用于将 record batch 序列化为二进制流，并可以无需内存拷贝重构 record batch。

序列化时会分为这三部分：

-   Schema
-   RecordBatch
-   DictionaryBatch

这里我们只提及前两个。

布局

一个 schema message 和多个 record batch message 就能完整表示一个 record batch。其中 schema message 存储表结构，record batch message 存储字段 metadata 和字段值。

值得注意的是，record batch message 包含实际的数据缓冲区、对应的物理内存布局。

然后问题又来了，Arrow 为何无需 pointer-swizzling 即可实现流与数据转换的呢？答案就是 message 的 metadata 中存储了每个缓冲区的位置和大小，因此可以字节通过指针计算来重建 Array 数据结构，同时还避免了内存拷贝。

于是定义 IPC 流格式：

    <SCHEMA>
    <DICTIONARY 0>
    ...
    <DICTIONARY k - 1>
    <RECORD BATCH 0>
    ...
    <DICTIONARY x DELTA>
    ...
    <DICTIONARY y DELTA>
    ...
    <RECORD BATCH n - 1>
    <EOS [optional]: 0xFFFFFFFF 0x00000000>

由于这部分比较 “定义”，本文不展开讲，更详细请看[规范](https://arrow.apache.org/docs/format/Columnar.html#ipc-streaming-format)。

## Arrow Flight

近段时间 Arrow 最大的变化就是添加了 Flight，一个通用 C/S 架构的高性能数据传输框架。Flight 基于 gRPC 开发，从最开始重点就是优化 Arrow 格式数据。

Flight 的具体细节请看[官方文档](https://arrow.apache.org/blog/2019/10/13/introducing-arrow-flight/)。这里只介绍它的优势：

-   无序列化 / 反序列化：Flight 会直接将内存中的 Arrow 发送，不进行任何序列化 / 反序列化操作
-   批处理：Flight 对 record batch 的操作无需访问具体的列、记录或者元素
-   高并发：Flight 的吞吐量只收到客户端和服务端的吞吐量以及网络的限制
-   网络利用率高：Flight 使用基于 HTTP/2 的 gRPC，不仅是快

[官方给出的数据](https://www.dremio.com/is-time-to-replace-odbc-jdbc/)是 Flight 的传输大约是标准 ODBC 的 20-50 倍。

对每个 batch record 平均行数 256K 时，在单节点传输时的性能对比（因为 flight 多节点时可以平行传输数据流）：

性能对比

### 使用场景

最过经典的非[PySpark](https://arrow.apache.org/blog/2017/07/26/spark-arrow/)莫属，此外还有[sparklyr](https://arrow.apache.org/blog/2019/01/25/r-spark-improvements/)。

另外，[ClickHouse](https://cloud.tencent.com/product/cdwch?from=10680)也有计划实现 Arrow Flight 的 server 端，一旦落地可用，spark 与 clickhouse 交互就可以抛弃 3G 网般的 JDBC 了~

## 总结

本文从 Arrow 立项的背景入手，再到 Arrow 实现所需的设计，最后到 Arrow 具体 columnar 格式定义，介绍了 Arrow 的各种相关概念。最后补上一张图作为 Arrow 的优点、限制的总结：

总结

## 参考

1.  Wes 和 Jacques 的视频访谈: [Starting Apache Arrow](https://www.dremio.com/starting-apache-arrow/)
2.  Arrow 起名投票: [Vector Naming Discussion](https://docs.google.com/spreadsheets/d/1q6UqluW6SLuMKRwW2TBGBzHfYLlXYm37eKJlIxWQGQM/edit#gid=0)
3.  思路来源: [伴鱼技术团队](https://tech.ipalfish.com/blog/2020/12/08/apache_arrow_summary/)
4.  [Arrow Columnar Format](https://github.com/apache/arrow/blob/master/docs/source/format/Columnar.rst)
5.  [Arrow FAQ](https://arrow.apache.org/faq/)

原创声明，本文系作者授权腾讯云开发者社区发表，未经许可，不得转载。

如有侵权，请联系 cloudcommunity@tencent.com 删除。 
 [https://cloud.tencent.com/developer/article/1902699](https://cloud.tencent.com/developer/article/1902699)
