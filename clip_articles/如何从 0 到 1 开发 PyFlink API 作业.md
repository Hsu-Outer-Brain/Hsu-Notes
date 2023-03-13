# 如何从 0 到 1 开发 PyFlink API 作业
[如何从 0 到 1 开发 PyFlink API 作业](https://mp.weixin.qq.com/s/GyFTjQl6ch8jc733mpCP7Q) 

 **摘要：** Apache Flink 作为当前最流行的流批统一的计算引擎，在实时 ETL、事件处理、数据分析、CEP、实时机器学习等领域都有着广泛的应用。从 Flink 1.9 开始，Apache Flink 社区开始在原有的 Java、Scala、SQL 等编程语言的基础之上，提供对于 Python 语言的支持。经过 Flink 1.9 ～ 1.12 以及即将发布的 1.13 版本的多个版本的开发，目前 PyFlink API 的功能已经日趋完善，可以满足绝大多数情况下 Python 用户的需求。接下来，我们以 Flink 1.12 为例，介绍如何使用 Python 语言，通过 PyFlink API 来开发 Flink 作业。内容包括：

1.  环境准备
2.  作业开发  
3.  作业提交  
4.  问题排查  
5.  总结

**Tips：** 点击文末「**阅读原文**」查看更多技术干货～

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2017-38-20/49ca2ea2-2bcb-4316-8b8c-88344d8d5158.png?raw=true)
 GitHub 地址 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2017-38-20/80f3f373-2c3e-4f0e-8afb-edddd13e8bd6.png?raw=true)

