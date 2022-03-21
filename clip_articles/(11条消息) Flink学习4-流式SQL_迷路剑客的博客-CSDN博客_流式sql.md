# (11条消息) Flink学习4-流式SQL_迷路剑客的博客-CSDN博客_流式sql
-   更多 Flink 系列文章请点击[Flink 系列文章](https://blog.csdn.net/baichoufei90/article/details/105674793)
-   更多大数据文章请点击[大数据好文推荐](https://blog.csdn.net/baichoufei90/article/details/90264272)

介绍 Flink Table Sql API 相关概念，还会提供一些例子。

本文大部分内容已经更新到 Flink 1.11。

## 1.1 概述

Flink 高层 API 有两种：Table 级别和 SQL 级别。两种 API 都是统一的处理批和流数据，也就是说对于无界、实时的流或者有界、记录型的流有着同样的处理语义，产生同样的结果。

**Table 和 SQL API 采用了`Apache Calcite`进行语句解析、验证和查询调优。**  
他们可以和 DataStream 及 DataSet API 无缝集成，并支持用户自定义的标量，聚合和表值函数。

Flink 的关系型 API 旨在简化数据分析，数据管道和 ETL 应用程序。

下面这个示例功能和 DataStream API 中的相同，也是展示一个 SQL 查询将一个点击流 session 化，然后对每个 session 中的点击数计数：

```java
SELECT userId, COUNT(*)
FROM clicks
GROUP BY SESSION(clicktime, INTERVAL '30' MINUTE), userId

```

这个 SQL 就是个流式处理 SQL，简洁，高效。

## 1.2 限制

虽然 flink 1.9.0 支持 DDL，但是尚不支持 Time 相关的元素。

可见:

-   [FLIP-66: Support Time Attribute in SQL DDL](https://cwiki.apache.org/confluence/display/FLINK/FLIP-66%3A+Support+time+attribute+in+SQL+DDL)
-   [JIRA-Support Time Attribute in SQL DDL](https://issues.apache.org/jira/browse/FLINK-14320)

**注意，1.10 开始版本中 DDL 已经支持 Time 相关内容！**

## 1.3 动态表 (Dynamic Tables)

### 1.3.1 概念

-   动态表主要是对应了传统有界数据概念上的表，对比如下：

    | /    | 关系代数 / SQL | 流处理                            |
    | ---- | ---------- | ------------------------------ |
    | 数据量  | 有界         | 无界                             |
    | 访问区域 | 查询时可访问完整数据 | 流式查询时不能访问完整数据，而是不断对新输入的数据访问并计算 |
    | 执行周期 | 执行一段时间后完毕  | 用不完结，持续执行，并根据收到的数据不断更新计算结果     |
-   物化视图 (Materialized Views) 与流 SQL  
    高级数据库用来应对流数据场景，他被定义为一条 SQL 查询，会缓存查询结果用来查询访问（与虚拟视图需要执行 SQL 不同，无需每次使用都执行查询计算）。有一种`Eager View Maintenance`技术被用来在基表（定义视图时的查询语句的原表）更新时更新物化视图。

    我们可以想象，数据库表一般拥有一个变化的由增删改组成的数据流（Changelog [Stream](https://so.csdn.net/so/search?q=Stream&spm=1001.2101.3001.7020)），那就需要持续查询处理该数据流来不断更新物化视图（物化视图本身就由 SQL 查询定义），也就是说物化视图就是流式 SQL 查询的结果。
-   动态表和持续查询（Continuous Query）  
    有了以上物化视图的相关概念，这里我们引出动态表。动态表是 Flink 流式 Table 和 SQL 的核心概念（一般是逻辑概念，并不一定要在查询执行中物化），会持续更新。

    需要注意的是，Flink 可以像查询静态表一样查询它们。

    动态表可像传统表一样执行查询（称为`持续查询 - Continuous Query`），这种持续查询的结果形成动态表，并且不断更新该动态表。这跟更新物化视图差不多，持续查询在语义上也和批量模式中查询任意时刻的输入表的快照相同。这个过程如下图：  
    ![](https://img-blog.csdnimg.cn/20200415200701974.png)

    1.  Input Stream 被转为动态表
    2.  持续查询在动态表上执行，生成新的动态表
    3.  生成的新动态表又被转回 Output Stream
-   动态表的存储  
    Flink 中动态表只是一个逻辑概念，Table 和 SQL API 用它来统一处理有界和无界数据，注意 Flink 是不会保存动态表数据的，而是存储在外部系统之中（如数据库、MQ、文件系统等）。

### 1.3.2 例子

#### 1.3.2.1 背景

比如一个点击流事件数据流，schema 如下：

```
[
  user:  VARCHAR,   // 用户名
  cTime: TIMESTAMP, // 访问 URL 的时间
  url:   VARCHAR    // 用户访问的 URL
]

```

#### 1.3.2.2 Group Aggregation

点击事件流随着事件不断流入，其实就相当于不断地对结果表的`INSERT`操作，可在逻辑上构成动态表（这种在流上定会以的表，其实在 Flink 内没有物化）。也就是说，本质上 Flink 使用仅有`INSERT`的 Chanelog 输入流来构建动态表。

下图左侧为点击流事件，右侧为转换后的动态表，随着点击流持续 Insert 而不断增长  
![](https://img-blog.csdnimg.cn/20200811110852576.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

首先考虑以下 sql，计算用户维度的 url 访问总量。

```sql
select user,count(url) as cnt from clicks group by user;

```

此时每条数据都会触发聚合计算，一般需要接入能更新的数据源，如果是不能更新的如 kafka 就是一条条单独数据，持续查询原表生成动态表如下图：  
![](https://img-blog.csdnimg.cn/2020041520265464.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

1.  查询刚开始时，输入表（左表）为空
2.  当第一行`[Mary, ./home]`输入 clicks 原表时，输出了首个结果 (INSERT)
3.  第二行输入时，计算后 INSERT 结果到动态表
4.  第三行又来一个 Mary 相关的，所以对动态表中的`[Mary, 1]`执行了 UPDATE 操作，更新为`[Mary, 2]`
5.  第四行记录也是新人，所以也是 INSERT 到动态表

#### 1.3.2.3 Window Aggregation

再来一个复杂一点的 sql，计算每个 1 小时的翻滚窗口内（以事件时间计算）的用户维度的 url 访问总量。

```sql
select user,TUMBLE_END(cTime, INTERVAL '1' HOURS) as endT,count(ur) as cnt from clicks 
group by user,TUMBLE(cTime, INTERVAL '1' HOURS);

```

此时数据会累积，直到水位到达窗口边界才会触发上一窗口计算，持续查询原表生成动态表如下图，这里颜色相同的为同一个 user：  
![](https://img-blog.csdnimg.cn/20200415203655461.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

-   根据 cTime 列不同，划分为不同窗口，分别进行计算。
-   每次计算后将结果 INSERT 到动态表

#### 1.3.2.4 Window Aggregation 对比 Group Aggregation

![](https://img-blog.csdnimg.cn/20200612184009656.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

`Group Aggregation`如果不配置会导致结果表无限增大，最终会出问题。所以一般使用该模式需要清理掉过期数据并设置最小、最大时间。

```java
StreamTableEnvironment.create(bsEnv, bsSettings).getConfig.setIdleStateRetentionTime(Time.days(1), Time.days(2))

```

这个用来指定空闲 State 被保留的的最小和最大时长，差值至少 5 分钟。超过最大时长未更新的 State 会被移除。

State 清理后，关于该已经被清理的 State 的新来的数据会被当做首条数据处理，可能导致覆盖之前的结果。

### 1.3.3 Update / Append

虽然以上例子都是聚合运算，但有很大不同：

-   普通聚合运算，是不断`UPDATE`结果表，用来定义结果表的 Changelog 流中包含`INSERT`和`UPDATE`。
-   窗口聚合运算的 Changelog 流仅包含`INSERT`，只能`Append`到结果表！

一个查询是`append-only`还是`update`的影响：

-   update 表通常需要维护更多状态，用来更新
-   两种表转为 Stream 的方式不同

### 1.3.4 表到流的转换

#### 1.3.4.1 概述

动态表支持`INSERT` 、`UPDATE`、`DELETE`操作来不断修改。所以动态表可能是：

-   只有不断被修改的单行数据
-   一个`insert-only`表，没有`UPDATE` / `DELETE`来修改表
-   或者介于以上两者之间

当要将动态表转为流或写入外部系统时，需要将将这三种操作编码。

需要注意的是，只有`Append`和`Retract`流模式下，才可以将动态表转为 DataStream。详见[table2datastream](#table2datastream)

使用`TableSink`接口来讲动态表发送到外部系统相关内容可见[TableSources and TableSinks](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/sourceSinks.html#define-a-tablesink)

还可参考

-   [Flink Table 的三种 Sink 模式](http://www.whitewood.me/2020/02/26/Flink-Table-%E7%9A%84%E4%B8%89%E7%A7%8D-Sink-%E6%A8%A1%E5%BC%8F/)

#### 1.3.4.2 Append Only Mode（增）

仅交互`INSERT`操作数据。

此时可通过将 insert 的新记录发送到下游来转换为流。

比如我们前面的例子 2，翻滚窗口计算后的变化数据流中仅有 INSERT 变化，全部 append 到结果动态表中。

#### 1.3.4.3 Retract Mode （增删改）

编码：  
交互`INSERT`（编码为`ADD Message`）和`DELETE`（编码为`RETRACT Message`）、`UPDATE`（对于修改前的行来说编码为`RETRACT Message`，对于正在修改的行来说编码为`ADD Message`）操作数据。

特点：

-   与 Upsert Mode 相反，Retract Mode 不需要定义 key。
-   每个 UPDATE 操作由两条消息（RETRACT 和 ADD）组成，效率较低。

将动态表转为 Retract Stream 的例子：  
![](https://img-blog.csdnimg.cn/20200415213858724.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

上图解读：

-   数据到来的顺序从上往下
-   第三行的 Mary 是第二次读到的 Mary，此时由于第一行已经读到过 Mary（`+ Mary,1`），所以需要更新。  
    此时需要先 DELETE 之前的`- Mary,1`，再 INSERT `+ Mary,2`，即 upsert 更新由两个操作实现。

#### 1.3.4.4 Upsert Mode（增删改）

编码：  
交互`INSERT`和`UPDATE`（编码为`UPSERT Message`）和`DELETE`（编码为`Delete Message`）操作数据。

特点：

-   Upsert Mode 需要一个唯一的 key（可能是组合的），用来传播 UPDATE 事件。  
    具体来说，**外部连接器需要了解该唯一 key 属性，才能正确处理消息。** 
-   Upsert Mode 和 Retract Mode 都支持增删改但编码不同  
    Upsert Mode 中的 UPDATE 事件使用单条 UPSERT 消息进行编码，而 Retract Mode 中的 UPDATE 由两条消息（RETRACT 和 ADD）组成，因此 Upsert 模式效率更高。

例子：  
比如我们前面的例子 1，计算后的变化数据流中有 INSERT 和 UPDATE 变化，会持续更新动态表中之前的结果。此时比起 Append Only Mode 来说会需要维护更多的状态信息，用来更新。但相比 Retract Mode，本模式在修改数据时可直接 Upsert，无需两个操作即可完成。

将动态表转为 Upsert Stream 的示意图如下：  
![](https://img-blog.csdnimg.cn/20200415214429364.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

-   比如第三行的第二次遇到 Mary，就直接修改即完成。
-   注意看，上面的`user`列就是唯一 KEY！

### 1.3.5 动态表查询限制

许多 (但不是全部) 语义上有效的查询可以作为流上持续查询进行代价评估。有些查询代价太高而无法计算，这可能是由于它们需要维护的状态大小，也可能是由于计算以及更新代价太高：

-   状态大小  
    持续运行的流式应用，有的需要更新之前输出的结果，那就必须维护所有输出行。

    比如前面的例子 1 就需要一直维护每个用户的 URL 总和以便每次计算时更新。随着时间流逝，注册用户越来越多，有的网站甚至会为未注册用户也绑定一个用户名，导致维护的状态会越来越大，最终可能导致这个持续查询失败！
-   计算和更新  
    一些持续查询需要重新计算和更新大量结果行，甚至即使只添加或更新了一行数据。这种情况显然不适合持续查询。

    比如以下 SQL，以每个用户最后一次点击时间来为每个用户计算 RANK：

    ````sql
    	SELECT user, RANK() OVER (ORDER BY lastLogin)
    	FROM (
    	  SELECT user, MAX(cTime) AS lastAction FROM clicks GROUP BY user
    	);
    	```
    此时一旦clicks表接收到一行新数据，则用户的`lastAction`字段就会更新，而新的rank也需要被计算。这会导致所有人的排名都重新计算！有一些参数可以控制维持状态大小和结果准确度之间的权衡：[Query Configuration](https:

    ````

## 1.4 Catalog

可参考

-   [Catalogs](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/catalogs.html)
-   [HiveCatalog](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/hive/hive_catalog.html)

### 1.4.1 简介

Catalog 用来存储元数据，包括数据库、表、分区、视图、函数以及需要访问数据库或外部系统数据的信息。

Catalog 提供的元数据包括两方面：

-   临时元数据  
    如临时表、通过 TableEnvironment 注册的临时 UDF 等
-   持久化元数据  
    如存在 HiveMetastore 中的数据

Catalog 提供统一的 Table API 和 SQL 来管理、访问元数据。

### 1.4.2 元数据映射

Catalog 使得用户可直引用其他数据系统的元数据，映射到 Flink。比如直接将 JDBC 表映射到 Flink 表，不需要重写 FlinkSql DDL 就能直接查询外部表了，十分方便。

### 1.4.3 可用的 Catalog

#### 1.4.3.1 概述

目前有

-   GenericInMemoryCatalog  
    基于内存，所有元数据只在 session 的生命周期内可用。

    注意，大小写敏感。
-   JdbcCatalog  
    使用 JDBC 协议连接管理元数据，目前只有`PostgresCatalog`实现。

    相见[JdbcCatalog documentation](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/connectors/jdbc.html)
-   HiveCatalog  
    可用来拿出 Flink 元数据、作为接口读写 Hive 现有元数据。

    注意，存储时都是小写。
-   [自定义 Catalog](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/catalogs.html#user-defined-catalog)

#### 1.4.3.2 HiveCatalog

可参考

-   [HiveCatalog](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/hive/hive_catalog.html)
-   [Hive 集成方法](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/hive/index.html)

可以让 Flink 使用 Hive Catalog 存储 Flink SQL 元数据。

可以在 Hive 命令行中使用`DESCRIBE FORMATTED`命令查看表的元数据，如果是`is_generic=true`代表是 Flink 专用表，这种表只能由 Flink 读写使用，不要用 Hive 去读写。

也可以直接使用 Flink 读写 Hive 表数据。

需要将以下包放入`$FLINK_HOME/lib`:

-   flink-connector-hive_2.11-1.11.0.jar
-   hive-exec-2.3.3.jar

使用 hive 存储 flink sql 元数据的时候，还可以做一些高级配置，可参考[Flink-hive_catalog](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/hive/)

简单来说，可以修改`$FLINK_HOME/conf/sql-client-defaults.yaml`:

```yaml
execution:
    planner: blink
    type: streaming
    ...
    current-catalog: zeppelin-hive  
    current-database: flink_meta
    
catalogs:
   - name: myhive
     type: hive
     hive-conf-dir: /opt/hive-conf  
     hive-version: 2.3.3

```

然后可以启动`sql-client`，会默认读取`$FLINK_HOME/conf/sql-client-defaults.yaml`：

```shell
sql-client.sh embedded

```

如果报错`flink sql-client ClassNotFoundException: LogFactory`，则还需要设置`HADOOP_CLASSPATH`。

出现如下界面表示 sql-client 启动成功：  
![](https://img-blog.csdnimg.cn/2020070717104529.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

关于 sql-client，可参考

-   [Flink 与 Hive 的磨合期](https://developer.aliyun.com/article/761469)
-   [SQL Client Beta](https://ci.apache.org/projects/flink/flink-docs-master/dev/table/sqlClient.html)
-   [Flink SQL Client 初探](https://cloud.tencent.com/developer/article/1632965)

### 1.4.4 Catalog 注册和创建

#### 1.4.4.1 SQL

-   创建和注册  
    默认有内存中的 default_catalog，包括一个 default_database

```java、
// Create a HiveCatalog 
val catalog = new HiveCatalog("myhive", null, "<path_of_hive_conf>", "<hive_version>")

// Register the catalog
tableEnv.registerCatalog("myhive", catalog)

// Create a catalog database
tableEnv.executeSql("CREATE DATABASE mydb WITH (...)")

```

-   切换当前 catalog 和数据库  
    Flink 总是通过当前 catalog 和数据库来搜索表、视图、UDF，可切换使用的 catalog 和数据库

```sql
USE CATALOG myCatalog;
USE myDB;

```

```java
tableEnv.useCatalog("myCatalog");
tableEnv.useDatabase("myDb");

```

-   查询非当前 catalog 的表

```sql
SELECT * FROM not_the_current_catalog.not_the_current_db.my_table;

```

-   创建表

```sql
tableEnv.executeSql("CREATE TABLE mytable (name STRING, age INT) WITH (...)")

```

-   展示 catalog

```sql
show catalogs;

```

```java
tableEnv.listCatalogs();

```

-   展示数据库

```sql
show databases;

```

```java
tableEnv.listDatabases();

```

-   展示表

```sql
show tables;

```

```java

tableEnv.listTables() 

```

也可以用`yaml`来配置 catalog，再执行 sql.

#### 1.4.4.2 TableAPI

```java
import org.apache.flink.table.api._
import org.apache.flink.table.catalog._
import org.apache.flink.table.catalog.hive.HiveCatalog
import org.apache.flink.table.descriptors.Kafka

val tableEnv = TableEnvironment.create(EnvironmentSettings.newInstance.build)


val catalog = new HiveCatalog("myhive", null, "<path_of_hive_conf>", "<hive_version>")


tableEnv.registerCatalog("myhive", catalog)


catalog.createDatabase("mydb", new CatalogDatabaseImpl(...))


val schema = TableSchema.builder()
    .field("name", DataTypes.STRING())
    .field("age", DataTypes.INT())
    .build()

catalog.createTable(
        new ObjectPath("mydb", "mytable"), 
        new CatalogTableImpl(
            schema,
            new Kafka()
                .version("0.11")
                ....
                .startFromEarlist()
                .toProperties(),
            "my comment"
        ),
        false
    )
    
val tables = catalog.listTables("mydb") 

```

参考：

-   [Table API](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/tableApi.html)

Table&Sql API 可参考[Concepts & Common API](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/common.html)

## 2.1 概述

Table&Sql API 核心在于`Table`，作为输入和输出。

## 2.2 Scala Api 示例

### 2.2.1 基本代码结构

```scala

val tableEnv = ... 


tableEnv.connect(...).createTemporaryTable("table1")

tableEnv.connect(...).createTemporaryTable("outputTable")


val tapiResult = tableEnv.from("table1").select(...)

val sqlResult  = tableEnv.sqlQuery("SELECT ... FROM table1 ...")


val tableResult = tapiResult.executeInsert("outputTable")
tableResult...

```

### 2.2.2 Blink 例子

批流统一的代码结构。

以下是使用 Blink 时的 scala Streaming 示例：

```java



import org.apache.flink.streaming.api.scala.StreamExecutionEnvironment
import org.apache.flink.table.api.EnvironmentSettings
import org.apache.flink.table.api.bridge.scala.StreamTableEnvironment


val bsEnv = StreamExecutionEnvironment.getExecutionEnvironment
val bsSettings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build()

val tableEnv = StreamTableEnvironment.create(bsEnv, bsSettings)


val bbSettings = EnvironmentSettings.newInstance().useBlinkPlanner().inBatchMode().build()
val bbTableEnv = TableEnvironment.create(bbSettings)



tableEnv.connect(...).createTemporaryTable("table1")

tableEnv.connect(...).createTemporaryTable("outputTable")


val tapiResult = tableEnv.from("table1").select(...)

val sqlResult  = tableEnv.sqlQuery("SELECT ... FROM table1 ...")


val tableResult = tapiResult.executeInsert("outputTable")
tableResult...

```

使用 Blink 时的 scala Batch 示例：

```java



import org.apache.flink.table.api.{EnvironmentSettings, TableEnvironment}

val bbSettings = EnvironmentSettings.newInstance().useBlinkPlanner().inBatchMode().build()
val bbTableEnv = TableEnvironment.create(bbSettings)

```

## 2.3 TableEnvironment

### 2.3.1 概述

TableEnvironment 职责如下：

-   注册 table 到内部 catalog
-   注册 catalog
-   加载插件化的 module
-   执行 sql 查询
-   注册 UDF
-   将 DtaStream/DataSet 转为 Table
-   持有指向 ExecutionEnvironment 或 StreamExecutionEnvironment 的引用

每个 Table 都会绑定到某个 TableEnvironment，不能在同一个查询里跨 TableEnvironment 执行连接表操作（如 join/union）。

### 2.3.2 创建和配置

```java

val bsEnv = StreamExecutionEnvironment.getExecutionEnvironment

val bsSettings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build()

val tableEnv = StreamTableEnvironment.create(bsEnv, bsSettings)

```

## 2.4 与 DataStream/DataSet 集成

### 2.4.1 概述

参见[Integration with DataStream and DataSet API](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/common.html#integration-with-datastream-and-dataset-api)

-   在流处理方面，两种 Planner 都可以与 DataStream API 集成使用。
-   只有旧 planner 可以与 DataSet API 结合。
-   在批处理方面，Blink Planner 不能同两种 DataStream 或 DataSet 集成。

### 2.4.2 隐式转换

Scala Table API 含有对 DataSet、DataStream、 Table 的隐式转换，DataStream API 需要导入：

```java
import org.apache.flink.table.api.bridge.scala._
import org.apache.flink.api.scala._

```

### 2.4.3 通过 DataSet / DataStream 创建临时视图

视图的 schema 依赖于 DataStream / DataSet 注册的数据类型，详见[mapping of data types to table schema](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/common.html#mapping-of-data-types-to-table-schema)。

**注意： 通过 DataStream 或 DataSet 创建的视图只能注册为临时视图。** 

```java


val tableEnv: StreamTableEnvironment = ... 

val stream: DataStream[(Long, String)] = ...


tableEnv.createTemporaryView("myTable", stream)


tableEnv.createTemporaryView("myTable2", stream, 'myLong, 'myString)

```

### 2.4.4 将 DataStream / DataSet 转换成 Table

```java


val tableEnv = ... 

val stream: DataStream[(Long, String)] = ...


val table1: Table = tableEnv.fromDataStream(stream)


val table2: Table = tableEnv.fromDataStream(stream, $"myLong", $"myString")

val stream2: DataStream[(String, Int)] = ...



val table = stream2.toTable(tEnv, 'name, 'amount)

```

### 2.4.5 将 Table 转换成 DataStream / DataSet

#### 2.4.5.1 类型转换

详细请参考[Mapping of Data Types to Table Schema](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/common.html#mapping-of-data-types-to-table-schema)

Table 可转为 DataStream/DataSet，这样可以使自定义 DataStream/DataSet 来使用 Table&SQL 查询结果。

转变时，需要指定生成的 DataStream/DataSet 的数据类型，通常来说最方便的转换类型是`Row`，所有选项如下：  
![](https://img-blog.csdnimg.cn/20200808234634676.png)

#### 2.4.5.2 Table 转为 DataStream

这里说的 Table 就是流式查询的结果，会随着新数据流入输入流中执行查询被不断地动态更新。因此，这种由动态查询转换成的 DataStream 需要对表的`UPDATE`进行编码。Table 转为 DataStream 有两种模式：

-   Append Mode  
    仅当动态表只被`INSERT`操作改动被使用，是`append-only`的，而且旧的转换结果不会再被更新。
-   Retract Mode  
    任何情况下都可以使用，使用 `boolean` 值对`INSERT`和`DELETE`操作的改动进行标记，`True`表示 INSERT, `False` 表示 DELETE.。

示例代码：

```java


val tableEnv: StreamTableEnvironment = ... 


val table: Table = ...


val dsRow: DataStream[Row] = tableEnv.toAppendStream[Row](table)


val dsTuple: DataStream[(String, Int)] dsTuple = 
  tableEnv.toAppendStream[(String, Int)](table)





val retractStream: DataStream[(Boolean, Row)] = tableEnv.toRetractStream[Row](table)

```

Table 转为 DataStream 后，必须使用`StreamExecutionEnvironment.execute()`来运行该 DataStream 程序。

#### 2.4.5.3 Table 转为 DataSet

```java


val tableEnv = BatchTableEnvironment.create(env)


val table: Table = ...


val dsRow: DataSet[Row] = tableEnv.toDataSet[Row](table)


val dsTuple: DataSet[(String, Int)] = tableEnv.toDataSet[(String, Int)](table)

```

Table 转为 DataSet 后，必须使用`ExecutionEnvironment.execute()`来运行该 DataSet 程序。

## 2.5 创建表

### 2.5.1 视图 - 虚拟表

```java

val tableEnv = ... 


val projTable: Table = tableEnv.from("X").select(...)


tableEnv.createTemporaryView("projectedTable", projTable)

```

内部其实就是封装了逻辑查询计划，但如果使用了 Blink Planner，则在被多表引用时也只会被执行一次，结果可被多表共享。

### 2.5.2 创建 connector 表

其实就是外表，描述了外部存储系统

```java
tableEnvironment
  .connect(...)
  .withFormat(...)
  .withSchema(...)
  .inAppendMode()
  .createTemporaryTable("MyTable")

```

### 2.5.3 identifier

包括 catalog/database/table，可直接指定前两者，则使用可省略

```java

val tEnv: TableEnvironment = ...;
tEnv.useCatalog("custom_catalog")
tEnv.useDatabase("custom_database")

val table: Table = ...;



tableEnv.createTemporaryView("exampleView", table)



tableEnv.createTemporaryView("other_database.exampleView", table)



tableEnv.createTemporaryView("`example.View`", table)



tableEnv.createTemporaryView("other_catalog.other_database.exampleView", table)

```

## 2.6 查询

```sql

val tableEnv = ... 




val orders = tableEnv.from("Orders")

val revenue = orders
  .filter($"cCountry" === "FRANCE")
  .groupBy($"cID", $"cName")
  .select($"cID", $"cName", $"revenue".sum AS "revSum")




```

## 2.7 Emit 输出表

Table 最终被输出到`TableSink`，他是一个通用接口，支持多种文件 format、存储系统、MQ 等。

-   流 Table 可写入 AppendStreamTableSink / RetractStreamTableSink / UpsertStreamTableSink
-   批 Table 只能写入 BatchTableSink。

```java

val tableEnv = ... 


val schema = new Schema()
    .field("a", DataTypes.INT())
    .field("b", DataTypes.STRING())
    .field("c", DataTypes.LONG())

tableEnv.connect(new FileSystem("/path/to/file"))
    .withFormat(new Csv().fieldDelimiter('|').deriveSchema())
    .withSchema(schema)
    .createTemporaryTable("CsvSinkTable")


val result: Table = ...


result.executeInsert("CsvSinkTable")

```

参考

-   [SQL API](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/sql/)
-   [依赖介绍](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/)

## 3.1 Table API & SQL 架构

![](https://img-blog.csdnimg.cn/20200330224314293.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

## 3.2 执行原理

-   [Flink 原理与实现：Table & SQL API](https://ververica.cn/developers/flink-internals-table-and-sql-api/)  
    阿里 - 伍翀
-   [基于 Flink1.8 深入理解 Flink Sql 执行流程 + Flink Sql 语法扩展](https://blog.csdn.net/super_wj0820/article/details/95623380)
-   [Flink | 使用 Calcite 做 Sql 语法解析](https://mp.weixin.qq.com/s/y4HhPZfQqB2YMnhLfDIHTg)

在 1.11 的 Flink 中，Table API 不再被翻译为 DataStream API，而是翻译为 Transformation。  
![](https://img-blog.csdnimg.cn/20201014161903346.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

Flink 1.11 Sql 执行流程如下：  
![](https://img-blog.csdnimg.cn/2020101416135048.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

1.  使用 Calcite 解析 SQL，这个过程会用到 Catalog（表名、列信息、类型信息、PK 信息等，可用于逻辑计划的后续优化），翻译为 Logic Plan
2.  将 Logic Plan 按一系列规则（如谓词下推、投影下推、Join 重排等）翻译为 Physical Plan
3.  通过 Code Generation 将 Physical Plan 转为 Transformation DAG 图  
    这一阶段也有许多优化
4.  Transformation 转为 StreamGraph->JobGraph 提交到 Flink Cluster
5.  JobGraph 生成带并行的信息的 ExecutionGraph，最后生成虚拟的物理执行图运行

## 3.3 Blink 和 Old Planner

-   Blink 是阿里贡献的
-   Blink 将 batch job 也作为特殊的流来处理（会被转为 DataStream 程序处理），所以不能转为 DataSet。
-   Blink 不支持`BatchTableSource`，而是使用`StreamTableSource`
-   Sink 处理不同：
    -   Old Planner 将 Job 中的多个 Sink 各自单独优化为一个 DAG，每个 DAG 相互独立；
    -   Blink 将多个 Sink 优化为一个 DAG。
-   Table&Sql 翻译执行不同
    -   Blink 将多个他们全部转为 DataStream 程序，不论输入是流还是批。一个查询在 Flink 内部被描述为逻辑查询计划，并被翻译为两个阶段：
        -   逻辑执行计划优化
        -   转换为一个 DataStream 程序  
            不同语句的 Blink 翻译时机转自官网：  
            ![](https://img-blog.csdnimg.cn/20200724161051397.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
-   我们实践中现在都用 Blink 了

## 3.4 示例

### 3.4.1 基本结构

批流统一的代码结构，以下是使用 Blink 时的 scala 示例：

```java

val bsEnv = StreamExecutionEnvironment.getExecutionEnvironment
val bsSettings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build()

val tableEnv = StreamTableEnvironment.create(bsEnv, bsSettings)


val bbSettings = EnvironmentSettings.newInstance().useBlinkPlanner().inBatchMode().build()
val bbTableEnv = TableEnvironment.create(bbSettings)



tableEnv.connect(...).createTemporaryTable("table1")

tableEnv.connect(...).createTemporaryTable("outputTable")


val tapiResult = tableEnv.from("table1").select(...)

val sqlResult  = tableEnv.sqlQuery("SELECT ... FROM table1 ...")


val tableResult = tapiResult.executeInsert("outputTable")
tableResult...

```

参见[Integration with DataStream and DataSet API](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/common.html#integration-with-datastream-and-dataset-api)可查看 Table API 和 DataStream、DataSet 整合、转换的例子。

### 3.4.2 创建 connector 表

其实就是外表，描述了外部存储系统

```sql
tableEnvironment.executeSql("CREATE [TEMPORARY] TABLE MyTable (...) WITH (...)")

```

### 3.4.3 SQL

可参考[FlinkSQL 查询](https://ci.apache.org/projects/flink/flink-docs-release-1.11/zh/dev/table/sql/queries.html)

查询指定表示例：

```java

val tableEnv = ... 




val revenue = tableEnv.sqlQuery("""
  |SELECT cID, cName, SUM(revenue) AS revSum
  |FROM Orders
  |WHERE cCountry = 'FRANCE'
  |GROUP BY cID, cName
  """.stripMargin)




```

更新指定表示例：

```sql

val tableEnv = ... 





tableEnv.executeSql("""
  |INSERT INTO RevenueFrance
  |SELECT cID, cName, SUM(revenue) AS revSum
  |FROM Orders
  |WHERE cCountry = 'FRANCE'
  |GROUP BY cID, cName
  """.stripMargin)

```

## 3.5 配置

-   [Table-Configuration](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/config.html)
-   [通用配置](https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/config.html)

## 3.6 Catalog 和 Identifier 标识符

### 3.6.1 概述

由`TableEnvironment`维护着一个由标识符（identifier）创建的表 catalog 的映射，标识符由`catalog`、`database`、`table`三部分组成。

用户可以指定默认的 catalog 和 database

可以通过 sql 创建 catalog

```sql
CREATE CATALOG catalog_name
  WITH (key1=val1, key2=val2, ...)

```

创建 DATABASE

```sql
CREATE DATABASE [IF NOT EXISTS] [catalog_name.]db_name
  [COMMENT database_comment]
  WITH (key1=val1, key2=val2, ...)

```

Catalog 可参考[catalog](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/catalogs.html)

## 3.7 Table 和 View 的创建

参考

-   [CREATE Statements](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/sql/create.html#create-catalog)
-   Flink Table 描述了外部数据，比如文件、数据库、消息队列等。

    ```sql
    CREATE TABLE [catalog_name.][db_name.]table_name
      (
        { <column_definition> | <computed_column_definition> }[ , ...n]
        [ <watermark_definition> ]
        [ <table_constraint> ][ , ...n]
      )
      [COMMENT table_comment]
      [PARTITIONED BY (partition_column_name1, partition_column_name2, ...)]
      WITH (key1=val1, key2=val2, ...)
      [ LIKE source_table [( <like_options> )] ]
       
    <column_definition>:
      column_name column_type [ <column_constraint> ] [COMMENT column_comment]
      
    <column_constraint>:
      [CONSTRAINT constraint_name] PRIMARY KEY NOT ENFORCED

    <table_constraint>:
      [CONSTRAINT constraint_name] PRIMARY KEY (column_name, ...) NOT ENFORCED

    <computed_column_definition>:
      column_name AS computed_column_expression [COMMENT column_comment]

    <watermark_definition>:
      WATERMARK FOR rowtime_column_name AS watermark_strategy_expression
      
    <like_options>:
    {
       { INCLUDING | EXCLUDING } { ALL | CONSTRAINTS | PARTITIONS }
     | { INCLUDING | EXCLUDING | OVERWRITING } { GENERATED | OPTIONS | WATERMARKS } 
    }[, ...]

    ```
-   Table 也可以是虚表即`view`视图，可由对现存 Table 操作 Table Api 和 SQL 查询来创建。

    ```java
    CREATE [TEMPORARY] VIEW [IF NOT EXISTS] [catalog_name.][db_name.]view_name
      [{columnName [, columnName ]* }] [COMMENT view_comment]
      AS query_expression

    ```

## 3.8 临时表和永久表

-   临时表的生命周期绑定到 Flink Session  
    临时表存于内存，对其他 Session 不可见，也没有绑定 catalog 或数据库，但可以被创建到 namespace 中，但即使删除对应的 namespace database 也不会 drop 临时表！
-   永久表一直存在直到被 Drop，跨 Flink Session 和集群可见  
    永久表需要[catalog](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/catalogs.html)(如 HiveCatalog) 来存放表的元数据，对连接到该 catalog 的 FlinkSession 可见。
-   Shadowing  
    如果注册和永久表同`identifier`的临时表，则临时表会掩盖永久表，此时永久表不可访问，所有对该标识符的表的访问都会路由到临时表。

    这个特性用来进行试验，一旦测试通过可以 drop 该临时表。

## 3.9 insert

参考

-   [INSERT Statement](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/sql/insert.html)

## 3.10 Explain

参考

-   [EXPLAIN Statements](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/sql/explain.html)
-   [Explaining a Table](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/common.html#explaining-a-table)
-   sql 方式

```sql
EXPLAIN PLAN FOR 
	SELECT count, word FROM MyTable1 WHERE word LIKE 'F%' 
	UNION ALL 
	SELECT count, word FROM MyTable2;

```

还可以通过`TableEnvironment.explainSql()` 和 `TableEnvironment.executeSql()`执行

-   Table.explain()  
    得到表的查询计划

    ```java
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    val tEnv = StreamTableEnvironment.create(env)

    val table1 = env.fromElements((1, "hello")).toTable(tEnv, $"count", $"word")
    val table2 = env.fromElements((1, "hello")).toTable(tEnv, $"count", $"word")
    val table = table1
      .where($"word".like("F%"))
      .unionAll(table2)
    println(table.explain())

    ```
-   StatementSet.explain()  
    得到多个 sink 的查询计划

    ```java
    val settings = EnvironmentSettings.newInstance.useBlinkPlanner.inStreamingMode.build
    val tEnv = TableEnvironment.create(settings)

    val schema = new Schema()
        .field("count", DataTypes.INT())
        .field("word", DataTypes.STRING())

    tEnv.connect(new FileSystem("/source/path1"))
        .withFormat(new Csv().deriveSchema())
        .withSchema(schema)
        .createTemporaryTable("MySource1")
    tEnv.connect(new FileSystem("/source/path2"))
        .withFormat(new Csv().deriveSchema())
        .withSchema(schema)
        .createTemporaryTable("MySource2")
    tEnv.connect(new FileSystem("/sink/path1"))
        .withFormat(new Csv().deriveSchema())
        .withSchema(schema)
        .createTemporaryTable("MySink1")
    tEnv.connect(new FileSystem("/sink/path2"))
        .withFormat(new Csv().deriveSchema())
        .withSchema(schema)
        .createTemporaryTable("MySink2")
        
    val stmtSet = tEnv.createStatementSet()

    val table1 = tEnv.from("MySource1").where($"word".like("F%"))
    stmtSet.addInsert("MySink1", table1)

    val table2 = table1.unionAll(tEnv.from("MySource2"))
    stmtSet.addInsert("MySink2", table2)

    val explanation = stmtSet.explain()
    println(explanation)

    ```

返回结果：

-   查询转换后得到的抽象语法树 AST，他是尚未优化的逻辑查询计划
-   优化后的逻辑查询计划
-   物理查询计划

## 3.11 TableSink

TableSink 指输出表：

-   可支持各种文件格式，如 CSV, Apache Parquet, Apache Avro
-   可支持各种存储系统，如 JDBC、Apache HBase、Apache Cassandra、Elasticsearch
-   可支持消息队列 MQ，如 Apache Kafka、RabbitMQ

## 3.12 Blink 对查询的优化

Flink 利用并扩展 Calcite 来执行查询优化，有基于规则和基于代价的优化：

-   子查询去相关性
-   投影裁剪  
    投影解释：投影运算也是一个单目运算，它是从一个关系 R 中选取所需要的列组成一个新关系，就是`select a,b,c`这样的语句，让结果集仅包含指定列，这种操作称为投影查询。
-   分区裁剪
-   过滤器下推  
    否则会收集大量数据后再过滤
-   子计划去重以避免重复计算
-   特殊子查询重写，包括两部分：
    -   将`IN`和`EXISTS`转为`left semi-joins`
    -   将`NOT IN`和 `NOT EXISTS` 转换为 `left anti-join`
-   可选的 join 重排序，通过`table.optimizer.join-reorder-enabled`开启

注意： 当前仅在子查询重写的结合条件下支持 IN / EXISTS / NOT IN / NOT EXISTS。

Flink 优化器不仅基于查询计划，还基于从数据源获得的统计信息以及每个算子的细粒度成本（如 io，cpu，网络和内存）来做最佳决策。

高级用户还可以通过`CalciteConfig`提供自定义优化，可以通过调用 `TableEnvironment#getConfig#setPlannerConfig` 将该对象传递给 TableEnvironment。

## 3.13 [SQL Hints](https://ci.apache.org/projects/flink/flink-docs-master/en/dev/table/sql/hints.html)

### 3.13.1 概述

Flink 1.11 加入的新特性，可用来改变 SQL 的查询计划。

-   指定 planner  
    Blink 和默认的 Planner 都有各自最佳场景，所以可指定 Planner
-   追加元数据和统计数据  
    如`table index for scan`和`skew info of some shuffle keys`是动态变化的
-   算子资源限制  
    多数情况给与算子默认资源配置（最小并行度或 managed memory 或指定的资源需求（GPC/SSD）等）。可通过 Hints 在 Query 级别控制资源。

### 3.13.2 Dynamic Table Options

可在每次 Query 时动态指定或覆写表的 option。需要加上配置`set table.dynamic-table-options.enabled=true`来开启，默认关闭。

否则在查询该表时使用 hints 会报错：

```
Fail to run sql command: select * from xxx /*+ OPTIONS('scan.startup.mode'='earliest-offset') */
org.apache.flink.table.api.ValidationException: OPTIONS hint is allowed only when table.dynamic-table-options.enabled is set to true

```

### 3.13.3 hints 语法

```sql
table_path 

```

-   key:  
    字符串类型的配置名
-   val:  
    字符串类型的配置值

### 3.13.4 例子

```sql
CREATE TABLE kafka_table1 (id BIGINT, name STRING, age INT) WITH (...);
CREATE TABLE kafka_table2 (id BIGINT, name STRING, age INT) WITH (...);


select id, name from kafka_table1 ;


select * from
    kafka_table1  t1
    join
    kafka_table2  t2
    on t1.id = t2.id;


insert into kafka_table1  select * from kafka_table2;

```

## 3.14 时态表 Temporal Table

### 3.14.1 概述

注意和临时表（Temporary Table）区分！完全是不同概念。

时态表表示一个参数化的视图 View，运行在一个变动中的 Table 上，返回指定时间点的该表的内容。

变动 Table 包括两类：

-   数据库表的 changelog 构成的 changelog 历史表  
    Flink 可追踪变动，允许查询特定时刻表的内容。在 Flink 里，这种表由`Temporal Table Function`表示。
-   物化变动构成的维度表  
    允许查询特定 ProcessingTime 时的表内容。在 Flink 里，这种表由`Temporal Table`表示。

### 3.14.2 目标

#### 3.14.2.1 关联 changing history table

Flink 可追踪变动，允许查询特定时刻表的内容。在 Flink 里，这种表由`Temporal Table Function`表示。

可将`append-only`表的行解释为表的 ChangeLog，返回指定时间点的该表的内容。将 append-only 的表解释为 ChangeLog 需要指定主键以及时间戳属性：

-   主键用来确定哪些行将被覆盖
-   时间戳用来确定行有效的时间

时态表主要目标是简化此类查询，加速执行，减少 State 用量。

比如有下表

```java
SELECT * FROM RatesHistory;

rowtime currency   rate
======= ======== ======
09:00   US Dollar   102
09:00   Euro        114
09:00   Yen           1
10:45   Euro        116
11:15   Euro        119
11:49   Pounds      108

```

`RatesHistory`表表示其他币种兑换日元`Yen`的汇率，`append-only`。

如果要输出所有`10:58`时刻的汇率，SQL 如下：

```sql
SELECT *
FROM RatesHistory AS r
WHERE r.rowtime = (
  SELECT MAX(rowtime)
  FROM RatesHistory AS r2
  WHERE r2.currency = r.currency
  AND r2.rowtime <= TIME '10:58');

```

结果：

```sql
rowtime currency   rate
======= ======== ======
09:00   US Dollar   102
09:00   Yen           1
10:45   Euro        116

```

可以看到确实是`10:58`时刻的汇率，也就是该表的快照。

该例子中，`currency`即为主键，`rowtime`为时间戳属性列。

该表在 Flink 由`Temporal Table Function`表示。

#### 3.14.2.2 关联 changing dimension table

允许查询特定 ProcessingTime 时的表内容。在 Flink 里，这种表由`Temporal Table`表示。

现有表`LatestRates`，内容是最新的汇率，也就是说他是物化的`RatesHistory`历史的物化终态表。

如果我们`10:58`（处理时间）的时候查询该表：

```sql
10:58> SELECT * FROM LatestRates;
currency   rate
======== ======
US Dollar   102
Yen           1
Euro        116

```

如果我们`12:00`（处理时间）的时候查询该表：

```sql
12:00> SELECT * FROM LatestRates;
currency   rate
======== ======
US Dollar   102
Yen           1
Euro        119
Pounds      108

```

### 3.14.3 Temporal Table Function

使用时态表，要访问数据时，必须传递一个[时间属性](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/functions/udfs.html#table-functions)，该属性用来确定将要返回的表的版本。

定义后，Temporal Table Function 可用一个时间参数`timeAttribute`产生一些列行，该集合包含相对于给定时间属性的所有现有主键的行的最新版本。

比如在`RatesHistory`表上定义了一个 Temporal Table Function：`Rates(timeAttribute)`，可返回给定时间属性列值时的 Rates 状态，则查询如下：

```sql
SELECT * FROM Rates('10:15');

rowtime currency   rate
======= ======== ======
09:00   US Dollar   102
09:00   Euro        114
09:00   Yen           1

SELECT * FROM Rates('11:00');

rowtime currency   rate
======= ======== ======
09:00   US Dollar   102
10:45   Euro        116
09:00   Yen           1

```

**注意，其实目前 Flink 不支持直接传入常量到 Temporal Table Function，目前只能应用在 join 中，上例只是为了让我们更好理解而已！**

对 append-only table 定义 Temporal Table Function 示例代码：

```java

val env = StreamExecutionEnvironment.getExecutionEnvironment
val tEnv = StreamTableEnvironment.create(env)


val ratesHistoryData = new mutable.MutableList[(String, Long)]
ratesHistoryData.+=(("US Dollar", 102L))
ratesHistoryData.+=(("Euro", 114L))
ratesHistoryData.+=(("Yen", 1L))
ratesHistoryData.+=(("Euro", 116L))
ratesHistoryData.+=(("Euro", 119L))



val ratesHistory = env
  .fromCollection(ratesHistoryData)
  .toTable(tEnv, 'r_currency, 'r_rate, 'r_proctime.proctime)

tEnv.createTemporaryView("RatesHistory", ratesHistory)




val rates = ratesHistory.createTemporalTableFunction($"r_proctime", $"r_currency") 

tEnv.registerFunction("Rates", rates)

```

### 3.14.4 Temporal Table

仅支持 Blink planner。

为了访问时态表，当前必须使用 `LookupableTableSource` 定义一个 TableSource。可使用`FOR SYSTEM_TIME AS OF`语法来查询时态表。

查询时态表 LatestRates：

```sql
SELECT * FROM LatestRates FOR SYSTEM_TIME AS OF TIME '10:15';

currency   rate
======== ======
US Dollar   102
Euro        114
Yen           1

SELECT * FROM LatestRates FOR SYSTEM_TIME AS OF TIME '11:00';

currency   rate
======== ======
US Dollar   102
Euro        116
Yen           1

```

结果和上面使用`Temporal Table Function`的示例相同。

**注意，其实目前 Flink 不支持直接传入常量来查询 Temporal Table，目前只能将时态表应用在 join 中，上例只是为了让我们更好理解而已！**

定义时态表：

```java

val env = StreamExecutionEnvironment.getExecutionEnvironment
val settings = EnvironmentSettings.newInstance().build()
val tEnv = StreamTableEnvironment.create(env, settings)




tEnv.executeSql(
    s"""
       |CREATE TABLE LatestRates (
       |    currency STRING,
       |    fam1 ROW<rate DOUBLE>
       |) WITH (
       |    'connector' = 'hbase-1.4',
       |    'table-name' = 'Rates',
       |    'zookeeper.quorum' = 'localhost:2181'
       |)
       |""".stripMargin)

```

LookupableTableSource 定义请参考[这里](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/sourceSinks.html#defining-a-tablesource-for-lookups)

## 3.15 Join

### 3.15.1 概述

动态表的 join 概念比普通静态表要难理解一些。

关于 FlinkSql 支持的 Join 可参照[这里](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/sql/queries.html#joins)

以下开始讲不同 Join 场景。

### 3.15.2 Regular Join

指两个流的 join。

这种场景下，两个表中的新数据、旧数据改动都会造成新的 join 计算。比如左表来了条新数据，则会和右表的旧的、未来的数据全部计算 join。

这个 join 语法适用于存在任意修改类型（insert, update, delete）的输入表。

但这样 Join 需要被 join 的两个表的输入永久保存在 Flink 的 State 里。因此，使用的资源量也会随着输入表的增长而无限增长。所以需要配置数据状态驻留期限。参阅[Query Configuration](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/streaming/query_configuration.html)

-   Inner Equi-join  
    目前只支持等值连接，即 join 条件至少包含一个`=`的维持，不支持任何 cross join 和 theta join。

    还要注意, Join 的顺序未优化，目前会按照 sql 中`from`后表定义的顺序依次执行 join，需要确保该顺序不会导致 cross join（笛卡儿积），否则会导致失败。

    最后需要注意的是需要配置 State 过期时间，否则会无限增长。参阅[Query Configuration](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/streaming/query_configuration.html)

    正例如下：

    ```sql
    SELECT * FROM Orders
    INNER JOIN Product
    ON Orders.productId = Product.id

    ```
-   Outer Equi-join  
    目前只支持等值连接，还要注意, Join 的顺序未优化，也需要配置 State 过期时间，同`Inner Equi-join`

    正例如下：

    ```sql
    SELECT *
    FROM Orders LEFT JOIN Product ON Orders.productId = Product.id

    SELECT *
    FROM Orders RIGHT JOIN Product ON Orders.productId = Product.id

    SELECT *
    FROM Orders FULL OUTER JOIN Product ON Orders.productId = Product.id

    ```

### 3.15.3 Interval Join(时间区间关联)

是`Regular Join`的子集，可以使用流的方式进行处理。

不同的是，`Interval Join`只支持带有时间属性的`append-only`表，不支持删改。

而且，由于时间属性是准单调递增的，因而 Flink 可以从 State 移除旧的值，而不会影响结果正确性。

执行时，Interval join 需要至少一个等值连接谓词和一个限制了两表的时间的 join 条件。如使用两个适当的 Range 谓词（&lt;, &lt;=,>=, >），一个 `BETWEEN` 谓词或单个比较两个输入表中相同事件类型 (process/event time) 的时间属性的等值谓词。

正例如下

```sql
ltime = rtime
ltime >= rtime AND ltime < rtime + INTERVAL '10' MINUTE
ltime BETWEEN rtime - INTERVAL '10' SECOND AND rtime + INTERVAL '5' SECOND

```

将所有收到订单后 4 小时内发货的订单表和他们对应的发货单进行 join 的例子：

```sql
SELECT *
FROM Orders o, Shipments s
WHERE o.id = s.orderId AND
      o.ordertime BETWEEN s.shiptime - INTERVAL '4' HOUR AND s.shiptime

```

### 3.15.4 时态表函数 Join

#### 3.15.4.1 概述

这是将一个`append-only table`作为左表和时态表作为右表进行 join。

以下例子展示了一个 join 的例子。

-   Orders  
    append-only table

```sql
SELECT * FROM Orders;

rowtime amount currency
======= ====== =========
10:15        2 Euro
10:30        1 US Dollar
10:32       50 Yen
10:52        3 Euro
11:04        5 US Dollar

```

-   RatesHistory  
    时态表，具体是个 append-only table 类型的各币种对日元 (Yen) 汇率变动。该表在 Flink 由`Temporal Table Function`表示。

```sql
SELECT * FROM RatesHistory;

rowtime currency   rate
======= ======== ======
09:00   US Dollar   102
09:00   Euro        114
09:00   Yen           1
10:45   Euro        116
11:15   Euro        119
11:49   Pounds      108

```

如果要将 Orders 表的金额转为日元，一般会这么做

```sql
SELECT
  SUM(o.amount * r.rate) AS amount
FROM Orders AS o,
  RatesHistory AS r
WHERE r.currency = o.currency
AND r.rowtime = (
  SELECT MAX(rowtime)
  FROM RatesHistory AS r2
  WHERE r2.currency = o.currency
  AND r2.rowtime <= o.rowtime);

```

而如果使用 `Temporal Table Function` ，注册一个在时态表`RatesHistory`上的函数`Rates`，可大大简化：

```sql
SELECT
  o.amount * r.rate AS amount
FROM
  Orders AS o,
  LATERAL TABLE (Rates(o.rowtime)) AS r
WHERE r.currency = o.currency

```

每条 probe 左表的数据会和 build 右表的对应时间属性版本的所有数据行做 jon。也就是说，上例中`Orders`表的每条记录会和右表`Rates`在对应的`o.rowtime`的版本的数据做 join。

右表需要主键以修改，这里是`currency`列。

如果使用 ProcessingTime 来查询，则新 append 的 order 总是会 join 最新版本的`Rates`数据。

可以发现，时态表 Join 和常规 Join 最大不同就是，右表（时态表）有新数据也不会影响之前的 join 结果。这可限制 Flink 在 State 中保存元素的数量。也就是说，随着时间推移，之前的不再需要的版本的数据会从 State 中移除。

#### 3.15.4.2 使用

注意，目前时态表 join 时的状态过期尚未实现（Fink1.11），可能导致查询使用的状态无限增长。

关于 ProcessingTime 和 EventTime 时的时态表 join 可参考[这里](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/streaming/joins.html#processing-time-temporal-joins)

-   ProcessingTime  
    此时不能传递过去的时间属性给 temporal table function，而只能总是当前时间。所以这种场景下每次调用 temporal table function 总是返回最新的已知版本的表数据。

    注意，此时仅 build 表的最新版本（相对于定义的主键）的数据会被保存在状态中。

    build 表的更新不会影响先前发出的 join 结果，因为只会让左表和 build 表最新版本数据 join。

    可以将这种 join 想象为`HashMap<K, V>`，存有所有 build 表的行。如果新数据有着和旧数据重复 key，此时直接覆盖该缓存即可。此时每个 probe 左表总是使用最新的 HashMap。
-   EventTime  
    此时可传递过去的时间给 temporal table function，进行 join。

    -   EventTime  
        此时可传递过去的时间给 temporal table function，进行 join。

    此场景下，会将自从上次水位依赖所有版本的行保存到 State 中。水位的作用是推进 join 算子进程，丢弃不需要的版本的 build table 数据。

    举个例子，比如拥有`12:30:00`的事件时间戳的行 append 到 probe 表，此时就会和时态表`12:30:00`版本的数据（即时间戳小于等于该时间的数据）进行 join，并会不断更新主键，并根据主键应用更新直到该时间点。

### 3.15.5 时态表 Join

#### 3.15.4.1 概述

这是将一个包含任意操作的表作为 probe 左表和时态表 (必须由[LookupableTableSource](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/sourceSinks.html#defining-a-tablesource-with-lookupable)支撑) 作为 build 右表进行 join。

比如我们有一个不断物化变化的当前汇率表`LatestRates`作为时态表：

```sql
10:15> SELECT * FROM LatestRates;

currency   rate
======== ======
US Dollar   102
Euro        114
Yen           1

10:30> SELECT * FROM LatestRates;

currency   rate
======== ======
US Dollar   102
Euro        114
Yen           1


10:52> SELECT * FROM LatestRates;

currency   rate
======== ======
US Dollar   102
Euro        116     <==== changed from 114 to 116
Yen           1

```

另有一个`append-only`表 Orders：

```sql
SELECT * FROM Orders;

amount currency
====== =========
     2 Euro             <== arrived at time 10:15
     1 US Dollar        <== arrived at time 10:30
     2 Euro             <== arrived at time 10:52

```

以下例子展示了普通标 Orders 和时态表 LatestRates join 以计算订单金额转换为当时汇率对应的日元的例子。

期待的结果如下：

```sql
amount currency     rate   amout*rate
====== ========= ======= ============
     2 Euro          114          228    <== arrived at time 10:15
     1 US Dollar     102          102    <== arrived at time 10:30
     2 Euro          116          232    <== arrived at time 10:52

```

joinSQL:

```sql
SELECT
  o.amout, o.currency, r.rate, o.amount * r.rate
FROM
  Orders AS o
  JOIN LatestRates FOR SYSTEM_TIME AS OF o.proctime AS r
  ON r.currency = o.currency

```

每条 probe 左表的数据会和 build 右表的当前版本的数据行做 join。也就是说，上例中`Orders`表的每条新记录会和右表`LatestRates`最新版本的数据做 join，因为这里使用的是`proctime`。

可以发现，时态表 Join 和常规 Join 最大不同就是，右表（时态表）有新数据也不会影响之前的 join 结果。同时，时态表 join 算子十分轻量级，不需要保存任何 State！

#### 3.15.5.2 使用

时态表 join 语法：

```sql
SELECT [column_list]
FROM table1 [AS <alias1>]
[LEFT] JOIN table2 FOR SYSTEM_TIME AS OF table1.proctime [AS <alias2>]
ON table1.column-name1 = table2.column-name1

```

-   目前仅支持`Blink planner`
-   目前仅支持 SQL API，不支持 Table API
-   目前仅支持 ProcessingTime 时态表 join，不支持 EventTime
-   目前仅支持`INNER JOIN`和`LEFT JOIN`
-   `FOR SYSTEM_TIME AS OF`必须跟在时态表之后
-   proctime 是 probe 表 table1 的处理时间属性，含义是 join 时，会获取时态表在处理时间的快照进行 join

实例：

```sql
SELECT
  SUM(o_amount * r_rate) AS amount
FROM
  Orders
  JOIN LatestRates FOR SYSTEM_TIME AS OF o_proctime
  ON r_currency = o_currency

```

### 3.15.6 小结

-   Temporal Tables 是跟随时间变化而变化的表。
-   Temporal Table Function 提供访问 Temporal Tables 在某一时间点的状态的能力
-   Join Temporal Table Function 的语法与 Join Table Function 一致
-   目前仅支持在 Temporal Tables 上的 inner join

temporal table function join 对比 temporal table join：

-   相同：
    -   目标相同
-   不同：
    -   SQL 语法不同  
        前者使用 join UDTF，后者使用标准的时态表语法 (SQL:2011)
    -   State 保存不同  
        前者将 join 的两个流保存在 State，后者仅接受输入流，并在外部数据库查找对应 key 的记录
    -   前者一般用来和 changelog stream 做 join，后者一般和外部表（维表）做 join

### 3.15.7 Join with Table Function (UDTF)

将表与另一个表的函数执行的结果进行 join 操作。左表（outer）中的每一行将会与右表的函数执行所产生的所有结果中相关联的行进行 join 。

当然，要先注册[UDTF](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/functions/udfs.html)。

-   Inner Join  
    此时如果函数调用结果为空，则对应的左表行会被丢弃

```sql
SELECT users, tag
FROM Orders, LATERAL TABLE(unnest_udtf(tags)) t AS tag

```

-   Left Outer Join  
    此时如果函数调用结果为空，则对应的左表行会被保留，使用`null`填充

```sql
SELECT users, tag
FROM Orders LEFT JOIN LATERAL TABLE(unnest_udtf(tags)) t AS tag ON TRUE

```

## 4.1 概述

可参考:

-   [Connect to External Systems](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/connectors/)

~DDL 不行，我们可以用[Connect to External Systems](https://ci.apache.org/projects/flink/flink-docs-release-1.9/dev/table/connect.html#json-format)，直接读写外部数据源流批数据：~

用来定义外部数据源连接。不是所有都支持流 / 批，支持批的 Connector 支持的 Update Mode 也不尽相同。

支持的输出：

注意，从 1.11.0 开始相关 API 有巨大变化，要用老的请查看[legacy documentation](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/connect.html)

-   Table Schema  
    定义表的 schema，描述了怎么将 Table Source 的数据格式映射到 Table API 的 schema，以及 Table 映射到 Sink 的方式。可暴露给 SQL 查询。
-   支持 Time 属性  
    可以使用一个或多个字段来提取或插入时间属性到 Table Schema。

Flink 连接外部系统可通过以下两种方式指定：

-   使用 Table & SQL API，搭配`org.apache.flink.table.descriptors`下的内容
-   通过 SQL 客户端的 YAML 配置文件声明

一个 Table & SQL API 中连接外部数据源语句基本结构：

```java
tableEnvironment

  .connect(...)
  
  .withFormat(...)
  
  .withSchema(...)
  
  .inAppendMode()
  
  .registerTableSource("MyTable")
  
  .registerTableSink
  
  .registerTableSourceAndSink

```

一个从 Kafka 中读 Avro 格式存储的数据的例子：

```java
tableEnvironment

.connect(
  new Kafka()
    .version("0.10")
    .topic("test-input")
    .startFromEarliest()
    .property("zookeeper.connect", "localhost:2181")
    .property("bootstrap.servers", "localhost:9092")
)


.withFormat(
  new Avro()
    .avroSchema(
      "{" +
      "  \"namespace\": \"org.myorganization\"," +
      "  \"type\": \"record\"," +
      "  \"name\": \"UserMessage\"," +
      "    \"fields\": [" +
      "      {\"name\": \"timestamp\", \"type\": \"string\"}," +
      "      {\"name\": \"user\", \"type\": \"long\"}," +
      "      {\"name\": \"message\", \"type\": [\"string\", \"null\"]}" +
      "    ]" +
      "}"
    )
)


.withSchema(
  new Schema()
    .field("rowtime", Types.SQL_TIMESTAMP)
      .rowtime(new Rowtime()
        .timestampsFromField("timestamp")
        .watermarksPeriodicBounded(60000)
      )
    .field("user", Types.LONG)
    .field("message", Types.STRING)
)


.inAppendMode()


.registerTableSource("MyUserSourceTable");

.registerTableSink("MyUserSinkTable");

```

配置的连接属性会被转换为标准化的、基于 String 的 key-value 键值对。会基于 Java SPI 机制搜索唯一匹配的 Table Factory 来创建 Table Source、Table Sink 以及相应的 format。

## 4.2 示例

我们需要使用 FlinkSql 定义表名、表的 Schema 以及表的属性，以便连接外部系统。

以下注册 Kafka 数据源 + 读取 json 数据的例子：

```sql
CREATE TABLE MyUserTable (
  
  `user` BIGINT,
  message STRING,
  ts TIMESTAMP,
  proctime AS PROCTIME(), 
  WATERMARK FOR ts AS ts - INTERVAL '5' SECOND  
) WITH (
  
  'connector' = 'kafka',
  'topic' = 'topic_name',
  'scan.startup.mode' = 'earliest-offset',
  'properties.bootstrap.servers' = 'localhost:9092',
  'format' = 'json'   
)

```

由[dynamic-table-factories](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/sourceSinks.html#dynamic-table-factories)（基于[java SPI](https://docs.oracle.com/javase/tutorial/sound/SPI-intro.html)查找）来根据 sql 创建 TableSource、TableSink 以及对应的`format`，。

## 4.3 主键

1.11.0 以前靠的是`group by`后的关键字进行推断，1.11.0 以后可以指定。

主键约束表名一个或若干列是一个表内的唯一一行，且不包含 null 值。

在 SinkTable 内使用主键，一般被用来做`upsert`操作。

由用户来确保查询强制执行键完整性。

```sql
CREATE TABLE MyTable (
  MyField1 INT,
  MyField2 STRING,
  MyField3 BOOLEAN,
  PRIMARY KEY (MyField1, MyField2) NOT ENFORCED  
) WITH (
  ...
)

```

## 4.4 时间属性

### 4.4.1 Processing time

即执行相应算子所在机器系统时间 (`wall-clock time`墙上时间)，最简单的时间属性，但有不确定性。

该属性不需要时间戳提取器（timestamp extract）和水位生成器。

-   DDL\`\`\`sql
    CREATE TABLE user_actions (
      user_name STRING,
      data STRING,
      user_action_time AS PROCTIME()) WITH (
      ...
    );

    SELECT TUMBLE_START(user_action_time, INTERVAL '10' MINUTE), COUNT(DISTINCT user_name)
    FROM user_actions
    GROUP BY TUMBLE(user_action_time, INTERVAL '10' MINUTE);

    ```

    ```
-   DataStream-to-Table 转换时定义  
    必须在 schema 最后声明 processingTime 属性列 \`\`\`java
    val stream: DataStream[(String, String)] = ...

    val table = tEnv.fromDataStream(stream, $"UserActionTimestamp", $"user_name", $"data", $"user_action_time".proctime)

    val windowedTable = table.window(Tumble over 10.minutes on $"user_action_time" as "userActionWindow")

    ```

    ```
-   TableSource

```java


class UserActionSource extends StreamTableSource[Row] with DefinedProctimeAttribute {
	
	override def getReturnType = {
		val names = Array[String]("user_name" , "data")
		val types = Array[TypeInformation[_]](Types.STRING, Types.STRING)
		Types.ROW(names, types)
	}

	override def getDataStream(execEnv: StreamExecutionEnvironment): DataStream[Row] = {
		
		val stream = ...
		stream
	}
	
	override def getProctimeAttribute = {
		
		"user_action_time"
	}
}


tEnv.registerTableSource("user_actions", new UserActionSource)

val windowedTable = tEnv
	.from("user_actions")
	.window(Tumble over 10.minutes on $"user_action_time" as "userActionWindow")

```

### 4.4.2 Event time

指用户附加给每行数据的时间戳，Flink 基于该时间戳作为时间基准进行处理。

特点是可以处理一定范围内的乱序事件和迟到时间（一致性保证），可保证从同一个存储层内反复读取同样一批数据时结果是可重放的。

将 Event time 设为时间属性时，Flink 需要从数据记录中提取时间戳以及生成水位，以便处理乱序和迟到事件。

-   DDL

    ```sql
    CREATE TABLE user_actions (
      user_name STRING,
      data STRING,
      user_action_time TIMESTAMP(3),
      
      WATERMARK FOR user_action_time AS user_action_time - INTERVAL '5' SECOND
    ) WITH (
      ...
    );

    SELECT TUMBLE_START(user_action_time, INTERVAL '10' MINUTE), COUNT(DISTINCT user_name)
    FROM user_actions
    GROUP BY TUMBLE(user_action_time, INTERVAL '10' MINUTE);

    ```
-   DataStream-to-Table 转换时定义  
    待转换的 DataStream 必须已经定义时间戳和水位。

    有两种方法定义 DataStream 中的 EventTime 时间戳：

    -   x.rowtime 时 x 字段不存在  
        在 schema 的结尾追加一个新的字段
    -   x.rowtime 时 x 字段存在  
        替换已经存在的 x 字段

    ```java



    val stream: DataStream[(String, String)] = inputStream.assignTimestampsAndWatermarks(...)


    val table = tEnv.fromDataStream(stream, $"user_name", $"data", $"user_action_time".rowtime)





    val stream: DataStream[(Long, String, String)] = inputStream.assignTimestampsAndWatermarks(...)



    val table = tEnv.fromDataStream(stream, $"user_action_time".rowtime, $"user_name", $"data")


    val windowedTable = table.window(Tumble over 10.minutes on $"user_action_time" as "userActionWindow")

    ```
-   TableSource

```java


class UserActionSource extends StreamTableSource[Row] with DefinedRowtimeAttributes {

	override def getReturnType = {
		val names = Array[String]("user_name" , "data", "user_action_time")
		val types = Array[TypeInformation[_]](Types.STRING, Types.STRING, Types.LONG)
		Types.ROW(names, types)
	}
	
	
	
	
	
	override def getDataStream(execEnv: StreamExecutionEnvironment): DataStream[Row] = {
		
		
		
		val stream = inputStream.assignTimestampsAndWatermarks(...)
		stream
	}
	
	
	override def getRowtimeAttributeDescriptors: util.List[RowtimeAttributeDescriptor] = {
		
		
		
		val rowtimeAttrDescr = new RowtimeAttributeDescriptor(
			"user_action_time",
			new ExistingField("user_action_time"),
			new AscendingTimestamps)
		val listRowtimeAttrDescr = Collections.singletonList(rowtimeAttrDescr)
		listRowtimeAttrDescr
	}
}


tEnv.registerTableSource("user_actions", new UserActionSource)

val windowedTable = tEnv
	.from("user_actions")
	.window(Tumble over 10.minutes on $"user_action_time" as "userActionWindow")

```

### 4.4.3 Ingestion time

数据到达 Flink Source 的时间，内部处理方式类似 Event time

注意：只要时间属性没有被修改过，而是仅仅被传递，那就一直保持为合法时间属性，可以和普通时间戳一样被访问计算，但只要被计算就会被物化、称为常规时间戳，就不能再作为 Flink 时间属性了，也不能在水位系统中起作用了，更不能作用到时间相关计算！

### 4.4.4 时间属性使用

-   时间属性种类指定

```java
val env = StreamExecutionEnvironment.getExecutionEnvironment

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime) 





```

-   时间属性字段指定  
    如 Window 等基于时间的算子，就需要使用时间属性，所以应该在建表时指定时间属性字段。指定时机如下：

    -   DDL `CREATE TABLE`
    -   DataStream
    -   TableSource
-   时间属性字段传递  
    时间属性可被传递，只要没有修改，就可以在一个表简单传递到另一个表后继续使用。
-   时间属性字段物化  
    当时间属性字段被用在计算中时，就会被物化为普通时间戳，该时间戳不能配合 Flink 时间和水位使用了，也就无法用在基于时间的算子中

## 4.5 [Connector 原理](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/sourceSinks.html)

### 4.5.1 概述

`DynamicSource` 和 `DynamicSink`可被用来从外部系统读或写入数据到外部系统，他们也可以被称为`Connector`。

下图展示了在翻译过程中，从一个 stage 到下一个 stage 时，Object 如何被转为其他 Object：  
![](https://img-blog.csdnimg.cn/20200811162716673.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

### 4.5.2 Metadata

动态表的元数据（包括 DDL 和由 Catalog 提供的）由`CatalogTable`实例表示。

对应上图的 Catalog->CatalogTable。

### 4.5.3 Planning

#### 4.5.3.1 概述

利用 CatalogTable 来生成具体 Connector 相关的逻辑执行计划。

当开始为 Table 生成执行计划和优化，CatalogTable 被解析为`DynamicTableSource`（用在`SELECT`语句中读取）和`DynamicTableSink`（用在`INSERT INTO`语句中写入）。

解析过程中`DynamicTableSourceFactory`（生成`DynamicTableSource`具体实现类，如`JdbcDynamicTableSource`）和`DynamicTableSinkFactory`（生成`DynamicTableSink`具体实现类，如`JdbcDynamicTableSink`）接口的具体实现类就会根据指定的 Connector 逻辑来翻译 CatalogTable 元数据。Dynamic Factory 被用来根据 catalog 和 session 信息中为外部存储系统配置动态表 Connector。这些工厂类的目的是验证 DDL 选项（如图中的`'port' = '5022'`）、配置编解码格式等，最后创建参数化的 Table Connector 实例。

#### 4.5.3.2 SPI

这两个工厂类接口的实现类查找和初始化是通过 java SPI 机制实现，也就是说 DDL 中的`'connector' = 'custom'`必须能找到对应的工厂类。SPI 将检查唯一匹配的工厂类，该工厂类由工厂标识符和请求的父类名字（例如`DynamicTableSourceFactory`）唯一标识。

比如可以看看`flink-connector-jdbc`相关 SPI 定义文件:

-   META-INF/services/org.apache.flink.table.factories.Factory  
    org.apache.flink.connector.jdbc.table.JdbcDynamicTableFactory
-   META-INF/services/org.apache.flink.table.factories.TableFactory  
    org.apache.flink.connector.jdbc.catalog.factory.JdbcCatalogFactory  
    org.apache.flink.connector.jdbc.table.JdbcTableSourceSinkFactory

更多关于 SPI 内容可参考:

-   [Java SPI 机制在 Flink SQL 中的应用](https://mp.weixin.qq.com/s/dYi9_CSwKTanJjl6C4qEgg)
-   [Flink1.10 基于工厂模式的任务提交与 SPI 机制](https://mp.weixin.qq.com/s/HX-JGkkq56ZzWgibsDOzww)

如果需要，Catalog 也可以绕过这个工厂 SPI 发现机制，那就需要 `org.apache.flink.table.catalog.Catalog#getFactory`返回一个实现了所请求的父类的实现类的实例。

FlinkSql Planner 会使用 Source 和 Sink 实例来执行指定 Connector 相关的相互通信，直到发现最佳逻辑执行计划。

依赖于可选的接口（如 SupportsProjectionPushDown 或 SupportsOverwrite 等），planner 也许会将修改应用到生成的实例上，并修改运行时实现。

#### 4.5.3.3 DynamicTableSource

套路为`DynamicTableSourceFactory -> DynamicTableSource -> ScanRuntimeProvider`，工厂类负责翻译 DDL option，Source 负责创建运行时逻辑。

当读取动态表时，读到的数据可能被视为以下两个：

-   ScanTableSource  
    ChangeLog，Source 在 batch 场景发送有界的、insert-only 流；Streaming 场景发送无界的、insert-only 流；CDC(Change Data Capture) 场景发送无界 / 有界的、可包含增删改的记录的流。

    从外部数据中扫描整张表的所有行。可包含增删改的行，所以可被用来读取 changelog。

    `getChangelogMode`返回`ChangelogMode`，表示运行时可能返回的 Changelog 类型集合。

    如果 Source 有一些额外的能力，可以实现`org.apache.flink.table.connector.source.abilities`包内的接口，如`SupportsProjectionPushDown`投影下推。

    ScanTableSource 生产的记录必须是`org.apache.flink.table.data.RowData`，用来在 Flink 内部传递。
-   LookupTableSource  
    一个持续变化或非常大的外部表，这个表的内容一般不会被整个读取，而是在需要时单独读取。

    在运行时，通过一个或多个 Key 来从外部系统中查找数据行。不需要读取整张表，可以懒惰地获取单独的数据。

    注意，LookupTableSource 目前仅支持`insert-only`变更；不支持 abilities 包内的接口。

    LookupTableSource 的运行时实现是`TableFunction`或`AsyncTableFunction`。 在运行期间，将使用给定查找 key 的值来调用该函数。

Class 可以同时实现以上两个接口，由 Flink Planner 来根据具体的查询内容来决定使用的方法。

#### 4.5.3.4 DynamicTableSink

当写入动态表时，数据可被视为 ChangeLog。`getChangelogMode`方法返回`ChangelogMode`，表示运行时可能返回的 Changelog 类型集合。

-   Sink 在 batch 场景只能接收 insert-only 行，输出有界流；
-   Sink 在 Streaming 场景只能接收 insert-only 行，输出无界流；
-   Sink 在 CDC(Change Data Capture) 场景可接收增删改的行，输出无界 / 有界流。

可实现额外功能，详见`org.apache.flink.table.connector.sink.abilities`包，如`SupportsOverwrite`。

与`DynamicTableSource`输出`org.apache.flink.table.data.RowData`对应的，DynamicTableSink 必须消费该类结构的数据。

#### 4.5.3.5 Encoding / Decoding Format

一些 Table Connector 接收不同的 format 以编解码 key 和 value。Format 实现层次类似`DynamicTableSourceFactory -> DynamicTableSource -> ScanRuntimeProvider`，发现机制也是用的 SPI。

### 4.5.4 Runtime

逻辑执行计划 -> 物理执行计划。

一旦逻辑执行计划生成，Planner 会从具体的 TableConnector 中生成运行时实现逻辑，这些运行逻辑是 Flink Connector 的接口如`InputFormat`、`SourceFunction`等的实现类实现的。

这些接口按其他层级的抽象，分组为`ScanRuntimeProvider`、 `LookupRuntimeProvider`、 `SinkRuntimeProvider`的子类。

比如`OutputFormatProvider`(提供`org.apache.flink.api.common.io.OutputFormat`) 和`SinkFunctionProvider` (提供 `org.apache.flink.streaming.api.functions.sink.SinkFunction`) 都是`SinkRuntimeProvider`的子类。

## 4.6 Table Schema

### 4.6.1 概述

Table Schema 定义表的每个列的名字和类型，类似于 SQL `create table`语句那样，用来暴露给 SQL 查询。此外，还可以指定如何将列与表数据编码 schema 的字段进行映射。当输入列无序时，Tabel Schema 可清晰地定义列名、顺序和来源。Table Schema 会和 Table Format 匹配来在 Table 数据输入和输出的过程中完成 Schema 转换。

此外， Table Schema 还可指定 Time 属性提取器。

### 4.6.2 例子

简单例子：

```java
.withSchema(
  new Schema()
  	
    .field("MyField1", Types.INT)
    .field("MyField2", Types.STRING)
    .field("MyField3", Types.BOOLEAN)
)

```

复杂例子：

```java
.withSchema(
  new Schema()
    .field("MyField1", Types.SQL_TIMESTAMP)
    	
      .proctime()     
    .field("MyField2", Types.SQL_TIMESTAMP)
    	
      .rowtime(...)   
    .field("MyField3", Types.BOOLEAN)
    	
      .from("mf3")     

```

### 4.6.3 Rowtime

上述的`.rowtime(...)`本小节详细说下。

rowtime 在 flink 里用来处理事件时间 event-time。

采用 Rowtime 时，总是需要设置 timestamp 提取策略和 watermark 策略。

timestamp 提取为 rowtime 例子如下：

```java
.rowtime(
  new Rowtime()
  	
    .timestampsFromField("ts_field")  
)

.rowtime(
  new Rowtime()
  	
    .timestampsFromSource()
)

.rowtime(
  new Rowtime()
  	
  	
    .timestampsFromExtractor(...)
)

```

水位策略例子：

```java
.rowtime(
  new Rowtime()
  	
  	
  	
    .watermarksPeriodicAscending()
)

.rowtime(
  new Rowtime()
  	
  	
    .watermarksPeriodicBounded(2000)
)


.rowtime(
  new Rowtime()
  	
    .watermarksFromSource()
)

```

## 4.7 Table Formats

### 4.7.1 概述

一些外部数据系统支持不同的 Table Formats，比如 kafka 或文件就支持其内存储的表的行使用 CSV、JSON、Avro 进行编码，所以需要指定 Table Format 来阐明外部数据源解析方式。

### 4.7.2 JSON Table Format

JSON 格式允许读取和写入与给定的`format schema`相对应的 JSON 数据。format schema 可用 Flink type（SQl-like，映射到对应的 SQL 数据类型）、 JSON schema（适合复杂的嵌套数据结构）或目标表的 schema（适合 format schema 等于 table schema 的场景，可自动派生出 schema）来定义。

目前支持的 JSON schema 类型和 Flink SQL 类型如下：  
![](https://img-blog.csdnimg.cn/20191023153831677.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

Missing Field Handling: By default, a missing JSON field is set to null. You can enable strict JSON parsing that will cancel the source (and query) if a field is missing.

Make sure to add the JSON format as a dependency.

需要在项目中添加 JSON 依赖：

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-json</artifactId>
    <version>${flink.version}</version>
</dependency>

```

实例：

```java
.withFormat(
  new Json()
  	
    .failOnMissingField(true) 

    
    
    .schema(Type.ROW(...))

    
    .jsonSchema(
      "{" +
      "  type: 'object'," +
      "  properties: {" +
      "    lon: {" +
      "      type: 'number'" +
      "    }," +
      "    rideTime: {" +
      "      type: 'string'," +
      		
      "      format: 'date-time'" +
      "    }" +
      "  }" +
      "}"
    )

    
    
    
    .deriveSchema()
)

```

## 4.8 Update Modes

### 4.8.1 概述

流式查询中，需要声明怎么执行动态表和外部 Connector 之间的数据交换，有以下模式：

-   Append Mode（增）  
    仅交互`INSERT`操作数据
-   Retract Mode （增删改）  
    交互`INSERT`（编码为`ADD`）和`DELETE`（编码为`RETRACT`）、`UPDATE`（对于修改前的行来说编码为`RETRACT`，对于修改后的新行来说编码为`ADD`）操作数据。

    与 Upsert Mode 相反，Retract Mode 不能定义 key。

    每个 UPDATE 操作由两条消息（RETRACT 和 ADD）组成，效率较低。
-   Upsert Mode（增删改）  
    交互`UPSERT`（可编码`INSERT`和`UPDATE`）和`DELETE`操作数据。

    该模式需要一个唯一的 key（可能是组合的），用来传播 update 事件。具体来说，外部连接器需要了解该唯一 key 属性，才能正确处理消息。

    Upsert Mode 和 Retract Mode 都支持增删改，但不同是，Upsert Mode 中的 UPDATE 事件使用单条 UPSERT 消息进行编码，而 Retract Mode 中的 UPDATE 由两条消息（RETRACT 和 ADD）组成，因此本模式效率更高。

### 4.8.2 例子

```java
.connect(...)
  .inAppendMode()    

```

每个 connector 支持哪些 update mode，请参阅具体 connector 文档。

## 4.9 更多例子

-   [Flink 官方 - table_api](https://ci.apache.org/projects/flink/flink-docs-master/getting-started/walkthroughs/table_api.html)  
    展示了如何实现一个对 batch source 的简单的 Table API 查询，以及如何拓展为 streaming source 的持续查询。
-   [Flink 学习 5 - 使用 rowtime 且分窗，Connector 读取 Kafka 写入 MySQL 例子](https://blog.csdn.net/baichoufei90/article/details/102747748)
-   [Apache Flink 各类关键数据格式读取 / SQL 支持](https://blog.csdn.net/rongyongfeikai2/article/details/83656715)
-   [如何在 Flink 1.9 中使用 Hive？](https://segmentfault.com/a/1190000020306773?utm_source=tag-newest)
-   [Flink SQL 解析复杂（嵌套）JSON](https://www.cnblogs.com/Springmoon-venn/p/12664547.html)

## 5.1 [File System Connector](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/connectors/filesystem.html)

请点击[File System Connector](https://blog.csdn.net/baichoufei90/article/details/107832306#filesystem)

## 5.2 [Kafka Connector](https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/table/connectors/kafka.html)

### 5.2.1 概述

![](https://img-blog.csdnimg.cn/20200729154957942.png)

Kafka Connector 使得 Flink 可从 Kafka 中消费、写入数据。

-   依赖  
    Kafka 0.11 以上版本后可采用统一的[flink-sql-connector-kafka_2.11-1.11.0.jar](https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-connector-kafka_2.11/1.11.0/flink-sql-connector-kafka_2.11-1.11.0.jar)
-   关于 Flink 分区和 Kafka 分区关系  
    可通过 connector 参数`sink.partitioner`指定：

    -   fixed  
        默认情况下，KafkaSink 最多可以写入与其自身并行性（parallelism）一样多的 Kafka 分区，即每个并行的 KafkaSink 实例都固定写入一个 Kafka 分区。 如果实例数大于 Kafka 分区数，则空闲。

    即 Flink 生产者发送消息到 Kafka 时默认用的 FlinkFixedPartitioner，如下图  
    ![](https://img-blog.csdnimg.cn/20191208161922368.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

    这种方法有个明显的问题，即 partition 数大于 task 数时，多出来的 partition 不会有数据写入。partition 扩容时，也需要重启 Flinik 程序。

    -   round-robin  
        循环分区器对于避免不平衡分区很有用， 但是，这将导致所有 Flink 实例与所有 Kafka Broker 节点之间的大量网络连接。
    -   自定义  
        为了将写操作分配到更多分区或自定义每行数据到分区的路由，可以提供自定义接收器分区程序，继承`FlinkKafkaPartitioner`。
-   一致性保证  
    默认情况下，如果在启用检查点的时执行 Flink，则 KafkaSink 会将具有至少一次 (at least once) 保证的数据提取到 Kafka 中。

    当使用 Checkpoint 时，提供 exactly-once 精准一次语义，具体原理是两阶段提交。

    除了使用 Checkpoint，还可以通过设置参数`sink.semantic`来调节语义：

    -   at-least-once  
        默认值。保证至少一次，可能重复
    -   none  
        没有任何保障，可能多或少
    -   exactly-once  
        使用 Kafka 生产者事务来保证。注意此时必须同时设定 Kafka Consumer 事务隔离等级`isolation.level`为`read_committed`才会 读不到未提交的事务数据
-   Kafka 0.10 + 的 Timestamp 属性  
    Kafka0.10 开始，数据就带了一个`timestamp`作为元数据的一部分，该字段含义是数据写入 Kafka 的时间。该字段可用作 Flink rowtime，请参考 Java/Scala 的`timestampsFromSource`方法。
-   Kafka 0.11 + 版本  
    因为 Flink1.7 开始，Kafka Connector 的定义就应该是独立于硬编码的 Kafka version 了，所以使用`.version("universal")`作为 Kafka0.11 开始的所有版本 Kafka 的通配符。
-   Commit offset 策略

    -   不使用 Checkpoint 时，由`enable.auto.commit`, `auto.commit.interval.ms`一起决定 commit 行为，每隔一段时间向 kafka commit 一次 offset。
    -   使用 Checkpoint 时，kafka consumer 将 offset commit 到 checkpoint state 中。
-   Consumer 分配策略  
    ![](https://img-blog.csdnimg.cn/20191208160436987.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

    如上图，不同 topic 之间的 startIndex 是随机的，故解决了多 topic 负载均衡问题。

    **注意，Consumer 无法感应获取 Kafka topic 新增的 Partition！也就是说，某 topic 扩容 Partition 后必须重启 Flink 程序。** 
-   Changelog Source  
    Flink 支持将 Kafka 作为`Changelog Source`。

    只要使用 CDC 工具从其他数据库捕获变动 event 放入 Kafka，则可以使用`cdc format`来解释 kafka 中数据，转为`INSERT/UPDATE/DELETE`来放入 Flink SQL 系统中。目前支持[debezium-json](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/connectors/formats/debezium.html) 和 [canal-json](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/connectors/formats/canal.html)。
-   数据类型映射  
    Kafka 按[format](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/connectors/formats/)将数据序列化为字节，没有数据类型和 schema。反序列化也是根据`format`.
-   元数据  
    可直接通过 SQL 获取 Kafka 元数据，应用到定义的列中  
    ![](https://img-blog.csdnimg.cn/20201214223548354.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
    以上 R 表示可读，W 表示可写。只读的字段必须在定义时加上`VIRTUAL`，以在`INSERT INTO`时排除这些字段。

### 5.2.2 CDC - Changelog Source

如果 Kafka 数据来源于使用 CDC 工具从其他数据库捕获的，则你可以使用`CDC format`来将数据解析为 Flink SQL 中的`INSERT/UPDATE/DELETE`，然后继续处理，如同步数据到其他目的地等。

目前支持 CDC 格式如下：

-   [debezium-json](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/connectors/formats/debezium.html)
-   [canal-json](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/connectors/formats/canal.html)

相关文档：

-   [阿里伍翀讲解 CDC](https://www.bilibili.com/video/BV1zt4y1D7kt/)
-   [Flink SQL CDC 上线！我们总结了 13 条生产实践经验](https://mp.weixin.qq.com/s/Mfn-fFegb5wzI8BIHhNGvQ)

### 5.2.3 配置

| 配置项                          | Required           | Default       | Type   | 描述                                                                                                                                                                                                       |
| ---------------------------- | ------------------ | ------------- | ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| connector                    | required           | (none)        | String | ‘kafka’, ‘kafka-0.11’, ‘kafka-0.10’.                                                                                                                                                                     |
| topic                        | required           | none)         | String | Topic name from which the table is read.                                                                                                                                                                 |
| properties.bootstrap.servers | required           | (none)        | String | Kafka brokers，逗号分隔.                                                                                                                                                                                      |
| properties.group.id          | required by source | (none)        | String | Kafka source consumer id                                                                                                                                                                                 |
| optional for Kafka sink.     |                    |               |        |                                                                                                                                                                                                          |
| format                       | required           | (none)        | String | deserialize/serialize Kafka messages 的[format](https://ci.apache.org/projects/flink/flink-docs-release-1.11/zh/dev/table/connectors/formats/). 可填 ‘csv’, ‘json’, ‘avro’, ‘debezium-json’ , ‘canal-json’. |
| scan.startup.mode            | optional           | group-offsets | String | Kafka consumer Startup mode, valid values：                                                                                                                                                               |

‘earliest-offset’  
‘latest-offset’  
‘group-offsets’(从 zk/kafka 里保存的指定 consumer group 的 offset 开始消费)  
‘timestamp’(从用户定义的`scan.startup.timestamp-millis`中时间戳开始消费)  
‘specific-offsets’(从用户定义的`scan.startup.specific-offsets`开始消费) |
| scan.startup.specific-offsets | optional | (none) | String | ‘specific-offsets’ startup mode 时指定每个分区开始消费的 offset, 如’partition:0,offset:42;partition:1,offset:300’. |
| scan.startup.timestamp-millis | optional | (none) | Long | ‘timestamp’ startup mode 时指定消费起始毫秒级时间戳 |
| sink.partitioner | optional | (none) | String | Flink’s partitions 映射到输出的 Kafka’s partitions 方式. 合法值如下：  
fixed: 每个 Flink partition 最多输入一个 Kafka partition  
round-robin: 每个 Flink partition 轮询方式映射到所有 Kafka partitions  
自定义 FlinkKafkaPartitioner subclass: 如 ‘org.mycompany.MyPartitioner’. |

### 5.2.4 例子

-   1.11.0 之前的例子，比较复杂：\`\`\`java
    .connect(new Kafka()

        .version("0.11")   

        .topic("student_info")  


        .property("zookeeper.connect", "localhost:2181")
        .property("bootstrap.servers", "localhost:9092")
        .property("group.id", "testGroup")


        .startFromEarliest()
        .startFromLatest()
        .startFromSpecificOffsets(...)



        .sinkPartitionerFixed()

        .sinkPartitionerRoundRobin()    

        .sinkPartitionerCustom(MyCustom.class)    

    )

    ```

    ```
-   1.11.0 的例子，简化了很多：\`\`\`java
    CREATE TABLE kafka_source_table (
     user_id BIGINT,
     item_id BIGINT,
     category_id BIGINT,
     behavior STRING,
     message ROW&lt;log_create_time BIGINT>,
     ts_field AS TO_TIMESTAMP(FROM_UNIXTIME(message.log_create_time)),
    ) WITH (
     'connector' = 'kafka',
     'topic' = 'user_behavior',
     'properties.bootstrap.servers' = 'localhost:9092',
     'properties.group.id' = 'testGroup',
     'format' = 'json',
     'scan.startup.mode' = 'group-offsets'
    )

    ```

    ```
-   1.12.0 的例子，支持获取 Kafka 元数据 \`\``sql
    CREATE TABLE KafkaTable (
      `event_time`TIMESTAMP(3) METADATA FROM 'value.source.timestamp' VIRTUAL,  
     `origin_table`STRING METADATA FROM 'value.source.table' VIRTUAL, 
     `partition_id`BIGINT METADATA FROM 'partition' VIRTUAL,  
     `offset`BIGINT METADATA VIRTUAL,  
     `user_id`BIGINT,
     `item_id`BIGINT,
     `behavior\` STRING
    ) WITH (
      'connector' = 'kafka',
      'topic' = 'user_behavior',
      'properties.bootstrap.servers' = 'localhost:9092',
      'properties.group.id' = 'testGroup',
      'scan.startup.mode' = 'earliest-offset',
      'value.format' = 'debezium-json'
    );

    ```

    ```

## 5.3 [Upsert Kafka](https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/table/connectors/upsert-kafka.html)

Flink 1.12 以前版本仅支持 append only 元素写入，而当我们产生聚合元素的时候无法使用 kafka sink.

Flink 1.12 特地新增`Upsert Kafka SQL Connector`解决此问题。

作为 Source 时：

-   生产`changelog`流
    -   有 value 的表示`UPDATE`
    -   value 为 null 的表示`DELETE`
-   作为 Sink 时，可消费`changelog`流
    -   `INSERT`/`UPDATE_AFTER`作为普通消息输出到 Kafka
    -   `DELETE`会作为`null`值数据以示这是该 key 对应数据的墓碑
    -   由于 Flink 按照用户定义的主键列值进行消息分区，可保证该分区数据有序，同 Key 数据也会发到相同分区

官方示例：

```sql
CREATE TABLE pageviews_per_region (
  user_region STRING,
  pv BIGINT,
  uv BIGINT,
  PRIMARY KEY (region) NOT ENFORCED
) WITH (
  'connector' = 'upsert-kafka',
  'topic' = 'pageviews_per_region',
  'properties.bootstrap.servers' = '...',
  'key.format' = 'avro',
  'value.format' = 'avro'
);

CREATE TABLE pageviews (
  user_id BIGINT,
  page_id BIGINT,
  viewtime TIMESTAMP,
  user_region STRING,
  WATERMARK FOR viewtime AS viewtime - INTERVAL '2' SECOND
) WITH (
  'connector' = 'kafka',
  'topic' = 'pageviews',
  'properties.bootstrap.servers' = '...',
  'format' = 'json'
);


INSERT INTO pageviews_per_region
SELECT
  user_region,
  COUNT(*),
  COUNT(DISTINCT user_id)
FROM pageviews
GROUP BY user_region;

```

## 5.4 Elasticsearch Connector

## 5.5 [JDBC Connector](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/connectors/jdbc.html)

### 5.5.1 概述

![](https://img-blog.csdnimg.cn/20200729162621147.png)

可以通过 JDBC Connector 读写 JDBC 数据源。

目前支持[Mysql](https://repo.maven.apache.org/maven2/mysql/mysql-connector-java/)、PostgreSQL、Derby 等，需要下载对应 Jar 包放入`$FLINK_HOME/lib`下

需要添加的依赖为

-   [flink-connector-jdbc_2.11-1.11.0.jar](https://repo.maven.apache.org/maven2/org/apache/flink/flink-connector-jdbc_2.11/1.11.0/flink-connector-jdbc_2.11-1.11.0.jar)
-   特定的数据库所需的 driver  
    比如 mysql-connector-java-5.1.47.jar

更多配置及数据库和 Flink 的类型映射请参考[官网](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/connectors/jdbc.html)。

### 5.5.2 Streaming Mode

在流场景可支持

-   Append（适用于只交互 INSERT 信息场景）  
    DDL 不提供主键时
-   Upsert（使用唯一 key，适用于交互增删改信息场景）模式。  
    DDL 提供主键时

Flink 自动从流式查询中提取有效 key。 例如，查询`SELECT a, b, c FROM t GROUP BY a, b`定义了字段 a 和 b 的组合键。

如果将 JDBC 表用作`upsert sink`，请确保查询的 key 是数据库的唯一键之一或是主键。 这样才可以保证输出结果符合预期。

### 5.5.3 Temporary Join 与查询缓存

JDBC Connector 可以被视为`lookup source`来做临时 join。

当前，只支持同步模式的\`lookup source。

可使用`loopup source cache`来提高临时 join 的性能（**默认不开启缓存**），具体原理是每个 TaskManager 先查询本节点持有的缓存而不是直接查询数据库本身，当目标不在缓存时再请求远程数据库，并将返回的结果更新到缓存。

**但务必注意，使用缓存时数据可能不是最新，这需要使用者进行权衡**！

loopup source cache 相关选项有：

-   lookup.cache.max-rows  
    lookup source cache 缓存的最大行数，超过时会删除最旧的缓存行。

    **注意设置了本项就必须同时设置`lookup.cache.ttl`**
-   lookup.cache.ttl  
    缓存内每行的最大有效时间，超过就删除最旧的行

    设置太小会导致数据库请求负载过大，所以需要权衡性能和时效性。一般可以用在缓慢变换的维表中。

    填写格式为`Time interval unit label recognized units: DAYS: (d | day | days), HOURS: (h | hour | hours), MINUTES: (min | minute | minutes), SECONDS: (s | sec | secs | second | seconds), MILLISECONDS: (ms | milli | millis | millisecond | milliseconds), MICROSECONDS: (µs | micro | micros | microsecond | microseconds), NANOSECONDS: (ns | nano | nanos | nanosecond | nanoseconds)`
-   connector.lookup.max-retries  
    可选的，当查询数据库表失败时的最大重试次数。

当数据作为维表时，如果使用缓存，需要注意：

-   指定缓存后 Join 维表时拿到的可能是过期数据，其实已经被改变。但设置缓存过期时间太短会造成频繁查库增加性能开销。而将缓存条数设置过大可能打爆内存（可以适当调大内存，但不是根本方式）。所以，缓存的大小和时间配置需要权衡和实验。不过通常来说，维表中的数据应该缓慢变化的。
-   Flink Sql 1.9 原生支持的维表关联只支持同步模式，如果需要异步模式或者想用其他的第三方存储只能够自己去实现。

### 5.5.4 数据写入缓存与 Flush

Flink JDBC Connector 会将数据先放入内存，到达触发条件时再 flush 到 jdbc 数据库中。相关配置如下：

-   sink.buffer-flush.interval  
    默认 1 秒，可选的。当 flush 间隔时间超过此值时，会有一个异步线程将数据刷入数据库表。

    可设为 0 禁用。

    可将`sink.buffer-flush.max-rows`设为 0 ，本值设为大于 0 的数，以实现完全异步的缓冲处理。
-   sink.buffer-flush.max-rows  
    默认 100 行，可选的。当数据条数（包括增、删、改）累积到该值时，才会将数据输入数据库表.

    可设为 0 禁用。
-   sink.max-retries  
    默认 3，可选的。当某批数据写入数据表失败时的最大重试次数。

### 5.5.5 主键

1.11 可使用 DDL 来定义主键了，定义后采用`UPSERT`流模式，否则为`APPEND`模式。

-   采用`UPSERT`流模式时，是根据主键来插入或更新记录，由此来保证幂等性。

    推荐将设置主键，且需要确保主键是该数据表的唯一键或主键。
-   `APPEND`模式下，所有数据都作为`INSERT`消息处理，当底层数据库表的主键或唯一键冲突时会导致插入失败。

### 5.5.6 幂等写入

JDBCSink 在 DDL 中定义了主键列的情况，会使用 UPSERT 语义而不是 INSERT 语义。

UPSERT 语义可使得当底层数据库存在唯一键约束时，原子性得插入一行或更新已有行，提供了幂等性。

应用场景：

-   这样一来，即使发生失败需要从最后成功的 Checkpoint 重启导致重复处理数据，也能保证幂等，不会出现数据重复。
-   数据源头生成相同主键数据，需要更新

没有标准 upsert 语法，根据底层数据库不同而不同，比如 MySQL 语法如下：

```sql
INSERT .. ON DUPLICATE KEY UPDATE ..

```

### 5.5.7 Partitioned Scan

目的是加速从 JDBC 数据表读取数据，手段是使用并行 Source task 多实例。

以下任一`scan partition`选项被指定，则所有`scan partition`选项都必须被指定。作用是在并行读取数据时指定将表分区的方式。以下这些选项是当从表读数据时使用：

-   scan.partition.column  
    线程读取表时分区的列，只能是数值、日期或 timestamp
-   scan.partition.num  
    partitions 数，决定了多线程并行读取表时指定怎么对表分区.
-   scan.partition.lower-bound  
    用于确定分区步幅下界（不是用于过滤表中的行）
-   scan.partition.upper-bound  
    用于确定分区步幅上界

### 5.5.8 例子

-   1.11 以前 \`\`\`sql
    CREATE TABLE MyUserTable (...) WITH (
      'connector.type' = 'jdbc', 

      'connector.url' = 'jdbc:mysql://localhost:3306/flink-test', 

      'connector.table' = 'jdbc_table_name',  

      'connector.driver' = 'com.mysql.jdbc.Driver',

      'connector.username' = 'name',
      'connector.password' = 'password',

      'connector.read.partition.column' = 'column_name', 
      'connector.read.partition.num' = '50', 
      'connector.read.partition.lower-bound' = '500', 
      'connector.read.partition.upper-bound' = '1000', 

      'connector.read.fetch-size' = '100',

      'connector.lookup.cache.max-rows' = '5000', 
      'connector.lookup.cache.ttl' = '10s', 
      'connector.lookup.max-retries' = '3', 

      'connector.write.flush.max-rows' = '5000', 
      'connector.write.flush.interval' = '2s', 
      'connector.write.max-retries' = '3' 
    )

    ```

    ```
-   1.11\`\`\`sql

    CREATE TABLE MyUserTable (
      id BIGINT,
      name STRING,
      age INT,
      status BOOLEAN,
      PRIMARY KEY (id) NOT ENFORCED
    ) WITH (
       'connector' = 'jdbc',
       'url' = 'jdbc:mysql://localhost:3306/mydatabase',
       'table-name' = 'users'
    );

    INSERT INTO MyUserTable
    SELECT id, name, age, status FROM T;

    SELECT id, name, age, status FROM MyUserTable;

    SELECT \* FROM myTopic
    LEFT JOIN MyUserTable FOR SYSTEM_TIME AS OF myTopic.proctime
    ON myTopic.key = MyUserTable.id;

    ```

    ```

## 5.6 [Hive](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/hive)

请点击[HiveConnector](https://blog.csdn.net/baichoufei90/article/details/107832306#hive)

## 5.7 自定义 Source Sink

### 5.7.1 概述

请参考[User-defined Sources & Sinks](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/sourceSinks.html)

### 5.7.2 自定义 Source

#### 5.7.2.1 ScanTableSource

##### 5.7.2.1.1 概述

ChangeLog，Source 在 batch 场景发送有界的、insert-only 流；Streaming 场景发送无界的、insert-only 流；CDC(Change Data Capture) 场景发送无界 / 有界的、可包含增删改的记录的流。

从外部数据中扫描整张表的所有行。可包含增删改的行，所以可被用来读取 changelog。

`getChangelogMode`返回`ChangelogMode`，表示运行时可能返回的 Changelog 类型集合。

如果 Source 有一些额外的能力，可以实现`org.apache.flink.table.connector.source.abilities`包内的接口，如`SupportsProjectionPushDown`投影下推。

ScanTableSource 生产的记录必须是`org.apache.flink.table.data.RowData`，用来在 Flink 内部传递

其中定义了一个重要的方法`getScanRuntimeProvider`，需要得到一个`ScanRuntimeProvider`，可通过`InputFormatProvider.of(InputFormat<RowData, ?> inputFormat)`得到，如`HBaseDynamicTableSource`：

```java
@Override
public ScanRuntimeProvider getScanRuntimeProvider(ScanContext runtimeProviderContext) {
	return InputFormatProvider.of(new HBaseRowDataInputFormat(conf, tableName, hbaseSchema, nullStringLiteral));
}

```

也可通过`SourceFunctionProvider of(SourceFunction<RowData> sourceFunction, boolean isBounded)`得到，如`KafkaDynamicSourceBase`：

```java
@Override
public ScanRuntimeProvider getScanRuntimeProvider(ScanContext runtimeProviderContext) {
	DeserializationSchema<RowData> deserializationSchema =
			this.decodingFormat.createRuntimeDecoder(runtimeProviderContext, this.outputDataType);
	
	FlinkKafkaConsumerBase<RowData> kafkaConsumer =
			getKafkaConsumer(topic, properties, deserializationSchema);
	return SourceFunctionProvider.of(kafkaConsumer, false);
}

```

##### 5.7.2.1.2 HBaseDynamicTableSource

```java
public class HBaseRowDataInputFormat extends AbstractTableInputFormat<RowData> {
	public HBaseRowDataInputFormat(
			org.apache.hadoop.conf.Configuration conf,
			String tableName,
			HBaseTableSchema schema,
			String nullStringLiteral) {
		super(conf);
		this.tableName = tableName;
		this.schema = schema;
		this.nullStringLiteral = nullStringLiteral;
	}
}

```

```java
abstract class AbstractTableInputFormat<T> extends RichInputFormat<T, TableInputSplit> {
	public AbstractTableInputFormat(org.apache.hadoop.conf.Configuration hConf) {
		serializedConfig = HBaseConfigurationUtil.serializeConfiguration(hConf);
	}
}

```

1.  HBaseRowDataInputFormat#configure  
    创建了 serde、Connection、HTable、Scan(此时无 startRow 和 stopRow)
2.  createInputSplits  
    获取该 HTable 的每个 Region Start 和 End Key，为每个 Region 创建一个`TableInputSplit`（包含 spilitId, hosts, tableName(), splitStart, splitStop），构成`TableInputSplit[]`
3.  每个并行的 input task 创建一个实例，并调用`open(T split)`方法为该并行实例负责的 spilit 初始化。\`\`\`java

    currentRow = split.getStartRow();
    scan.setStartRow(currentRow);
    scan.setStopRow(split.getEndRow());

    resultScanner = table.getScanner(scan);

    endReached = false;
    scannedRows = 0;

    ```

    ```
4.  读取数据 \`\`\`java

    public T nextRecord(T reuse) throws IOException {
    	Result res;
    	try {

        	res = resultScanner.next();
        } catch (Exception e) {
        	resultScanner.close();
        	
        	LOG.warn("Error after scan of " + scannedRows + " rows. Retry with a new scanner...", e);
        	
        	scan.withStartRow(currentRow, false);
        	resultScanner = table.getScanner(scan);
        	res = resultScanner.next();
        }
        if (res != null) {
        	scannedRows++;
        	
        	currentRow = res.getRow();
        	
        	return mapResultToOutType(res);
        }

        endReached = true;
        return null;	

    }	

    ```

    ```
5.  close  
    当`reachedEnd`后说明 一个 input split 读取完成后调用本方法，被用来关闭 channel 和流，释放资源。\`\`\`java
    currentRow = null;
    try {if (resultScanner != null) {
    		resultScanner.close();
    	}
    } finally {
    	resultScanner = null;
    }

    ```

    ```
6.  可能会再次 open 来读取下一个 split

### 5.7.3 自定义 Sink

### 5.7.4 自定义 TableFactory

## 5.8 [DataGen](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/connectors/datagen.html)

### 5.8.1 概述

DataGen 是一个调试用的 Connector，可以配合使用[Computed Column syntax](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/sql/create.html#create-table)来生成简单类型数据。

### 5.8.2 数据生成器

目前有两类：

-   RandomGenerator  
    默认选项，是一个无界数据生成器。

    可指定随机值的最大和最小区间。

    对于字符串类型数据（char/varchar/string），可指定长度。
-   SequenceGenerator  
    是一个有界数据生成器。

    可指定序列的开始和结束值，当 Sequence 抵达结束值时，读取结束。

### 5.8.3 配置

转自官网：  
![](https://img-blog.csdnimg.cn/20200803214657491.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

### 5.8.4 例子

```sql
CREATE TABLE datagen (
 f_sequence INT,
 f_random INT,
 f_random_str STRING,
 
 ts AS localtimestamp,
 WATERMARK FOR ts AS ts
) WITH (
 'connector' = 'datagen',

 
 
 'rows-per-second'='5',
 
 'fields.f_sequence.kind'='sequence',
 'fields.f_sequence.start'='1',
 'fields.f_sequence.end'='1000',
 
 'fields.f_random.min'='1',
 'fields.f_random.max'='1000',
 
 'fields.f_random_str.length'='10'
)

```

## 5.9 HBase Connector

参考[Functions](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/functions/)

## 6.1 内置函数

## 6.2 UDF

参考:[Configuration](https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/config.html)

## 7.1 checkpoint

```java
pipeline.time-characteristic EventTime
execution.checkpointing.interval 120000
execution.checkpointing.min-pause 60000
execution.checkpointing.timeout 60000
execution.checkpointing.externalized-checkpoint-retention RETAIN_ON_CANCELLATION
execution.savepoint.path hdfs:/tmp/flink-checkpoints/xxx/chk-yyy  

```

## 8.1 概述

目前官网只有流式聚合调优一项。

## 8.2 流式聚合调优调优

参考[Streaming Aggregation](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/tuning/streaming_aggregation_optimization.html)

### 8.2.1 概述

Flink SQL 允许用户直接用 SQL 来定义效率高的流式分析程序，已经做了大量工作来做查询优化和算子调优，但有一些选项需要手动开启进行调优。

注意：

-   当前这里的调优只支持 Blink Planner。
-   当前流式聚合调优只支持[无界聚合](https://ci.apache.org/projects/flink/flink-docs-master/dev/table/sql/queries.html#aggregations)，未来会支持对窗口聚合的调优。

默认状况下，无界聚合算子逐条处理输入记录：

1.  从状态读取累加器`accumulator`
2.  从累加器中累加 / 撤回记录
3.  将累加器写回位于 StateBackend 的状态中
4.  下一条记录又从步骤 1 到 3 进行处理

通过了解无界聚合处理步骤，可以看到这样的处理模式会增加 StateBackend（尤其是硬盘上的 RocksDBStatebackend）的开销。此外，数据倾斜在生产中非常常见，更会加重此问题，使得 job 容易发生反压。

### 8.2.2 MiniBatch 聚合

#### 8.2.2.1 概述

MiniBatch 聚合核心思想是在聚合算子内部缓冲区缓存一组输入的数据，当触发计算时，同一个 key 的所有记录只需要一个操作即可访问状态了。  
![](https://img-blog.csdnimg.cn/20200822121158947.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

#### 8.2.2.2 特点

-   这和传统无界聚合方法有很大不同，这样做能很大程度减少状态开销，提升吞吐。
-   但同时也会因为需要缓存输入到聚合算自内部而带来额外开销和延时，而不是立刻处理每条输入记录，其实就是吞吐和延时之间的一个折中

#### 8.2.2.3 配置

MiniBatch 聚合默认关闭。

要开启，需要配置：

-   table.exec.mini-batch.enabled  
    是否允许 MiniBatch 聚合。一旦开启，必须配置一下两个选项。
-   table.exec.mini-batch.allow-latency  
    最大允许的 MiniBatch 聚合延迟时间
-   table.exec.mini-batch.size  
    最大允许的 MiniBatch 聚合缓存数据大小，级别是每个聚合算子 task

配置方法：

```java

val tEnv: TableEnvironment = ...


val configuration = tEnv.getConfig().getConfiguration()

configuration.setString("table.exec.mini-batch.enabled", "true") 
configuration.setString("table.exec.mini-batch.allow-latency", "5 s") 
configuration.setString("table.exec.mini-batch.size", "5000") 

```

### 8.2.3 LocalGlobalBatch 聚合

#### 8.2.3.1 概述

LocalGlobalBatch 聚合主要目的是解决数据倾斜，方法是将`group by`聚合拆解为两个阶段：

1.  先在上游做本地聚合
2.  在下游做全局聚合

这个思路类似于 MR 中的 MapCombine + Reduce Combine 两个阶段。

比如以下 SQL

```java
SELECT color, sum(id)
FROM T
GROUP BY color

```

有可能上游`color`字段数据已经倾斜，因此下游的某几个聚合算子实例必须处理比其他实例更多的数据，这就造成了数据热点，即数据倾斜。

此时就可使用 LocalGlobalBatch 聚合，在上游算子将同 key 数据进行本地聚合，放入同一个累加器，只将聚合后的累加器发送到下游聚合算子，而不是原始的所有记录，这样大大减少网络 shuffle 时的开销以及状态访问开销。  
![](https://img-blog.csdnimg.cn/20200822131542796.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

每次本地聚合时累加的输入记录数目取决于`mini-batch interval`，也就是说 LocalGlobalBatch 聚合前提是`mini-batch`优化必须已经开启。

#### 8.2.3.2 特点

大大减少网络 shuffle 时的开销以及状态访问开销

#### 8.2.3.3 配置

LocalGlobalBatch 聚合默认关闭。

要开启，需要配置：

-   table.exec.mini-batch.enabled  
    是否允许 MiniBatch 聚合。一旦开启，必须配置一下两个选项。
-   table.optimizer.agg-phase-strategy  
    TWO_PHASE

    默认 AUTO（根据 cost 来决定采用两阶段还是一阶聚段合），还可填 TWO_PHASE(两阶段聚合)、ONE_PHASE(一阶段聚合)

配置方法：

```java

val tEnv: TableEnvironment = ...


val configuration = tEnv.getConfig().getConfiguration()

configuration.setString("table.exec.mini-batch.enabled", "true") 
configuration.setString("table.exec.mini-batch.allow-latency", "5 s")
configuration.setString("table.exec.mini-batch.size", "5000")
configuration.setString("table.optimizer.agg-phase-strategy", "TWO_PHASE") 

```

### 8.2.4 拆分 DistinctBatch 聚合

#### 8.2.4.1 概述

LocalGlobalBatch 聚合优化可在常规聚合场景有效消除数据倾斜，如 SUM, COUNT, MAX, MIN, AVG，但不适合处理`DISTINCT`聚合！

比如计算每天的用户 UV:

```sql
SELECT day, COUNT(DISTINCT user_id)
FROM T
GROUP BY day

```

当 distinct key（比如这里的 user_id，因为 distinct 的是`user_id`，而不是`day`）稀疏时，`COUNT DISTINCT`一般不能有效聚合而减少记录数，即使开启了 LocalGlobalBatch 聚合提升也不大。因为 key 稀疏场景下在`COUNT DISTINCT`后累加器依然持有大部分原始记录（比如这里在本地聚合后，只是得到了去重后的`user_id`而已），所以全局聚合阶段依然存在瓶颈。比如以上例子中，大多数负载高的累加器由一个 task 处理，也就是属于同一天的被一个 task 处理。

针对`COUNT DISTINCT`以上问题，就有了拆分 DistinctBatch 聚合优化思想，将`COUNT DISTINCT`拆分为两个层级：

1.  首次聚合先通过`group key`和额外自定义的`bucket key`进行 shuffle，并进行聚合计算。  
    `bucket key`是通过`HASH_CODE(distinct_key) % BUCKET_NUM`计算，`BUCKET_NUM`默认 1024，可通过`table.optimizer.distinct-agg.split.bucket-num`调整。
2.  第二次聚合通过原始`group key`进行 shuffle，并使用`SUM`聚合来自不同 bucket 的 `COUNT DISTINCT` 值

这个过程如下图，这里不同颜色表示热点`group key`，类似我们例子的`day`；字母表示 user_id：左图为本地全局聚合，右图为拆分 DistinctBatch 聚合：  
![](https://img-blog.csdnimg.cn/20200823145035526.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

可以看到：

-   开启本地全局聚合计算`COUNT DISTINCT`时，本地聚合阶段其实并没有减少太多数据量，导致传输到右上聚合算子时的数据过多，造成数据热点和倾斜，成为瓶颈。
-   开启拆分 DistinctBatch 聚合计算`COUNT DISTINCT`时，首先根据 group key 即 day 和 distinct key 即 user_id 计算 hash 并对 bucket 取余后得到 BucketKey 联合进行 shuffle，这个时候就已经将热点数据打散到各个第一阶段聚合算子上了。这一阶段就会计算`COUNT DISTINCT`，输出就是唯一值的数量；

    随后根据 group key 即 day 进行第二阶段聚合，此时由于第一阶段已经将相同 group by 数据聚合计算，所以第二阶段做的工作就少了很多了。此时只需要将第一阶段的唯一值 SUM 求和即可。

相同 distinct key 的行会在同一个 Bucket 内计算，所以能这样分阶段聚合。第一阶段聚合的意义是分担热点的`group key`，比如以上例子时可将最新一天的数据 shuffle 到不同 task 计算，有效解决 `COUNT DISTINCT` 时的数据倾斜和数据热点问题。

在拆分 DistinctBatch 聚合优化后，以上查询会被自动重写：

```sql
SELECT day, SUM(cnt)
FROM (
    SELECT day, COUNT(DISTINCT user_id) as cnt
    FROM T
    GROUP BY day, MOD(HASH_CODE(user_id), 1024)
)
GROUP BY day

```

以上为做简单的例子，但其实 Flink 支持拆分更复杂的聚合查询，比如，多个具有不同 distinct key （例如 COUNT(DISTINCT a), SUM(DISTINCT b) ）的 distinct 聚合，可以与其他非 distinct 聚合（例如 SUM、MAX、MIN、COUNT ）配合使用。

需要注意的是，目前聚合拆分优化不支持包含用户定义的 AggregateFunction 的聚合运算。

#### 8.2.4.2 特点

适用于 distinct 聚合时出现数据倾斜的场景。

#### 8.2.4.3 配置

配置方法：

```java

val tEnv: TableEnvironment = ...

tEnv.getConfig         
  .getConfiguration    
  .setString("table.optimizer.distinct-agg.split.enabled", "true")  

```

相关选项

-   table.optimizer.distinct-agg.split.enabled  
    默认 false，是否开启拆分 distinct 聚合
-   table.optimizer.distinct-agg.split.bucket-num  
    默认 1024，开启聚合拆分后的 Bucket 数量

### 8.2.5 在 Distinct 聚合上使用`Filter`修饰符替换`CASE WHEN`

有时用户可能需要从不同维度计算 UV 的数量，例如来自 Android 的 UV、iPhone 的 UV、Web 的 UV 和总 UV。一般会选择使用 `CASE WHEN`，例如：

```sql
SELECT
 day,
 COUNT(DISTINCT user_id) AS total_uv,
 COUNT(DISTINCT CASE WHEN flag IN ('android', 'iphone') THEN user_id ELSE NULL END) AS app_uv,
 COUNT(DISTINCT CASE WHEN flag IN ('wap', 'other') THEN user_id ELSE NULL END) AS web_uv
FROM T
GROUP BY day

```

在这种情况下，Flink 官方建议使用 `FILTER` 语法而不是 `CASE WHEN`，因为 FILTER 更符合 SQL 标准，并且能获得更多的性能提升。

FILTER 是用于限制聚合中使用的 value 的聚合函数修饰符。

将上面的示例替换为 FILTER 后如下所示：

```sql
SELECT
 day,
 COUNT(DISTINCT user_id) AS total_uv,
 COUNT(DISTINCT user_id) FILTER (WHERE flag IN ('android', 'iphone')) AS app_uv,
 COUNT(DISTINCT user_id) FILTER (WHERE flag IN ('wap', 'other')) AS web_uv
FROM T
GROUP BY day

```

Flink SQL 优化器可以识别在相同 `distinct key`上的不同 Filter 参数。例如，在上面的示例中，这三个 `COUNT DISTINCT` 都针对 `user_id`列。随后，Flink 可以只使用一个共享的 State 实例而不是三个 State 实例，这样可以减少 State 访问开销和 State 大小。在某些工作负载下，这样做可获得显著的性能提升。

## 8.3 Query Configuration

### 8.3.1 概述

有时需要限制 State 的大小，避免在消费无界流时状态无限增大。需要根据时间语义、查询本身场景业务要求、对计算结果精度影响来综合评估。

主要就是通过`TableConfig`来设定相关运行参数：

```java
val env = StreamExecutionEnvironment.getExecutionEnvironment
val tableEnv = StreamTableEnvironment.create(env)


val tConfig: TableConfig = tableEnv.getConfig

tConfig.setIdleStateRetentionTime(Time.hours(12), Time.hours(24))

```

### 8.3.2 空闲状态保留时长控制

#### 8.3.2.1 背景

Flink 需要为那些在一个或多个 key 上进行聚合或 Join 查询搜集数据或维护每个 key 的部分结果。随着更多唯一 key 流入，持续查询累积的 State 会不断增大。然而，通常经过一段时间后，一些 key 就不再活跃，对应的状态也随之变得无用了。

如现有一个统计每个 session 的点击事件：

```sql
SELECT sessionId, COUNT(*) FROM clicks GROUP BY sessionId;

```

此时 Flink 就会为每个`GROUP BY KEY` `sessionId`维护一个 count 值。

这种场景下，在客户端 session 结束后 sessionId 就变得不再活跃了。但 Flink 的 sessionId State 不能感知到并自动清理，会认为这些 sessionId 将来还会出现并实现。这样就导致该查询程序的整个 State 大小不断增加。

此时可配置`Idle State Retention Time`参数，他定义了每个 key 的状态在最后一次被修改后的保留时长。比如可谓上例的 sessionId 配置一个过期时间。

#### 8.3.2.2 注意事项

但需要注意，如果一个 key 的状态被移除，则该持续查询程序会彻底遗忘是否见过该 key。如果该 key 后面再次出现，则会被认为是首次出现！比如上例中，假设 sessionId abc 的 count 状态被移除，但一段时间后该 abc 的 session 再次出现，则会被给与一个初始化的`count = 0` 的状态！这可导致之前的结果被覆盖！

#### 8.3.2.3 配置

```java
val tConfig: TableConfig = ???


tConfig.setIdleStateRetentionTime(Time.hours(12), Time.hours(24))

```

-   min  
    表示不活跃的空闲状态的最小保留时间
-   max  
    表示不活跃的空闲状态的最大保留时间

注意，他们之间的差值至少为 5 分钟。如果两值都设为 0，表示永不清理状态。

这两个值可保证，不活跃的空闲状态至少保留`min`，但也绝不会超过`max`。

#### 8.3.2.4 源码分析

FlinkSql 关于 State 的相关函数有一个重要的基类`KeyedProcessFunction`，内部定义了两个重要的方法

-   public abstract void processElement(I value, Context ctx, Collector out) throws Exception;  
    处理每条输入元素
-   public void onTimer(long timestamp, OnTimerContext ctx, Collector out) throws Exception {}  
    当使用`TimerService`注册的`Timer`触发时调用

我们看一个用于表的`group by`聚合的类`GroupTableAggFunction`，它继承自`KeyedProcessFunctionWithCleanupState`，同是也是 KeyedProcessFunction 的子孙类：

```java

@Override
public void open(Configuration parameters) throws Exception {
	...
	
	RowDataTypeInfo accTypeInfo = new RowDataTypeInfo(accTypes);
	ValueStateDescriptor<RowData> accDesc = new ValueStateDescriptor<>("accState", accTypeInfo);
	accState = getRuntimeContext().getState(accDesc);
			
	initCleanupTimeState("GroupTableAggregateCleanupTime");
}


protected void initCleanupTimeState(String stateName) {
	
	if (stateCleaningEnabled) {
		ValueStateDescriptor<Long> inputCntDescriptor = new ValueStateDescriptor<>(stateName, Types.LONG);
		
		cleanupTimeState = getRuntimeContext().getState(inputCntDescriptor);
	}
}

```

处理每条记录：

```java
@Override
public void processElement(RowData input, Context ctx, Collector<RowData> out) throws Exception {
	
	long currentTime = ctx.timerService().currentProcessingTime();
	
	registerProcessingCleanupTimer(ctx, currentTime);
	
	RowData currentKey = ctx.getCurrentKey();

	boolean firstRow;
	
	RowData accumulators = accState.value();
	
	if (null == accumulators) {
		firstRow = true;
		accumulators = function.createAccumulators();
	} else {
		firstRow = false;
	}

	
	function.setAccumulators(accumulators);
	
	
	if (!firstRow && generateUpdateBefore) {
		function.emitValue(out, currentKey, true);
	}

	
	if (isAccumulateMsg(input)) {
		
		
		function.accumulate(input);
	} else {
		
		function.retract(input);
	}

	
	accumulators = function.getAccumulators();
	if (!recordCounter.recordCountIsZero(accumulators)) {
		
		
		function.emitValue(out, currentKey, false);

		
		accState.update(accumulators);

	} else {
		
		accState.clear();
		
		function.cleanup();
	}
}

```

KeyedProcessFunctionWithCleanupState#registerProcessingCleanupTimer

```java
protected void registerProcessingCleanupTimer(Context ctx, long currentTime) throws Exception {
	if (stateCleaningEnabled) {
		registerProcessingCleanupTimer(
			cleanupTimeState,
			
			currentTime,
			minRetentionTime,
			maxRetentionTime,
			ctx.timerService()
		);
	}
}

```

CleanupState#registerProcessingCleanupTimer  
其实这是一个接口，定义了 default 方法

```java
default void registerProcessingCleanupTimer(
		ValueState<Long> cleanupTimeState,
		long currentTime,
		long minRetentionTime,
		long maxRetentionTime,
		TimerService timerService) throws Exception {

	
	Long curCleanupTime = cleanupTimeState.value();

	
	
	
	
	
	
	
	
	if (curCleanupTime == null || (currentTime + minRetentionTime) > curCleanupTime) {
		
		
		long cleanupTime = currentTime + maxRetentionTime;
		
		timerService.registerProcessingTimeTimer(cleanupTime);
		
		if (curCleanupTime != null) {
			
			timerService.deleteProcessingTimeTimer(curCleanupTime);
		}
		
		cleanupTimeState.update(cleanupTime);
	}
}

```

onTimer  
定时清理状态

```java
@Override
public void onTimer(long timestamp, OnTimerContext ctx, Collector<RowData> out) throws Exception {
	if (stateCleaningEnabled) {
		
		cleanupState(accState);
		function.cleanup();
	}
}


protected void cleanupState(State... states) {
	for (State state : states) {
		state.clear();
	}
	this.cleanupTimeState.clear();
}

```

#### 8.3.2.5 小结

-   假设 minRetentionTime = 12 小时，maxRetentionTime=24 小时
-   第一次该元素 A 来的时间为 20200827 00:00，则 curCleanupTime=20200828 00:00，并将 A 元素放入累加器，并放入 State 保存
-   A 第二次出现是 20200827 11:00，则 12:00+12 小时 = 20200828 00:00 不大于 curCleanupTime，此时还是什么都不做。当然，还是需要将 A 放入累加器累加，并放入 State 保存。

    也就是说如果在 20200828 00:00 前 A 不再出现，则对应的累加器状态会在 0 点被清理，虽然在 12 小时前出现过一次，\*\*这也就是状态的最小保留时间的含义！\*\*也就是说虽然 A 最后一次出现是 12 小时以前，但在此 12 小时后就被清理了，符合状态最小保留时间。
-   A 第三次出现是 20200827 13:00，则 13:00+12 小时 = 20200828 01:00 大于 curCleanupTime，此时就会更新 curCleanupTime = 20200827 13:00+24 小时 = 20200828 13:00

    随后将 A 放入累加器累加，并放入 State 保存。
-   如果 A 第四次出现是 20200829 14:00，此时已经距离上次出现超过 24 小时，则 Key A 对应的累加器状态已经被清理！

    此时新来的 A 会被当做首次出现，创建新的累加器并放入 State。

## 9.1 维表 Join

**本小节转自\[Flink Sql 教程（3）- 维表 Join]**([https://blog.csdn.net/weixin\\\_47482194/article/details/106672613](https://blog.csdn.net/weixin\_47482194/article/details/106672613))，作者 Flink - 狄杰

-   什么是维表  
    维表，维度表的简称，来源于数据仓库，一般用来给事实数据补充信息。假设现在有一张销售记录表。销售记录表里面的一条销售记录就是一条事实数据，而这条销售记录中的地区字段就是一个维度。通常销售记录表里面的地区字段是地区表的主键，地区表就是一张维表。更多的细节可以面向百度 / 谷歌编程。
-   为什么 Flink 中需要维表  
    以流计算为例，一般情况下，消费的消息中间件中的消息，是事实表中的数据，我们需要把数据补全，不然谁也不知道字段地区对应的值 01、02 是个什么东西。所以，我们通常会在计算过程中，通过 Join 维表来补全数据。

维表相关配置可见[join cache](#joincache)

[Flink Sql 教程（3）- 维表 Join](https://blog.csdn.net/weixin_47482194/article/details/106672613)

-   [Flink 官网](https://ci.apache.org/projects/flink/flink-docs-release-1.11/)  
    部分内容翻译或转自官网
-   《Stream Processing with Apache Flink》  
    出处：Flink 极客训练营  
    作者：崔星灿
-   《Flink SQL \_ Table 介绍与实战》  
       出处：Flink 极客训练营  
       作者：伍翀 
    [https://blog.csdn.net/baichoufei90/article/details/101054148](https://blog.csdn.net/baichoufei90/article/details/101054148)
