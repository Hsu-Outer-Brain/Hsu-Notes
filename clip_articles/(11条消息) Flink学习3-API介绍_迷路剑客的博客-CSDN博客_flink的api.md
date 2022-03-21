# (11条消息) Flink学习3-API介绍_迷路剑客的博客-CSDN博客_flink的api
-   更多 Flink 系列文章请点击[Flink 系列文章](https://blog.csdn.net/baichoufei90/article/details/105674793)
-   更多大数据文章请点击[大数据好文推荐](https://blog.csdn.net/baichoufei90/article/details/90264272)

本文主要是介绍 Flink 的不同层次 (level)API 抽象，学习怎么通过 API 高效处理有状态性的计算无界和有界的数据流。

Flink 提供了三个不同层次的 API，每种 API 在简洁和易表达间有自己的权衡，适用于不同的场景：  
![](https://img-blog.csdn.net/20180928232805785?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

可以看到 Flink 一共有三个抽象层次的 API，目测应该前两个会用的比较多，他们更加简洁但是表达性比较差。下面自底向上分别简要介绍下这三个 API。

## 1.1 ProcessFunctions

看了上面的图我们知道`ProcessFunctions`最具表现力但是简洁性最差，是最底层的抽象 API，他被主要用来处理包含单独事件的一个或两个输入流或者是分组到一个窗口类的事件，所以提供了对时间和状态的细粒度控制。`ProcessFunctions`可强制修改 state、重注册未来某时触发回调函数的 timer，所以可以实现复杂事件处理逻辑，**这正适合很多有状态的事件驱动应用程序**。

因为最近作者调研主要涉及 FLink 流式 SQL API，这里没有详看，想要了解的请参见最后参考文档中给出的连接学习。

## 1.2 DataStream API

-   DataStream API 提供了若干常用的流 / 批处理操作，如窗口等。
-   有 Java 和 Scala 的 API 可选，都是依赖一些底层的基本方法如 map/aggregate 等实现的。

下面示例展示 session 化一个 click 流然后对每个 session 中的点击数计数：

```java

DataStream<Click> clicks = ...

DataStream<Tuple2<String, Long>> result = clicks
  
  .map(
    
    new MapFunction<Click, Tuple2<String, Long>>() {
      @Override
      public Tuple2<String, Long> map(Click click) {
        return Tuple2.of(click.userId, 1L);
      }
    })
  
  .keyBy(0)
  
  .window(EventTimeSessionWindows.withGap(Time.minutes(30L)))
  
  .reduce((a, b) -> Tuple2.of(a.f0, a.f1 + b.f1));

```

详见[第四章](#datastream)

## 1.3 SQL&Table API

见[第三章](#sql)

Flink 对常见的流式处理场景提供了若干内库，他们通常嵌入到 API 中，并非完全独立。 因此，他们可以从 API 的所有特性中受益，并与其他库集成：

## 2.1 Complex Event Processing (CEP)

该内库提供 API 来指定不同事件的模式，就像正则表达式或是状态机。模式识别是非常常见的事件流处理场景。

CEP 库的应用包括网络入侵检测，业务流程监控和欺诈检测。

## 2.2 DataSet API

DataSet API 是 Flink 的核心 API，用来应对批处理应用。

## 2.3 Gelly

Gelly 是一个可扩展的图形处理和分析库，他在 DataSet API 之上集成实现。

Gelly 具有内置算法，如标签传播，三角枚举和页面排名，但也提供了一个简化自定义图算法实现的 Graph API。

请点击[Flink 学习 4 - 流式 SQL](https://blog.csdn.net/baichoufei90/article/details/101054148)

## 4.1 处理时间设定

这里以 ProcessingTime 为例，分窗时间为 1 小时：

```java
val env = StreamExecutionEnvironment.getExecutionEnvironment

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime)





val stream: DataStream[MyEvent] = env.addSource(new FlinkKafkaConsumer09[MyEvent](topic, schema, props))

stream
    .keyBy( _.getUser )
    .timeWindow(Time.hours(1))
    .reduce( (a, b) => a.add(b) )
    .addSink(...)

```

`TimeCharacteristic`决定了 Source 怎么表现（比如是否分配 timestamp）以及时间窗口算子使用哪种时间作为计算标准。

如果使用 EventTIme，用户要么需要使用直接为数据定义 EventTIme 并自己发出 watermark 的 Source，要么在 Source 之后必须引入`Timestamp Assigner`来分配 timestamp 和`Watermark Generator`。

## 4.2 DataStream 转换

![](https://img-blog.csdnimg.cn/20200924224634819.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

## 4.3 Connector

### 4.3.1 KafkaConnector

请参考[Flink 学习 - DataStream-KafkaConnector](https://blog.csdn.net/baichoufei90/article/details/104009237)

### 4.3.2 HDFSConnector(StreamingFileSink)

请参考[Flink 学习 - DataStream-HDFSConnector(StreamingFileSink)](https://blog.csdn.net/baichoufei90/article/details/104009350)

## 4.4 分区策略

Operator 数据交换与分区策略  
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

## 4.5 DataStream Source 构建

-   fromElements\`\`\`java
    import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
    import org.apache.flink.streaming.api.datastream.DataStream;
    import org.apache.flink.api.common.functions.FilterFunction;

    public class Example {

        public static void main(String[] args) throws Exception {
            final StreamExecutionEnvironment env =
                    StreamExecutionEnvironment.getExecutionEnvironment();

            DataStream<Person> flintstones = env.fromElements(
                    new Person("Fred", 35),
                    new Person("Wilma", 35),
                    new Person("Pebbles", 2));

            DataStream<Person> adults = flintstones.filter(new FilterFunction<Person>() {
                @Override
                public boolean filter(Person person) throws Exception {
                    return person.age >= 18;
                }
            });

            adults.print();

            env.execute();
        }

        public static class Person {
            public String name;
            public Integer age;
            public Person() {};

            public Person(String name, Integer age) {
                this.name = name;
                this.age = age;
            };

            public String toString() {
                return this.name.toString() + ": age " + this.age.toString();
            };
        }

    }

    ```

    ```
-   fromCollection\`\`\`java
    List<Person> people = new ArrayList<Person>();

    people.add(new Person("Fred", 35));
    people.add(new Person("Wilma", 35));
    people.add(new Person("Pebbles", 2));

    DataStream<Person> flintstones = env.fromCollection(people);

    ```

    ```
-   socketTextStream\`\`\`java
    DataStream<String> lines = env.socketTextStream("localhost", 9999)

    ```

    ```
-   readTextFile\`\`\`java
    DataStream<String> lines = env.readTextFile("file:///path");

    ```

    ```
-   addSource(connector source)

## 4.6 DataStream Sink 构建

-   print  
    对结果流中每个元素代用`toString()`，将计算结果打印到 TM 日志中（如果是本地 ide，则打印到控制台）。

    输出看起来类似于

    ```
    1> Fred: age 35
    2> Wilma: age 35

    ```

    `1>` 和 `2>` 表示输出来自哪个 subtask 线程。
-   addSink(connector sink)  
    如 StreamingFileSink、JDBC 等

## 4.7 无状态算子

### 4.7.1 概述

这一节介绍的算子都是无状态的。也就是说新元素到来的计算和之前的元素毫无关系。

### 4.7.2 Map

输入元素的数量和输出元素数量一一对应，但可以改变元素类型。  
![](https://img-blog.csdnimg.cn/20200826164309153.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

### 4.7.3 FlatMap

输入一个元素，输出 0-N 个元素。可以改变元素类型。  
![](https://img-blog.csdnimg.cn/20200826164658641.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

### 4.7.4 Filter

每个元素调用该方法，返回一个 boolean。为 true 就保留，为 false 就忽略。  
![](https://img-blog.csdnimg.cn/2020082616535062.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

## 4.8 有状态算子

### 4.8.1 概述

有状态算子，输入元素计算会依赖早前来到的元素。状态可快照保存到 StateBackend。

### 4.8.2 keyBy

按照 key 的值 hash 后重分区到某个下游算子实例，可保证同 key 元素分配到相同分区。

DataStream 调用后，得到`KeyedStream`，需要使用 KeyedState。  
![](https://img-blog.csdnimg.cn/2020082617091075.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

每个 keyBy 会通过 hash shuffle 来为数据流进行重新分区。总体来说这个开销是很大的，它涉及网络通信、序列化和反序列化。

指定字段进行 keyBy:

```java
dataStream.keyBy("someKey") 
dataStream.keyBy(0) 

```

注意，以下类型不能作为 key

-   没有覆写`hashCode`方法的 POJO 类型
-   数组类型

以上直接指定字段为 key 的方式有个缺点就是无法推断 key 的字段类型，在 flink 内部会作为`Tuple`传递，比较难处理。所以也可以用一个合适的 KeySelector：

```java
rides
    .flatMap(new NYCEnrichment())
    .keyBy(
        new KeySelector<EnrichedRide, int>() {
            @Override
            public int getKey(EnrichedRide enrichedRide) throws Exception {
                return enrichedRide.startCell;
            }
        })

```

可使用 lambda 表达式简写

```java
rides
    .flatMap(new NYCEnrichment())
    .keyBy(enrichedRide -> enrichedRide.startCell)

```

当然，也可以不从 Event 中抽取 Key，可以直接通过[指定的计算](https://ci.apache.org/projects/flink/flink-docs-release-1.11/zh/learn-flink/etl.html#%E9%80%9A%E8%BF%87%E8%AE%A1%E7%AE%97%E5%BE%97%E5%88%B0%E9%94%AE)得到

```java
keyBy(ride -> GeoUtils.mapToGridCell(ride.startLon, ride.startLat))

```

### 4.8.3 Reduce

将当前元素和最后聚合的值进行聚合，发送得到的新值。  
![](https://img-blog.csdnimg.cn/20200826172722192.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

### 4.8.4 Fold

加入了初始值的概念，将新值与之前折叠的旧值进行折叠操作，输出结果  
![](https://img-blog.csdnimg.cn/20200826172859197.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

### 4.8.5 聚合算子

注意，加了`By`后缀的算子含义是返回元素，不加的是返回值。  
![](https://img-blog.csdnimg.cn/20200826173055158.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

### 4.8.6 Window

#### 4.8.6.1 Window

Window 算子可在已分区的 KeyedStream 上定义，会根据时间予以将每个 key 的数据分窗。  
![](https://img-blog.csdnimg.cn/2020082617443212.png#pic_center)

![](https://img-blog.csdnimg.cn/20201112152056671.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

#### 4.8.6.2 WindowAll

直接在未按 key 分区的 DataStream 上定义，按时间分窗。大多数情况下，是只有一个算子实例来处理所有的数据聚合。  
![](https://img-blog.csdnimg.cn/20200826175649105.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/20201112152117544.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

#### 4.8.6.3 Window Apply

将函数应用于整个窗口。如果使用 windowAll，则需配合传递`AllWindowFunction`。  
![](https://img-blog.csdnimg.cn/20200826175908707.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

#### 4.8.6.4 Window Reduce

![](https://img-blog.csdnimg.cn/20200826180041210.png#pic_center)

#### 4.8.6.5 Window Fold

![](https://img-blog.csdnimg.cn/20200826180049552.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

#### 4.8.6.6 Window 聚合算子

![](https://img-blog.csdnimg.cn/20200826180131525.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

#### 4.8.6.7 Evictor

可以为窗口计算前后做一些预处理工作，如驱逐元素。  
![](https://img-blog.csdnimg.cn/20201112155102707.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

#### 4.8.6.8 Trigger

-   WindowAssigner  
    窗口分配器接口，如`TumblingWindowAssigner`，用来给每个元素分配 0 个或多个（TumblingWindowAssigner 最多有一个）Window。
-   Pane  
    在 Window 算子中，当某个列 key 可用时就用 key 和所属的 Window 来分组。而拥有同样的 key 和 Window 的所有元素就被称为一个 Pane。

    一个被 WindowAssigner 分配到多个 Window 的元素可以在同个 Pane 中，这些 Pane 每个都拥有自己独立的一份 Trigger 实例。

    目前只有`SlidingWindowAssigner`实现了`PanedWindowAssigner`
-   Trigger  
    由触发器决定何时让某个 Pane 触发所拥有窗口来生产输出元素。

    需要注意的是 Trigger 不允许内部维护状态，因为他们可被不同的 key 重建或重用，而应该使用 TriggerContext 来持久化状态  
    ![](https://img-blog.csdnimg.cn/20201112155137781.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

## 4.9 多 DataStream 操作算子

### 4.9.1 Union

讲两个或更多的流合并，创建一个新的包含子流的所有元素的新流。  
![](https://img-blog.csdnimg.cn/20200826180850432.png#pic_center)

### 4.9.2 Window Join

将两个流 join，join key 由用户指定，还需要指定共用的时间窗口以及 join 操作函数  
![](https://img-blog.csdnimg.cn/20200826180951607.png#pic_center)

### 4.9.3 Interval Join

在给定时间间隔内按给定的 key join 来自两个流的元素。  
![](https://img-blog.csdnimg.cn/20200826181543742.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

### 4.9.4 Window CoGroup

将两个流组合，组合 key 由用户指定，还需要指定共用的时间窗口以及组合操作函数。

组合的含义是将拥有相同 key 的数据分组到一起。  
![](https://img-blog.csdnimg.cn/20200826181734182.png#pic_center)

### 4.9.5 Connect

连接两个流，并保留他们的数据类型  
![](https://img-blog.csdnimg.cn/20200826182320187.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

### 4.9.6 CoMap, CoFlatMap

类似单流的 Map 和 FlatMap，不过这里是用在`ConnectedStreams`上  
![](https://img-blog.csdnimg.cn/20200826182354228.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

### 4.9.7 Split

将一个流拆分为 2 个或更多流

以下例子按 value 的奇数或偶数拆分为了两个流  
![](https://img-blog.csdnimg.cn/20200826182446704.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

### 4.9.8 Select

从 SplitStream 中选取 1 个或多个流  
![](https://img-blog.csdnimg.cn/2020082618254215.png#pic_center)

### 4.9.9 Iterate

![](https://img-blog.csdnimg.cn/20200826182616912.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

## 4.10 [异步 IO](https://ci.apache.org/projects/flink/flink-docs-master/dev/stream/operators/asyncio.html)

### 4.10.1 概述

![](https://img-blog.csdnimg.cn/20201119175200143.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

同步 IO，比如 MapFunction 中访问外部数据库，则一个请求发送到数据库后必须等待收到返回，占据函数运行大量时间。

而如果使用异步 IO，则一个算子并行实例可在向外部发出请求的等待时间中继续发送其他请求、接收响应。这样可摊销多个请求的等待时间，可提高流式吞吐。

这比单独提升 MapFunction 算子的并行度带来的资源开销要小很多。

还可参考

-   [Flink 原理与实现：Aysnc I/O](http://wuchong.me/blog/2017/05/17/flink-internals-async-io/)

### 4.10.2 先决条件

需要连接外部存储客户端支持异步请求。

如果没有这样的客户端，则可以尝试通过创建多个并发客户端，从而将同步客户端改造为并发客户端，这样可以通过线程池处理多个同步请求，但这个方式效率通常低于原生异步客户端。

### 4.10.3 异步 IO API

#### 4.10.3.1 概述

Flink 异步 IO API 可配合异步请求客户端在 DataStream 中使用，该 API 处理与数据流的集成，同时还能处理好顺序、EventTime 和容错等。还需要做的：

-   实现分派请求的`AsyncFunction`接口
-   定义一个要传递给`ResultFuture`的回调函数，该回调函数会接受操作结果
-   在 DataStream 上应用该异步 IO 作为转换操作

官方示例：

```java



class AsyncDatabaseRequest extends RichAsyncFunction<String, Tuple2<String, String>> {

    
    private transient DatabaseClient client;

	
    @Override
    public void open(Configuration parameters) throws Exception {
        client = new DatabaseClient(host, post, credentials);
    }
	
	
    @Override
    public void close() throws Exception {
        client.close();
    }
	
	
    @Override
    public void asyncInvoke(String key, final ResultFuture<Tuple2<String, String>> resultFuture) throws Exception {

        
        final Future<String> result = client.query(key);

        
        
        CompletableFuture.supplyAsync(new Supplier<String>() {

            @Override
            public String get() {
                try {
                    return result.get();
                } catch (InterruptedException | ExecutionException e) {
                    
                    return null;
                }
            }
        }).thenAccept( (String dbResult) -> {
        	
            resultFuture.complete(Collections.singleton(new Tuple2<>(key, dbResult)));
        });
    }
}


val stream: DataStream[String] = ...


val resultStream: DataStream[(String, String)] =
    AsyncDataStream.unorderedWait(stream, new AsyncDatabaseRequest(), 1000, TimeUnit.MILLISECONDS, 100)

```

#### 4.10.3.2 参数

此外，还有两个参数控制异步行为：

-   timeout  
    每个异步请求被认为失败前的超时时间，防止失败、僵死的请求。
-   capacity  
    同时能进行的最大并发异步请求数量。尽管异步 IO 一般可带来上佳的吞吐，但这类带有异步 IO 的算子可能在流式应用中成为瓶颈，所以需要限制异步 IO 请求总数。一旦超过本限制，会触发背压。

#### 4.10.3.3 异步 IO 超时处理

默认会抛出异常，导致 Job 重启。

也可以自己实现`AsyncFunction#timeout`方法来自定义处理异常。

#### 4.10.3.4 实现原理

由`AsyncDataStream.(un)orderedWait` 创建一个 `AsyncWaitOperator`，该算子用来支持异步 IO，就会调用我们定义的`AsyncFunction`内的方法，并处理异步执行后得到的结果。  
![](https://img-blog.csdnimg.cn/20201121210651473.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

-   StreamElementQueue  
    进行中的请求对应的未完成的[Promise](http://docs.scala-lang.org/zh-cn/overviews/core/futures.html#promises)的队列。

    StreamElementQueue 有实现， OrderedStreamElementQueue 和 UnorderedStreamElementQueue，分别用于有序和无序两种场景。
-   Promise 是表示未来某个值的异步抽象。
-   Emitter  
    收到异步执行结果后发送消息给下游的一个线程。
-   执行流程

    1.  将进入该算子的元素（如`E5`）包装为 Promise `P5`
    2.  将 P5 放入 StreamElementQueue
    3.  调用用户定义的`AsyncFunction#asyncInvoke`方法发起异步请求，处理回调交给`ResultFuture`处理  
        该`asyncInvoke`方法会向用户自定义的外部服务发起一个异步的请求，并注册回调。该回调会在异步请求成功返回时调用`AsyncCollector.collect`方法将返回的结果交给 Flink 框架处理。

    实际上`AsyncCollector`是一个 Promise ，也就是上图的`P5`，在调用`collect`的时候会标记 Promise 为完成状态，并通知 Emitter 线程有完成的消息可以发送了，此时 Emiter 就会从队列中拉取 Promise 并发送到下游。

#### 4.10.3.5 结果顺序

`AsyncFunction`带来的结果顺序是未知的，具体依赖于请求完成的先后顺序。

要控制结果发送顺序，有两种模式：

#### 4.10.3.5.1 无序

-   概述  
    此时一旦收到结果就立刻发送，也就是说该顺序和异步请求顺序不同，使得流中元素记录在异步算子内发生变化。

    使用 DataStream API 时可用`AsyncDataStream.unorderedWait(...)`开启本模式。
-   Processing Time  
    当使用`processing time`时，本模式的特点是最低的延迟和开销。  
    ![](https://img-blog.csdnimg.cn/2020121121594540.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
    本模式下有两个队列。新来的元素包装为 Promise 后放入`uncompletedQueue`队列，只要得到结果返回便会被立刻放入`completedQueue`队列，并通知 Emitter 拉取消费并发送。此时因为没有水位，所以不需要 watermark 与消息的顺序性。
-   EventTime  
    异步 IO 算子可以正确处理 EventTime 中的水位。水位和记录保持同步，不会互相超过。只有连续两个 watermark 之间的记录是无序发出的。

    此时，在某个水位 A 后出现的每条记录都仅会在那个水位被发送后才会被发送。

    反过来，仅当所有一堆输入记录对应的结果数据发送后，才会发送这些输入记录之后的那个水位。  
    ![](https://img-blog.csdnimg.cn/20201211221737857.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

    因为有 watermark，需要协调 watermark 与消息之间的顺序性，所以跟使用 Processing Time 时无序相比，uncompletedQueue 中存放的每个元素从原先的 单个 Promise 变成了 Promise 的集合：

    -   新到来的普通元素被包装为 Promise 放入 uncompletedQueue 队尾的 Promise 集合中
    -   新到来的水位元素会被包装为 Promise，并构建一个 Promise 集合放入，最后放入 uncompletedQueue 队尾，并创建新的 Promise 集合以放入后续普通元素
    -   这样，水位元素就成了边界

    仅当 uncompletedQueue 队首集合中的某个 Promise 返回结果后才将该 Promise 移入`completedQueue`队列，再由 Emitter 处理消费并发往下游。

    仅当队首 Promise 集合中的所有 Promise 都处理完毕后，才会发送紧邻的水位，随后才能继续处理下一个 Promise 集合内的元素。这样就保证了当且仅当某个 watermark 之前所有的消息都已经被发送了，该 watermark 才能被发送。

#### 4.10.3.5.2 有序

保留流顺序，也就是说结果发送顺序和异步请求触发顺序（记录流入算子顺序）相同。所以会将结果记录缓存，直到他之前的记录结果都已经发送完毕或被判定为超时。

这种模式带来顺序的同时会增加大量额外延时和 checkpoint 开销，因为记录或结果会被放在 checkpoint 状态中相比于无序模式更长时间。

使用`AsyncDataStream.orderedWait(...)`开启本模式。  
![](https://img-blog.csdnimg.cn/20201211213734877.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

本模式下只有一个有序队列。将所有进入算子元素（包括水位）都包装为 Promise 按序放入有序队列`OrderedStreamElementQueue`。

当某个 Promise 有结果返回时，只会被标记已完成，但不会立刻发送。

Emitter 会等到队列队首（如上图 P1）元素完成得到结果后才会触发 Emitter 拉取队首 Promise 元素进行发送到下游。

有序模式下使用 EventTime 时，水位和数据顺序被保留。此时开销和使用`ProccsingTime`时差别不算很大。

#### 4.10.3.6 容错保证

异步 IO 算子提供完全的 exactly-once 容错包证。

会为`in-flight`异步请求缓存对应记录到 checkpoint，还会当从错误中恢复时恢复或重触发请求。

具体如下，转自[Flink 原理与实现：Aysnc I/O](http://wuchong.me/blog/2017/05/17/flink-internals-async-io/#)，作者阿里 - 伍翀：  
![](https://img-blog.csdnimg.cn/20201211223814661.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

#### 4.10.3.7 代码实现小贴士

推荐使用`DirectExecutor`，因为一般来说回调函数会做尽量少的工作，DirectExecutor 避免了额外的线程切换带来的开销。回调函数通常仅把结果发送给 ResultFuture，其实就是将他们添加到输出缓存。

从这里开始，包括发送记录和与 chenkpoint 交互在内的繁重逻辑都将在专有的线程池中进行处理。

DirectExecutor 可使用`org.apache.flink.runtime.concurrent.Executors.directExecutor()` 或`com.google.common.util.concurrent.MoreExecutors.directExecutor()`实现。

#### 4.10.3.8 警告

-   **Flink 不会以多线程的方式调用`AsyncFunction`**  
    AsyncFunction 只有一个实例，它被流中相应分区内的每个记录顺序地调用。除非 `asyncInvoke(...)`方法快速返回并且依赖于客户端回调，否则无法实现正确的异步 I/O。

    比如说，以下方式会导致阻塞的`asyncInvoke`，从而使异步行为无效：

    -   使用同步的数据库客户端  
        它的查询方法调用在返回结果之前一直被阻塞。
    -   在 `asyncInvoke(...)` 方法内阻塞等待异步客户端返回的 future 类型对象
-   目前，出于一致性的原因，`AsyncFunction`算子（`AsyncWaitOperator`）必须位于算子链的头部

## 4.11 更多例子

-   [Flink1.9 官方 - datastream_api 例子](https://ci.apache.org/projects/flink/flink-docs-release-1.9/getting-started/tutorials/datastream_api.html)  
    展示了 maven 骨架方式构建 Flink 工程、写 Flink 程序步骤、将数据写入 Kafka 的 Flink 程序实例运行在集群上的例子。
-   [Flink1.9 官方 - datastream_api 开发指南](https://ci.apache.org/projects/flink/flink-docs-release-1.9/dev/datastream_api.html)  
    包括代码示例、DataSource、DataTransformation、DataSink、容错、延时控制、调试等内容
-   [Flink1.11 官方 - datastream_api](https://ci.apache.org/projects/flink/flink-docs-master/getting-started/walkthroughs/datastream_api.html)  
    从预备知识、环境准备、工程构建、样例代码、代码分析、实战等多个方面详细介绍了 datastream_api。

    代码展示了如何实现简单的 DataStream 应用，并且扩展为有状态的，还引入了时间概念。
-   [Flink-Kafka-Connector Flink 结合 Kafka 实战](https://www.cnblogs.com/importbigdata/p/10779930.html)

批处理 API。用到的时候补充。

## 6.1 概述

这里说的参数是指在提交 Flink 任务时要传递到 Flink 程序里的参数，比如指定路径、集群地址、系统参数等。

可以使用传统的[Commons CLI](https://commons.apache.org/proper/commons-cli/)、[argparse4j](http://argparse4j.sourceforge.net/)来解析参数，也可以使用 Flink 自带的`ParameterTool`来操作。

## 6.2 ParameterTool

### 6.2.1 ParameterTool 读取放入

#### 6.2.1.1 从. properties 文件读取

三种方式如下：

```java
String propertiesFilePath = "/home/sam/flink/myjob.properties";
ParameterTool parameter = ParameterTool.fromPropertiesFile(propertiesFilePath);

File propertiesFile = new File(propertiesFilePath);
ParameterTool parameter = ParameterTool.fromPropertiesFile(propertiesFile);

InputStream propertiesFileInputStream = new FileInputStream(file);
ParameterTool parameter = ParameterTool.fromPropertiesFile(propertiesFileInputStream);

```

#### 6.2.1.2 从命令行读取

可以读取类似以下格式的命令行参数：

```bash
--input hdfs:///mydata --elements 42

```

代码如下:

```java
public static void main(String[] args) {
  ParameterTool parameter = ParameterTool.fromArgs(args);
}

```

#### 6.2.1.3 从系统属性读取

可以读取 JVM 系统参数如`-Dinput=hdfs:///mydata`。

此时代码如下：

```java
ParameterTool parameter = ParameterTool.fromSystemProperties();

```

### 6.2.2 ParameterTool 的使用

#### 6.2.2.1 ParameterTool 直接使用

```java
ParameterTool parameters = 
parameter.getRequired("input");
parameter.get("output", "myDefaultValue");

int parallelism = parameters.get("mapParallelism", 2);
DataSet<Tuple2<String, Integer>> counts = text.flatMap(new Tokenizer()).setParallelism(parallelism);

parameter.getLong("expectedCount", -1L);
parameter.getNumberOfParameters()

```

ParameterTool 是可序列化的，所以可以将它传给 function 使用：

```java
ParameterTool parameters = ParameterTool.fromArgs(args);
DataSet<Tuple2<String, Integer>> counts = text.flatMap(new Tokenizer(parameters));

```

#### 6.2.2.2 ParameterTool 全局注册

Parameters 被注册为 ExecutionConfig 中的全局 Job Parameter，可以从 JobManager web 接口和用户定义的所有 function 中访问。

注册全局参数代码如下：

```java
ParameterTool parameters = ParameterTool.fromArgs(args);


final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

env.getConfig().setGlobalJobParameters(parameters);

```

然后就可以在 RichFunction 中访问：

```java
public static final class Tokenizer extends RichFlatMapFunction<String, Tuple2<String, Integer>> {

    @Override
    public void flatMap(String value, Collector<Tuple2<String, Integer>> out) {
	ParameterTool parameters = (ParameterTool)
	    getRuntimeContext().getExecutionConfig().getGlobalJobParameters();
	parameters.getRequired("input");
	

```

这篇文章主要讲了一些 Flink 编程中用到的基本概念和 API，为了更加深入理解，还要多学习下 Example 才行，请点击[这里](https://ci.apache.org/projects/flink/flink-docs-release-1.6/examples/)。

-   [Flink-Training 在线学习](https://training.ververica.com/intro/rideCleansing.html)

-   [Flink 官网 DataStream ETL 应用](https://ci.apache.org/projects/flink/flink-docs-release-1.11/zh/learn-flink/etl.html)

-   [使用 flink Table &Sql api 来构建批量和流式应用 (1)Table 的基本概念](https://www.cnblogs.com/davidwang456/p/11161621.html)

-   [flink 入门实战总结](https://www.cnblogs.com/davidwang456/p/11256748.html)

-   [Flink 中的状态与容错](https://www.cnblogs.com/davidwang456/p/11124698.html)

-   [flink DataStream API 使用及原理](https://www.cnblogs.com/davidwang456/p/11046857.html)

-   [Streaming 101: The world beyond batch](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101)

-   [Streaming 102: The world beyond batch](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-102)

-   [What is Apache Flink?](https://flink.apache.org/flink-applications.html)

-   [Flink 实时性、容错机制、窗口等介绍](http://www.aboutyun.com/thread-25540-1-1.html)

-   \[Flink 事件时间处理和水印]([https://blog.csdn.net/a6822342/article/details/78064815(](<https://blog.csdn.net/a6822342/article/details/78064815(>))

-   [Flink Kafka Connector 详解](https://www.meiwen.com.cn/subject/yhbkfftx.html)

-   《Stream Processing with Apache Flink》

    -   作者：崔兴灿
    -   出处：阿里巴巴 - Flink 极客训练营

-   [Apache Flink 零基础入门（六）：Flink Time & Window 解析](https://ververica.cn/developers/time-window/)

    -   作者：邱从贤
    -   出处：Ververica

-   [Flink 原理与实现：Aysnc I/O](http://wuchong.me/blog/2017/05/17/flink-internals-async-io/)
    -   作者：伍翀 
        [https://blog.csdn.net/baichoufei90/article/details/82891909](https://blog.csdn.net/baichoufei90/article/details/82891909)