[https://github.com/apache/flink](https://github.com/apache/flink)

欢迎大家给 Flink 点赞送 star~

**一、环境准备**

**第一步：安装 Python**  

* * *

PyFlink 仅支持 Python 3.5+，您首先需要确认您的开发环境是否已安装了 Python 3.5+，如果没有的话，需要先安装 Python 3.5+。

### **第二步：安装 JDK**

我们知道 Flink 的运行是使用 Java 语言开发的，所以为了执行 Flink 作业，您还需要安装 JDK。Flink 提供了对于 JDK 8 以及 JDK 11 的全面支持，您需要确认您的开发环境中是否已经安装了上述版本的 JDK，如果没有的话，需要先安装 JDK。

### **第三步：安装 PyFlink**

接下来需要安装 PyFlink，可以通过以下命令进行安装：

```sql

python3 -m pip install virtualenv
virtualenv -p `which python3` venv


./venv/bin/activate


python3 -m pip install apache-flink==1.12.2
```

**二、作业开发**

## **PyFlink Table API 作业**

我们首先介绍一下如何开发 PyFlink Table API 作业。

#### **■ 1）创建 TableEnvironment 对象**

对于 Table API 作业来说，用户首先需要创建一个 TableEnvironment 对象。以下示例定义了一个 TableEnvironment 对象，使用该对象的定义的作业，运行在流模式，且使用 blink planner 执行。

```http
env_settings = EnvironmentSettings.new_instance().in_streaming_mode().use_blink_planner().build()
t_env = StreamTableEnvironment.create(environment_settings=env_settings)
```

#### **■ 2）配置作业的执行参数**

可以通过以下方式，配置作业的执行参数。以下示例将作业的默认并发度设置为 4。

```javascript
t_env.get_config().get_configuration().set_string('parallelism.default', '4')
```

#### **■ 3）创建数据源表**

接下来，需要为作业创建一个数据源表。PyFlink 中提供了多种方式来定义数据源表。

**方式一：from_elements**

PyFlink 支持用户从一个给定列表，创建源表。以下示例定义了包含了 3 行数据的表：\[("hello", 1), ("world", 2), ("flink", 3)]，该表有 2 列，列名分别为 a 和 b，类型分别为 VARCHAR 和 BIGINT。

```http
tab = t_env.from_elements([("hello", 1), ("world", 2), ("flink", 3)], ['a', 'b'])
```

说明：  

-   这种方式通常用于测试阶段，可以快速地创建一个数据源表，验证作业逻辑。


-   from_elements 方法可以接收多个参数，其中第一个参数用于指定数据列表，列表中的每一个元素必须为 tuple 类型；第二个参数用于指定表的 schema。

**方式二：DDL**

除此之外，数据也可以来自于一个外部的数据源。以下示例定义了一个名字为 my_source，类型为 datagen 的表，表中有两个类型为 VARCHAR 的字段。

```python
t_env.execute_sql("""
        CREATE TABLE my_source (
          a VARCHAR,
          b VARCHAR
        ) WITH (
          'connector' = 'datagen',
          'number-of-rows' = '10'
        )
    """)

tab = t_env.from_path('my_source')
```

说明：

-   通过 DDL 的方式来定义数据源表是目前最推荐的方式，且所有 Java Table API & SQL 中支持的 connector，都可以通过 DDL 的方式，在 PyFlink Table API 作业中使用，详细的 connector 列表请参见 Flink 官方文档 \[1]。


-   当前仅有部分 connector 的实现包含在 Flink 官方提供的发行包中，比如 FileSystem，DataGen、Print、BlackHole 等，大部分 connector 的实现当前没有包含在 Flink 官方提供的发行包中，比如 Kafka、ES 等。针对没有包含在 Flink 官方提供的发行包中的 connector，如果需要在 PyFlink 作业中使用，用户需要显式地指定相应 FAT JAR，比如针对 Kafka，需要使用 JAR 包 \[2]，JAR 包可以通过如下方式指定：

```apache
# 注意：file:///前缀不能省略
t_env.get_config().get_configuration().set_string("pipeline.jars", "file:///my/jar/path/flink-sql-connector-kafka_2.11-1.12.0.jar")
```

**方式三：catalog**

```makefile
hive_catalog = HiveCatalog("hive_catalog")
t_env.register_catalog("hive_catalog", hive_catalog)
t_env.use_catalog("hive_catalog")

# 假设hive catalog中已经定义了一个名字为source_table的表
tab = t_env.from_path('source_table')
```

这种方式和 DDL 的方式类似，只不过表的定义事先已经注册到了 catalog 中了，不需要在作业中重新再定义一遍。

#### ■ **4）定义作业的计算逻辑**

**方式一：通过 Table API**

得到 source 表之后，接下来就可以使用 Table API 中提供的各种操作，定义作业的计算逻辑，对表进行各种变换，比如：

```sql
@udf(result_type=DataTypes.STRING())
def sub_string(s: str, begin: int, end: int):
   return s[begin:end]

transformed_tab = tab.select(sub_string(col('a'), 2, 4))
```

**方式二：通过 SQL 语句**

除了可以使用 Table API 中提供的各种操作之外，也可以直接通过 SQL 语句来对表进行变换，比如上述逻辑，也可以通过 SQL 语句来实现：

```makefile
t_env.create_temporary_function("sub_string", sub_string)
transformed_tab = t_env.sql_query("SELECT sub_string(a, 2, 4) FROM %s" % tab)
```

说明：

-   TableEnvironment 中提供了多种方式用于执行 SQL 语句，其用途略有不同：

| 方法名 | 使用说明 |
| --- | ---- |

\| 

sql_query

 \| 

用来执行 SELECT 语句

 \|
\| 

sql_update

 \| 

用来执行 INSERT 语句 / CREATE TABLE 语句。该方法已经被 deprecate，推荐使用 execute_sql 或者 create_statement_set 替代。

 \|
\| 

create_statement_set

 \| 

用来执行多条 SQL 语句，可以通过该方法编写 multi-sink 的作业。

 \|
\| 

execute_sql

 \| 

用来执行单条 SQL 语句。execute_sql VS create_statement_set: 前者只能执行单条 SQL 语句，后者可用于执行多条 SQL 语句 execute_sql VS sql_query：前者可用于执行各种类型的 SQL 语句，比如 DDL、 DML、DQL、SHOW、DESCRIBE、EXPLAIN、USE 等，后者只能执行 DQL 语句即使是 DQL 语句，两者的行为也不一样。前者会生成 Flink 作业，触发表数据的计算，返回 TableResult 类型，后者并不触发计算，仅对表进行逻辑变换，返回 Table 类型

 \|

■ **5）查看执行计划**  

用户在开发或者调试作业的过程中，可能需要查看作业的执行计划，可以通过如下方式。

**方式一：Table.explain**

比如，当我们需要知道 transformed_tab 当前的执行计划时，可以执行：print(transformed_tab.explain())，得到如下输出：

```sql
== Abstract Syntax Tree ==
LogicalProject(EXPR$0=[sub_string($0, 2, 4)])
+- LogicalTableScan(table=[[default_catalog, default_database, Unregistered_TableSource_582508460, source: [PythonInputFormatTableSource(a)]]])

== Optimized Logical Plan ==
PythonCalc(select=[sub_string(a, 2, 4) AS EXPR$0])
+- LegacyTableSourceScan(table=[[default_catalog, default_database, Unregistered_TableSource_582508460, source: [PythonInputFormatTableSource(a)]]], fields=[a])

== Physical Execution Plan ==
Stage 1 : Data Source
    content : Source: PythonInputFormatTableSource(a)

    Stage 2 : Operator
        content : SourceConversion(table=[default_catalog.default_database.Unregistered_TableSource_582508460, source: [PythonInputFormatTableSource(a)]], fields=[a])
        ship_strategy : FORWARD

        Stage 3 : Operator
            content : StreamExecPythonCalc
            ship_strategy : FORWARD
```

**方式二：TableEnvironment.explain_sql**

方式一适用于查看某一个 table 的执行计划，有时候并没有一个现成的 table 对象可用，比如：

```http
print(t_env.explain_sql("INSERT INTO my_sink SELECT * FROM %s " % transformed_tab))
```

其执行计划如下所示：  

```sql
== Abstract Syntax Tree ==
LogicalSink(table=[default_catalog.default_database.my_sink], fields=[EXPR$0])
+- LogicalProject(EXPR$0=[sub_string($0, 2, 4)])
   +- LogicalTableScan(table=[[default_catalog, default_database, Unregistered_TableSource_1143388267, source: [PythonInputFormatTableSource(a)]]])

== Optimized Logical Plan ==
Sink(table=[default_catalog.default_database.my_sink], fields=[EXPR$0])
+- PythonCalc(select=[sub_string(a, 2, 4) AS EXPR$0])
   +- LegacyTableSourceScan(table=[[default_catalog, default_database, Unregistered_TableSource_1143388267, source: [PythonInputFormatTableSource(a)]]], fields=[a])

== Physical Execution Plan ==
Stage 1 : Data Source
    content : Source: PythonInputFormatTableSource(a)

    Stage 2 : Operator
        content : SourceConversion(table=[default_catalog.default_database.Unregistered_TableSource_1143388267, source: [PythonInputFormatTableSource(a)]], fields=[a])
        ship_strategy : FORWARD

        Stage 3 : Operator
            content : StreamExecPythonCalc
            ship_strategy : FORWARD

            Stage 4 : Data Sink
                content : Sink: Sink(table=[default_catalog.default_database.my_sink], fields=[EXPR$0])
                ship_strategy : FORWARD
```

#### ■ **6）写出结果数据**

**方式一：通过 DDL**

和创建数据源表类似，也可以通过 DDL 的方式来创建结果表。

```python
t_env.execute_sql("""
        CREATE TABLE my_sink (
          `sum` VARCHAR
        ) WITH (
          'connector' = 'print'
        )
    """)

table_result = transformed_tab.execute_insert('my_sink')
```

说明：

-   当使用 print 作为 sink 时，作业结果会打印到标准输出中。如果不需要查看输出，也可以使用 blackhole 作为 sink。

**方式二：collect**

也可以通过 collect 方法，将 table 的结果收集到客户端，并逐条查看。

```http
table_result = transformed_tab.execute()
with table_result.collect() as results:
    for result in results:
        print(result)
```

说明：

-   该方式可以方便地将 table 的结果收集到客户端并查看。
-   由于数据最终会收集到客户端，所以最好限制一下数据条数，比如：

transformed_tab.limit(10).execute()，限制只收集 10 条数据到客户端。

**方式三：to_pandas**

也可以通过 to_pandas 方法，将 table 的结果转换成 pandas.DataFrame 并查看。

```makefile
result = transformed_tab.to_pandas()
print(result)
```

可以看到如下输出：

```http
  _c0
0  32
1  e6
2  8b
3  be
4  4f
5  b4
6  a6
7  49
8  35
9  6b
```

说明：

-   该方式与 collect 类似，也会将 table 的结果收集到客户端，所以最好限制一下结果数据的条数。

#### **■ 7）总结**

完整的作业示例如下：

```http
from pyflink.table import DataTypes, EnvironmentSettings, StreamTableEnvironment
from pyflink.table.expressions import col
from pyflink.table.udf import udf


def table_api_demo():
    env_settings = EnvironmentSettings.new_instance().in_streaming_mode().use_blink_planner().build()
    t_env = StreamTableEnvironment.create(environment_settings=env_settings)
    t_env.get_config().get_configuration().set_string('parallelism.default', '4')

    t_env.execute_sql("""
            CREATE TABLE my_source (
              a VARCHAR,
              b VARCHAR
            ) WITH (
              'connector' = 'datagen',
              'number-of-rows' = '10'
            )
        """)

    tab = t_env.from_path('my_source')

    @udf(result_type=DataTypes.STRING())
    def sub_string(s: str, begin: int, end: int):
        return s[begin:end]

    transformed_tab = tab.select(sub_string(col('a'), 2, 4))

    t_env.execute_sql("""
            CREATE TABLE my_sink (
              `sum` VARCHAR
            ) WITH (
              'connector' = 'print'
            )
        """)

    table_result = transformed_tab.execute_insert('my_sink')

    # 1）等待作业执行结束，用于local执行，否则可能作业尚未执行结束，该脚本已退出，会导致minicluster过早退出
    # 2）当作业通过detach模式往remote集群提交时，比如YARN/Standalone/K8s等，需要移除该方法
    table_result.wait()


if __name__ == '__main__':
    table_api_demo()
```

执行结果如下：

```shell
4> +I(a1)
3> +I(b0)
2> +I(b1)
1> +I(37)
3> +I(74)
4> +I(3d)
1> +I(07)
2> +I(f4)
1> +I(7f)
2> +I(da)
```

### **PyFlink DataStream API 作业**

■ **1）创建 StreamExecutionEnvironment 对象**

对于 DataStream API 作业来说，用户首先需要定义一个 StreamExecutionEnvironment 对象。

```ini
env = StreamExecutionEnvironment.get_execution_environment()
```

■ **2）配置作业的执行参数**

可以通过以下方式，配置作业的执行参数。以下示例将作业的默认并发度设置为 4。

```css
env.set_parallelism(4)
```

■ **3）创建数据源**

接下来，需要为作业创建一个数据源。PyFlink 中提供了多种方式来定义数据源。

**方式一：from_collection**

PyFlink 支持用户从一个列表创建源表。以下示例定义了包含了 3 行数据的表：\[(1, 'aaa|bb'), (2, 'bb|a'), (3, 'aaa|a')]，该表有 2 列，列名分别为 a 和 b，类型分别为 VARCHAR 和 BIGINT。

```nginx
ds = env.from_collection(
        collection=[(1, 'aaa|bb'), (2, 'bb|a'), (3, 'aaa|a')],
        type_info=Types.ROW([Types.INT(), Types.STRING()]))
```

说明：

-   这种方式通常用于测试阶段，可以方便地创建一个数据源。
-   from_collection 方法可以接收两个参数，其中第一个参数用于指定数据列表；第二个参数用于指定数据的类型。

**方式二：使用 PyFlink DataStream API 中定义的 connector**

此外，也可以使用 PyFlink DataStream API 中已经支持的 connector，需要注意的是，1.12 中仅提供了 Kafka connector 的支持。

```properties
deserialization_schema = JsonRowDeserializationSchema.builder() \
    .type_info(type_info=Types.ROW([Types.INT(), Types.STRING()])).build()

kafka_consumer = FlinkKafkaConsumer(
    topics='test_source_topic',
    deserialization_schema=deserialization_schema,
    properties={'bootstrap.servers': 'localhost:9092', 'group.id': 'test_group'})

ds = env.add_source(kafka_consumer)
```

说明：

-   Kafka connector 当前没有包含在 Flink 官方提供的发行包中，如果需要在 PyFlink 作业中使用，用户需要显式地指定相应 FAT JAR \[2]，JAR 包可以通过如下方式指定：

```apache
# 注意：file:///前缀不能省略
env.add_jars("file:///my/jar/path/flink-sql-connector-kafka_2.11-1.12.0.jar")
```

-   即使是 PyFlink DataStream API 作业，也推荐使用 Table & SQL connector 中打包出来的 FAT JAR，可以避免递归依赖的问题。

**方式三：使用 PyFlink Table API 中定义的 connector**

以下示例定义了如何将 Table & SQL 中支持的 connector 用于 PyFlink DataStream API 作业。

```http
t_env = StreamTableEnvironment.create(stream_execution_environment=env)

t_env.execute_sql("""
        CREATE TABLE my_source (
          a INT,
          b VARCHAR
        ) WITH (
          'connector' = 'datagen',
          'number-of-rows' = '10'
        )
    """)

ds = t_env.to_append_stream(
    t_env.from_path('my_source'),
    Types.ROW([Types.INT(), Types.STRING()]))
```

说明：

-   由于当前 PyFlink DataStream API 中 built-in 支持的 connector 种类还比较少，推荐通过这种方式来创建 PyFlink DataStream API 作业中使用的数据源表，这样的话，所有 PyFlink Table API 中可以使用的 connector，都可以在 PyFlink DataStream API 作业中使用。


-   需要注意的是，TableEnvironment 需要通过以下方式创建 StreamTableEnvironment.create(stream_execution_environment=env)，以使得 PyFlink DataStream API 与 PyFlink Table API 共享同一个 StreamExecutionEnvironment 对象。

■ **4）定义计算逻辑**

生成数据源对应的 DataStream 对象之后，接下来就可以使用 PyFlink DataStream API 中定义的各种操作，定义计算逻辑，对 DataStream 对象进行变换了，比如：

```python
def split(s):
    splits = s[1].split("|")
    for sp in splits:
       yield s[0], sp

ds = ds.map(lambda i: (i[0] + 1, i[1])) \
       .flat_map(split) \
       .key_by(lambda i: i[1]) \
       .reduce(lambda i, j: (i[0] + j[0], i[1]))
```

■ **5）写出结果数据**

**方式一：print**

可以调用 DataStream 对象上的 print 方法，将 DataStream 的结果打印到标准输出中，比如：

```css
ds.print()
```

**方式二：使用 PyFlink DataStream API 中定义的 connector**

可以直接使用 PyFlink DataStream API 中已经支持的 connector，需要注意的是，1.12 中提供了对于 FileSystem、JDBC、Kafka connector 的支持，以 Kafka 为例：

```http
serialization_schema = JsonRowSerializationSchema.builder() \
    .with_type_info(type_info=Types.ROW([Types.INT(), Types.STRING()])).build()

kafka_producer = FlinkKafkaProducer(
    topic='test_sink_topic',
    serialization_schema=serialization_schema,
    producer_config={'bootstrap.servers': 'localhost:9092', 'group.id': 'test_group'})

ds.add_sink(kafka_producer)
```

说明：

-   JDBC、Kafka connector 当前没有包含在 Flink 官方提供的发行包中，如果需要在 PyFlink 作业中使用，用户需要显式地指定相应 FAT JAR，比如 Kafka connector 可以使用 JAR 包 \[2]，JAR 包可以通过如下方式指定：

```apache
# 注意：file:///前缀不能省略
env.add_jars("file:///my/jar/path/flink-sql-connector-kafka_2.11-1.12.0.jar")
```

-   推荐使用 Table & SQL connector 中打包出来的 FAT JAR，可以避免递归依赖的问题。

**方式三：使用 PyFlink Table API 中定义的 connector**

以下示例展示了如何将 Table & SQL 中支持的 connector，用作 PyFlink DataStream API 作业的 sink。

```python
# 写法一：ds类型为Types.ROW
def split(s):
    splits = s[1].split("|")
    for sp in splits:
        yield Row(s[0], sp)

ds = ds.map(lambda i: (i[0] + 1, i[1])) \
       .flat_map(split, Types.ROW([Types.INT(), Types.STRING()])) \
       .key_by(lambda i: i[1]) \
       .reduce(lambda i, j: Row(i[0] + j[0], i[1]))

# 写法二：ds类型为Types.TUPLE
def split(s):
    splits = s[1].split("|")
    for sp in splits:
        yield s[0], sp

ds = ds.map(lambda i: (i[0] + 1, i[1])) \
       .flat_map(split, Types.TUPLE([Types.INT(), Types.STRING()])) \
       .key_by(lambda i: i[1]) \
       .reduce(lambda i, j: (i[0] + j[0], i[1]))

# 将ds写出到sink
t_env.execute_sql("""
        CREATE TABLE my_sink (
          a INT,
          b VARCHAR
        ) WITH (
          'connector' = 'print'
        )
    """)

table = t_env.from_data_stream(ds)
table_result = table.execute_insert("my_sink")
```

说明：

-   需要注意的是，t_env.from_data_stream(ds) 中的 ds 对象的 result type 类型必须是复合类型 Types.ROW 或者 Types.TUPLE，这也就是为什么需要显式声明作业计算逻辑中 flat_map 操作的 result 类型


-   作业的提交，需要通过 PyFlink Table API 中提供的作业提交方式进行提交


-   由于当前 PyFlink DataStream API 中支持的 connector 种类还比较少，推荐通过这种方式来定义 PyFlink DataStream API 作业中使用的数据源表，这样的话，所有 PyFlink Table API 中可以使用的 connector，都可以作为 PyFlink DataStream API 作业的 sink。

■ **7）总结**

完整的作业示例如下：

**方式一（适合调试）：** 

```python
from pyflink.common.typeinfo import Types
from pyflink.datastream import StreamExecutionEnvironment


def data_stream_api_demo():
    env = StreamExecutionEnvironment.get_execution_environment()
    env.set_parallelism(4)

    ds = env.from_collection(
        collection=[(1, 'aaa|bb'), (2, 'bb|a'), (3, 'aaa|a')],
        type_info=Types.ROW([Types.INT(), Types.STRING()]))

    def split(s):
        splits = s[1].split("|")
        for sp in splits:
            yield s[0], sp

    ds = ds.map(lambda i: (i[0] + 1, i[1])) \
           .flat_map(split) \
           .key_by(lambda i: i[1]) \
           .reduce(lambda i, j: (i[0] + j[0], i[1]))

    ds.print()

    env.execute()


if __name__ == '__main__':
    data_stream_api_demo()
```

执行结果如下：

```shell
3> (2, 'aaa')
3> (2, 'bb')
3> (6, 'aaa')
3> (4, 'a')
3> (5, 'bb')
3> (7, 'a')
```

方式二（适合线上作业）：

```python
from pyflink.common.typeinfo import Types
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.table import StreamTableEnvironment


def data_stream_api_demo():
    env = StreamExecutionEnvironment.get_execution_environment()
    t_env = StreamTableEnvironment.create(stream_execution_environment=env)
    env.set_parallelism(4)

    t_env.execute_sql("""
            CREATE TABLE my_source (
              a INT,
              b VARCHAR
            ) WITH (
              'connector' = 'datagen',
              'number-of-rows' = '10'
            )
        """)

    ds = t_env.to_append_stream(
        t_env.from_path('my_source'),
        Types.ROW([Types.INT(), Types.STRING()]))

    def split(s):
        splits = s[1].split("|")
        for sp in splits:
            yield s[0], sp

    ds = ds.map(lambda i: (i[0] + 1, i[1])) \
           .flat_map(split, Types.TUPLE([Types.INT(), Types.STRING()])) \
           .key_by(lambda i: i[1]) \
           .reduce(lambda i, j: (i[0] + j[0], i[1]))

    t_env.execute_sql("""
            CREATE TABLE my_sink (
              a INT,
              b VARCHAR
            ) WITH (
              'connector' = 'print'
            )
        """)

    table = t_env.from_data_stream(ds)
    table_result = table.execute_insert("my_sink")

    # 1）等待作业执行结束，用于local执行，否则可能作业尚未执行结束，该脚本已退出，会导致minicluster过早退出
    # 2）当作业通过detach模式往remote集群提交时，比如YARN/Standalone/K8s等，需要移除该方法
    table_result.wait()


if __name__ == '__main__':
    data_stream_api_demo()
```

**三、作业提交**

Flink 提供了多种作业部署方式，比如 local、standalone、YARN、K8s 等，PyFlink 也支持上述作业部署方式，请参考 Flink 官方文档 \[3]，了解更多详细信息。  

* * *

### **local**

说明：使用该方式执行作业时，会启动一个 minicluster，作业会提交到 minicluster 中执行，该方式适合作业开发阶段。

示例：python3 table_api_demo.py

### **standalone**

说明：使用该方式执行作业时，作业会提交到一个远端的 standalone 集群。

示例：

./bin/flink run --jobmanager localhost:8081 --python table_api_demo.py

### **YARN Per-Job**

说明：使用该方式执行作业时，作业会提交到一个远端的 YARN 集群。

示例：

./bin/flink run --target yarn-per-job --python table_api_demo.py

### **K8s application mode**

说明：使用该方式执行作业时，作业会提交到 K8s 集群，以 application mode 的方式执行。

示例：

./bin/flink run-application \\    --target kubernetes-application \\    --parallelism 8 \\    -Dkubernetes.cluster-id=<ClusterId> \\    -Dtaskmanager.memory.process.size=4096m \\    -Dkubernetes.taskmanager.cpu=2 \\    -Dtaskmanager.numberOfTaskSlots=4 \\    -Dkubernetes.container.image=<PyFlinkImageName> \\

\--pyModule table_api_demo \\    --pyFiles file:///path/to/table_api_demo.py

### **参数说明**

除了上面提到的参数之外，通过 flink run 提交的时候，还有其它一些和 PyFlink 作业相关的参数。

| 参数名 | 用途描述 | 示例  |
| --- | ---- | --- |

\| 

\-py / --python

 \| 

指定作业的入口文件

 \| 

\-py file:///path/to/table_api_demo.py

 \|
\| 

\-pym / --pyModule

 \| 

指定作业的 entry module，功能和 --python 类似，可用于当作业的 Python 文件为 zip 包，无法通过 --python 指定时，相比 --python 来说，更通用

 \| 

\-pym table_api_demo -pyfs file:///path/to/table_api_demo.py

 \|
\| 

\-pyfs / --pyFiles

 \| 

指定一个到多个 Python 文件（.py/.zip 等，逗号分割），这些 Python 文件在作业执行的时候，会放到 Python 进程的 PYTHONPATH 中，可以在 Python 自定义函数中访问到

 \| 

\-pyfs file:///path/to/table_api_demo.py,file:///path/to/deps.zip

 \|
\| 

\-pyarch / --pyArchives

 \| 

指定一个到多个存档文件（逗号分割），这些存档文件，在作业执行的时候，会被解压之后，放到 Python 进程的 workspace 目录，可以通过相对路径的方式进行访问

 \| 

\-pyarch file:///path/to/venv.zip

 \|
\| 

\-pyexec / --pyExecutable

 \| 

指定作业执行的时候，Python 进程的路径

 \| 

\-pyarch file:///path/to/venv.zip -pyexec venv.zip/venv/bin/python3

 \|
\| 

\-pyreq / --pyRequirements

 \| 

指定 requirements 文件，requirements 文件中定义了作业的依赖

 \| 

\-pyreq requirements.txt

 \|

**四、问题排查**

当我们刚刚上手 PyFlink 作业开发的时候，难免会遇到各种各样的问题，学会如何排查问题是非常重要的。接下来，我们介绍一些常见的问题排查手段。

### **client 端异常输出**

PyFlink 作业也遵循 Flink 作业的提交方式，作业首先会在 client 端编译成 JobGraph，然后提交到 Flink 集群执行。如果作业编译有问题，会导致在 client 端提交作业的时候就抛出异常，此时可以在 client 端看到类似这样的输出：

```sql
Traceback (most recent call last):
  File "/Users/dianfu/code/src/github/pyflink-usecases/datastream_api_demo.py", line 50, in <module>
    data_stream_api_demo()
  File "/Users/dianfu/code/src/github/pyflink-usecases/datastream_api_demo.py", line 45, in data_stream_api_demo
    table_result = table.execute_insert("my_")
  File "/Users/dianfu/venv/pyflink-usecases/lib/python3.8/site-packages/pyflink/table/table.py", line 864, in execute_insert
    return TableResult(self._j_table.executeInsert(table_path, overwrite))
  File "/Users/dianfu/venv/pyflink-usecases/lib/python3.8/site-packages/py4j/java_gateway.py", line 1285, in __call__
    return_value = get_return_value(
  File "/Users/dianfu/venv/pyflink-usecases/lib/python3.8/site-packages/pyflink/util/exceptions.py", line 162, in deco
    raise java_exception
pyflink.util.exceptions.TableException: Sink `default_catalog`.`default_database`.`my_` does not exists
     at org.apache.flink.table.planner.delegation.PlannerBase.translateToRel(PlannerBase.scala:247)
     at org.apache.flink.table.planner.delegation.PlannerBase$$anonfun$1.apply(PlannerBase.scala:159)
     at org.apache.flink.table.planner.delegation.PlannerBase$$anonfun$1.apply(PlannerBase.scala:159)
     at scala.collection.TraversableLike$$anonfun$map$1.apply(TraversableLike.scala:234)
     at scala.collection.TraversableLike$$anonfun$map$1.apply(TraversableLike.scala:234)
     at scala.collection.Iterator$class.foreach(Iterator.scala:891)
     at scala.collection.AbstractIterator.foreach(Iterator.scala:1334)
     at scala.collection.IterableLike$class.foreach(IterableLike.scala:72)
     at scala.collection.AbstractIterable.foreach(Iterable.scala:54)
     at scala.collection.TraversableLike$class.map(TraversableLike.scala:234)
     at scala.collection.AbstractTraversable.map(Traversable.scala:104)
     at org.apache.flink.table.planner.delegation.PlannerBase.translate(PlannerBase.scala:159)
     at org.apache.flink.table.api.internal.TableEnvironmentImpl.translate(TableEnvironmentImpl.java:1329)
     at org.apache.flink.table.api.internal.TableEnvironmentImpl.executeInternal(TableEnvironmentImpl.java:676)
     at org.apache.flink.table.api.internal.TableImpl.executeInsert(TableImpl.java:572)
     at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
     at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
     at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
     at java.lang.reflect.Method.invoke(Method.java:498)
     at org.apache.flink.api.python.shaded.py4j.reflection.MethodInvoker.invoke(MethodInvoker.java:244)
     at org.apache.flink.api.python.shaded.py4j.reflection.ReflectionEngine.invoke(ReflectionEngine.java:357)
     at org.apache.flink.api.python.shaded.py4j.Gateway.invoke(Gateway.java:282)
     at org.apache.flink.api.python.shaded.py4j.commands.AbstractCommand.invokeMethod(AbstractCommand.java:132)
     at org.apache.flink.api.python.shaded.py4j.commands.CallCommand.execute(CallCommand.java:79)
     at org.apache.flink.api.python.shaded.py4j.GatewayConnection.run(GatewayConnection.java:238)
     at java.lang.Thread.run(Thread.java:748)

Process finished with exit code 1
```

比如上述报错说明作业中使用的名字为 "my\_" 的表不存在。

### **TaskManager 日志文件**

有些错误直到作业运行的过程中才会发生，比如脏数据或者 Python 自定义函数的实现问题等，针对这种错误，通常需要查看 TaskManager 的日志文件，比如以下错误反映用户在 Python 自定义函数中访问的 opencv 库不存在。

```sql
Caused by: java.lang.RuntimeException: Error received from SDK harness for instruction 2: Traceback (most recent call last):
  File "/Users/dianfu/venv/pyflink-usecases/lib/python3.8/site-packages/apache_beam/runners/worker/sdk_worker.py", line 253, in _execute
    response = task()
  File "/Users/dianfu/venv/pyflink-usecases/lib/python3.8/site-packages/apache_beam/runners/worker/sdk_worker.py", line 310, in <lambda>
    lambda: self.create_worker().do_instruction(request), request)
  File "/Users/dianfu/venv/pyflink-usecases/lib/python3.8/site-packages/apache_beam/runners/worker/sdk_worker.py", line 479, in do_instruction
    return getattr(self, request_type)(
  File "/Users/dianfu/venv/pyflink-usecases/lib/python3.8/site-packages/apache_beam/runners/worker/sdk_worker.py", line 515, in process_bundle
    bundle_processor.process_bundle(instruction_id))
  File "/Users/dianfu/venv/pyflink-usecases/lib/python3.8/site-packages/apache_beam/runners/worker/bundle_processor.py", line 977, in process_bundle
    input_op_by_transform_id[element.transform_id].process_encoded(
  File "/Users/dianfu/venv/pyflink-usecases/lib/python3.8/site-packages/apache_beam/runners/worker/bundle_processor.py", line 218, in process_encoded
    self.output(decoded_value)
  File "apache_beam/runners/worker/operations.py", line 330, in apache_beam.runners.worker.operations.Operation.output
  File "apache_beam/runners/worker/operations.py", line 332, in apache_beam.runners.worker.operations.Operation.output
  File "apache_beam/runners/worker/operations.py", line 195, in apache_beam.runners.worker.operations.SingletonConsumerSet.receive
  File "pyflink/fn_execution/beam/beam_operations_fast.pyx", line 71, in pyflink.fn_execution.beam.beam_operations_fast.FunctionOperation.process
  File "pyflink/fn_execution/beam/beam_operations_fast.pyx", line 85, in pyflink.fn_execution.beam.beam_operations_fast.FunctionOperation.process
  File "pyflink/fn_execution/coder_impl_fast.pyx", line 83, in pyflink.fn_execution.coder_impl_fast.DataStreamFlatMapCoderImpl.encode_to_stream
  File "/Users/dianfu/code/src/github/pyflink-usecases/datastream_api_demo.py", line 26, in split
    import cv2
ModuleNotFoundError: No module named 'cv2'

    at org.apache.beam.runners.fnexecution.control.FnApiControlClient$ResponseStreamObserver.onNext(FnApiControlClient.java:177)
    at org.apache.beam.runners.fnexecution.control.FnApiControlClient$ResponseStreamObserver.onNext(FnApiControlClient.java:157)
    at org.apache.beam.vendor.grpc.v1p26p0.io.grpc.stub.ServerCalls$StreamingServerCallHandler$StreamingServerCallListener.onMessage(ServerCalls.java:251)
    at org.apache.beam.vendor.grpc.v1p26p0.io.grpc.ForwardingServerCallListener.onMessage(ForwardingServerCallListener.java:33)
    at org.apache.beam.vendor.grpc.v1p26p0.io.grpc.Contexts$ContextualizedServerCallListener.onMessage(Contexts.java:76)
    at org.apache.beam.vendor.grpc.v1p26p0.io.grpc.internal.ServerCallImpl$ServerStreamListenerImpl.messagesAvailableInternal(ServerCallImpl.java:309)
    at org.apache.beam.vendor.grpc.v1p26p0.io.grpc.internal.ServerCallImpl$ServerStreamListenerImpl.messagesAvailable(ServerCallImpl.java:292)
    at org.apache.beam.vendor.grpc.v1p26p0.io.grpc.internal.ServerImpl$JumpToApplicationThreadServerStreamListener$1MessagesAvailable.runInContext(ServerImpl.java:782)
    at org.apache.beam.vendor.grpc.v1p26p0.io.grpc.internal.ContextRunnable.run(ContextRunnable.java:37)
    at org.apache.beam.vendor.grpc.v1p26p0.io.grpc.internal.SerializingExecutor.run(SerializingExecutor.java:123)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
    ... 1 more
```

说明：

-   local 模式下，TaskManager 的 log 位于 PyFlink 的安装目录下：site-packages/pyflink/log/，也可以通过如下命令找到：

    > > \> import pyflink 

    > > \> print(pyflink.path)

    \['/Users/dianfu/venv/pyflink-usecases/lib/python3.8/site-packages/pyflink']，则 log 文件位于 / Users/dianfu/venv/pyflink-usecases/lib/python3.8/site-packages/pyflink/log 目录下

### **自定义日志**

有时候，异常日志的内容并不足以帮助我们定位问题，此时可以考虑在 Python 自定义函数中打印一些日志信息。PyFlink 支持用户在 Python 自定义函数中通过 logging 的方式输出 log，比如：

```python
def split(s):
    import logging
    logging.info("s: " + str(s))
    splits = s[1].split("|")
    for sp in splits:
        yield s[0], sp
```

通过上述方式，split 函数的输入参数，会打印到 TaskManager 的日志文件中。

### **远程调试**

PyFlink 作业，在运行过程中，会启动一个独立的 Python 进程执行 Python 自定义函数，所以如果需要调试 Python 自定义函数，需要通过远程调试的方式进行，可以参见\[4]，了解如何在 Pycharm 中进行 Python 远程调试。

1）在 Python 环境中安装 pydevd-pycharm：

pip install pydevd-pycharm~=203.7717.65

2）在 Python 自定义函数中设置远程调试参数：

```python
def split(s):
    import pydevd_pycharm
    pydevd_pycharm.settrace('localhost', port=6789, stdoutToServer=True, stderrToServer=True)
    splits = s[1].split("|")
    for sp in splits:
        yield s[0], sp
```

3）按照 Pycharm 中远程调试的步骤，进行操作即可，可以参见\[4]，也可以参考博客\[5]中 “代码调试” 部分的介绍。

说明：Python 远程调试功能只在 Pycharm 的 professional 版才支持。

### **社区用户邮件列表**

如果通过以上步骤之后，问题还未解决，也可以订阅 Flink 用户邮件列表 \[6]，将问题发送到 Flink 用户邮件列表。需要注意的是，将问题发送到邮件列表时，尽量将问题描述清楚，最好有可复现的代码及数据，可以参考一下这个邮件\[7]。

### **钉钉群**

此外，也欢迎大家加入 “PyFlink 交流群”，交流 PyFlink 相关的问题。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2017-38-20/9d26d836-15c4-466e-bbb8-b269430633d4.png?raw=true)

