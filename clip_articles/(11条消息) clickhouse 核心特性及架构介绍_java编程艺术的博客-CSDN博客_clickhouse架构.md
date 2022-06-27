# (11条消息) clickhouse 核心特性及架构介绍_java编程艺术的博客-CSDN博客_clickhouse架构
**ClickHouse 的全称是：Click Stream,Data WareHouse 简称：[CH](https://clickhouse.com/docs/)**

是用于联机分析处理 (OLAP) 的列式数据库管理系统(DBMS)，提供千亿级[大数据](https://so.csdn.net/so/search?q=%E5%A4%A7%E6%95%B0%E6%8D%AE&spm=1001.2101.3001.7020)集的在线多维查询和分布式存储

## 适用场景

-   各种 OLAP 分析场景
    -   提供 PB 级数据的列式存储，千亿级结构化数据的快速查询能力，即便是在复杂查询的场景下，也能极快响应。
    -   支撑大规模快速搜索、高并发查询、多维度关联查询分析场景。
    -   随着数据体量的增大，优势会变得越为明显

## 不适用的场景

-   不支持事务
-   不擅长根据主键按行粒度进行查询，不应该把 CH 当作 key-value 数据库使用
-   不擅长按行删除数据

## 核心特性

-   完备的 DBMS 功能
-   列式存储和数据压缩
-   向量化执行引擎: 为了实现向量化执行，需要 CPU 的 SIMD 指令（Single Instruction Multiple Data 即用单条指令操作多条数据，它是通过数据并行以提高性能的一种实现方式（其他的还有指令级并行和线程级并行），它的原理是在 CPU 寄存器层面实现数据的并行操作。CH 目前利用 SSE4.2 指令集实现向量化执行
-   LLVM：
    -   简化了条件分支
    -   使用代码生成来替代数据加载
    -   内联虚函数的调用
-   关系模型和 SQL
-   多样化的表引擎：拥有 MergeTree、内存、文件、接口和其它 6 大类 20 多种表引擎，可以根据业务场景的要求，选择合适的表引擎使用。
-   多线程和分布式
-   多主架构 ：采用多 master 节点方式，消除中心节点性能瓶颈，大幅提升集群性能。
-   在线查询
-   数据分片： 将数据切片，多节点**多分片同时写入数据**，提升数据写入性能。
-   分布式查询：多分片的数据并发查询，提升数据查询性能
-   线性扩展：灵活添加节点，支持平滑扩展。

在计算机系统的体系结构中，存储系统是一种层次结构。典型服务器计算机的存储层次结构如图 1 所示。**存储媒介距离 CPU 越近，则访问数据的速度越快。所以基于向量化执行引擎，在寄存器上并行计算，为上层应用程序的性能带来了指数级的提升**  
![](https://img-blog.csdnimg.cn/2c3b0901c8d540e29620295f0100ca80.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5p625p6E5biI5b-g5ZOl,size_20,color_FFFFFF,t_70,g_se,x_16)

## 架构示意图

目前 ClickHouse 公开的资料相对匮乏，架构图示意如下：

-   Column 和 Field 是 ClickHouse 数据最基础的映射单元
    -   ClickHouse 按列存储数据，内存中的一列数据由一个 Column 对象表示，Column 对象分为接口和实现两个部分，采用泛化设计思路
    -   Field 对象使用了聚合的设计模式。在 Field 对象内部聚合了 Null、UInt64、String 和 Array 等 13 种数据类型及相应的处理逻辑。
-   DataType 数据的序列化和反序列化工作由 DataType 负责。
    -   IDataType 接口定义了许多正反序列化的方法，它们成对出现，例如 serializeBinary 和 deserializeBinary
    -   DataType 不直接负责数据的读取，而是转由从 Column 或 Field 对象获取
-   Block 与 Block 流：ClickHouse 内部的数据操作是面向 Block 对象进行的，并且采用了流的形式
    -   Block 对象可以看作数据表的子集。Block 对象的本质是由数据对象、数据类型和列名称组成的三元组，即 Column、DataType 及列名称字符串
    -   Column 提供了数据的读取能力，而 DataType 知道如何正反序列化，所以 Block 在这些对象的基础之上实现了进一步的抽象和封装，从而简化了整个使用的过程，仅通过 Block 对象就能完成一系列的数据操作。
    -   在具体的实现过程中，Block 并没有直接聚合 Column 和 DataType 对象，而是通过 ColumnWithTypeAndName 对象进行间接引用。
-   Table 直接使用 IStorage 接口指代数据表。
    -   表引擎是 ClickHouse 的一个显著特性，不同的表引擎由不同的子类实现，例如 IStorageSystemOneBlock (系统表)、StorageMergeTree ( 合并树表引擎 ) 和 StorageTinyLog ( 日志表引擎 ) 等
    -   在数据查询时，IStorage 负责根据 AST 查询语句的指示要求，返回指定列的原始数据
-   Parser 分析器负责创建 AST 对象
-   Interpreter 解释器则负责解释 AST，并进一步创建查询的执行管道。它们与 IStorage 一起，串联起了整个数据查询的过程
    -   针对 IStorage 返回的原始数据进一步加工、计算和过滤，则会统一交由 Interpreter 解释器对象处理
-   Functions 与 Aggregate Functions
-   Cluster 与 Replication： ClickHouse 的集群由分片 (Shard) 组成，而每个分片又通过副本 ( Replica ) 组成。

## [clickhouse-jdbc-bridge](https://hub.fastgit.org/ClickHouse/clickhouse-jdbc-bridge)

clickhouse-jdbc-bridge 充当一个无状态代理，将查询从 ClickHouse 传递到外部数据源。有了这个扩展，你可以在 ClickHouse 上运行实时跨多个数据源的[分布式](https://so.csdn.net/so/search?q=%E5%88%86%E5%B8%83%E5%BC%8F&spm=1001.2101.3001.7020)查询，这在某种程度上简化了为数据仓库、监控和完整性检查等构建数据管道的过程。

Known Issues / Limitation 详见官网介绍。  
![](https://img-blog.csdnimg.cn/666313734e6f4761b114f4f054f17eeb.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5p625p6E5biI5b-g5ZOl,size_20,color_FFFFFF,t_70,g_se,x_16) 
 [https://blog.csdn.net/penriver/article/details/122057506](https://blog.csdn.net/penriver/article/details/122057506)