**五、总结**

在这篇文章中，我们主要介绍了 PyFlink API 作业的环境准备、作业开发、作业提交、问题排查等方面的信息，希望可以帮助用户使用 Python 语言快速构建一个 Flink 作业，希望对大家有所帮助。接下来，我们会继续推出 PyFlink 系列文章，帮助 PyFlink 用户深入了解 PyFlink 中各种功能、应用场景、最佳实践等。

另外，我们推出一个调查问卷，希望大家积极参与这个问卷，帮助我们更好的去整理 PyFlink 相关学习资料。填完问卷后即可参与抽奖 Flink 定制款 Polo 衫，4 月 30 日中午 12:00 准时开奖。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2017-38-20/955fb4b4-9ec6-4b39-a064-cb6a2efb1fe9.png?raw=true)

## 引用链接

\[1] [https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/table/connectors/](https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/table/connectors/)

\[2] [https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-connector-kafka\\\_2.11/1.12.0/flink-sql-connector-kafka\\\_2.11-1.12.0.jar](https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-connector-kafka\_2.11/1.12.0/flink-sql-connector-kafka\_2.11-1.12.0.jar)

\[3] [https://ci.apache.org/projects/flink/flink-docs-release-1.12/deployment/cli.html#submitting-pyflink-jobs](https://ci.apache.org/projects/flink/flink-docs-release-1.12/deployment/cli.html#submitting-pyflink-jobs)

\[4] [https://www.jetbrains.com/help/pycharm/remote-debugging-with-product.html#remote-debug-config](https://www.jetbrains.com/help/pycharm/remote-debugging-with-product.html#remote-debug-config)

\[5] [https://mp.weixin.qq.com/s?\_\_biz=MzIzMDMwNTg3MA==&mid=2247485386&idx=1&sn=da24e5200d72e0627717494c22d0372e&chksm=e8b43eebdfc3b7fdbd10b49e6749cb761b7aa5f8ddc90b34eb3170119a8bbb3ddd7327acb712&scene=178&cur_album_id=1386152464113811456#rd](https://mp.weixin.qq.com/s?__biz=MzIzMDMwNTg3MA==&mid=2247485386&idx=1&sn=da24e5200d72e0627717494c22d0372e&chksm=e8b43eebdfc3b7fdbd10b49e6749cb761b7aa5f8ddc90b34eb3170119a8bbb3ddd7327acb712&scene=21&cur_album_id=1386152464113811456#wechat_redirect)

\[6] [https://flink.apache.org/community.html#mailing-lists](https://flink.apache.org/community.html#mailing-lists)

\[7] [http://apache-flink-user-mailing-list-archive.2336050.n4.nabble.com/PyFlink-called-already-closed-and-NullPointerException-td42997.html](http://apache-flink-user-mailing-list-archive.2336050.n4.nabble.com/PyFlink-called-already-closed-and-NullPointerException-td42997.html)

更多 Flink 相关技术问题，可扫码加入社区钉钉交流群～

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2017-38-20/8aa2eeb6-7cd0-4db9-94ec-6b88e47aae0f.png?raw=true)

* * *

▼ 关注「**Flink 中文社区**」，获取更多技术干货 ▼  

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2017-38-20/01a157ea-91f4-42ed-b016-e006014ecff0.jpeg?raw=true)

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2017-38-20/113be599-db75-487a-a743-b674b5e3fdc0.gif?raw=true)
 戳我，查看更多技术干货！
