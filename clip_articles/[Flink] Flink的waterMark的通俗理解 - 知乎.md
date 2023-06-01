# [Flink] Flink的waterMark的通俗理解 - 知乎
[\[Flink\] Flink 的 waterMark 的通俗理解 - 知乎](https://zhuanlan.zhihu.com/p/126095557) 

### 导读

Flink 为实时计算提供了三种时间，即**事件时间**（event time）、**摄入时间**（ingestion time）和**处理时间**（processing time）。

### 遇到的问题：

假设在一个 5 秒的 Tumble 窗口，有一个 EventTime 是 11 秒的数据，在第 16 秒时候到来了。图示第 11 秒的数据，在 16 秒到来了，如下图：该如何处理迟到数据

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-1%2014-40-00/64b47e37-3cf7-49fc-8da9-33f5b8572a33.jpeg?raw=true)

### 什么是 Watermark

Watermark 的关键点：

-   目的：处理 EventTime 窗口计算
-   本质：时间戳
-   生成方式：Punctuated 和 Periodic(常用)
-   特性：单调递增

### Watermark 的产生方式

-   Punctuated

    数据流中每一个递增的 EventTime 都会产生一个 Watermark。
-   Periodic（推荐）

    周期性的（一定时间间隔或者达到一定的记录条数）产生一个 Watermark。

### Watermark 解决的问题

上面的问题在于如何将迟来的 EventTime 位 11 的元素正确处理？

当 Watermark 的时间戳等于 Event 中携带的 EventTime 时候，上面场景（Watermark=EventTime) 的计算结果如下：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-1%2014-40-00/3e9db395-d617-492a-aade-253dce76d91e.jpeg?raw=true)

如果想正确处理迟来的数据可以定义 Watermark 生成策略为 Watermark = EventTime -5s， 如下：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-1%2014-40-00/472de080-4998-4340-8387-50f195bb3926.jpeg?raw=true)

### WaterMark 的例子

设置 WaterMark 步骤：

1\. 设置 StreamTime Characteristic 为 Event Time，即设置流式时间窗口（也可以称为流式时间特性）

2\. 创建的 DataStreamSource 调用 assignTimestampsAndWatermarks 方法，并设置 WaterMark 种类：AssignerWithPeriodicWatermarks / AssignerWithPunctuatedWatermarks

或者 实现 AssignerWithPeriodicWatermarks 接口 / 实现 AssignerWithPunctuatedWatermarks 接口

3\. 重写 getCurrentWatermark 与 extractTimestamp 方法

getCurrentWatermark 方法：获取当前的水位线

extractTimestamp 方法：提取数据流中的时间戳（必须显式的指定数据中的 Event Time）

**实例**

通过一段程序，实践一下 WaterMark 的设定以及 WaterMark 的工作方式

**数据示例**：

key + 时间戳

**程序说明**：

1\. 使用 Socket 模拟接收数据

2\. 设置 WaterMark

设置的逻辑：在第一条数据进来时，设置 WaterMark 为 0，指定第一条数据的时间戳后，获取该时间戳与当前 WaterMark 的最大值，并将最大值设置为下一条数据的 WaterMark，以此类推

3\. 进行 map 基础转换，将 String 转换为 Tuple2&lt;String,String>

4\. 根据 Key 分组

5\. 使用滚动 Event Time 窗口，将 5 秒内的同组数据，进行 Fold 拼接输出

代码如下：

```text
package waterMark;

import org.apache.flink.api.common.functions.FoldFunction;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.AssignerWithPeriodicWatermarks;
import org.apache.flink.streaming.api.watermark.Watermark;
import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;


import javax.annotation.Nullable;

/**
 * waterMark实例
 *
 * @author lixiyan
 * @date 2019/10/22 4:45 PM
 */
public class MainWaterMark001 {
    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
        SingleOutputStreamOperator<String> dataStream = env.socketTextStream("localhost", 12345)
                .assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarks<String>() {
                    // 当前时间戳
                    long currentTimeStamp = 0L;
                    // 允许的迟到数据
                    long maxDelayAllowed = 0L;
                    // 当前水位线
                    long currentWaterMark;

                    @Nullable
                    @Override
                    public Watermark getCurrentWatermark() {
                        currentWaterMark = currentTimeStamp - maxDelayAllowed;
                        System.out.println("当前水位线:" + currentWaterMark);
                        return new Watermark(currentWaterMark);
                    }

                    @Override
                    public long extractTimestamp(String s, long l) {
                        String[] arr = s.split(",");
                        long timeStamp = Long.parseLong(arr[1]);
                        currentTimeStamp = Math.max(timeStamp, currentTimeStamp);
                        System.out.println("Key:" + arr[0] + ",EventTime:" + timeStamp + ",水位线:" + currentWaterMark);
                        return timeStamp;
                    }
                });

        dataStream.map(new MapFunction<String, Tuple2<String, String>>() {
            @Override
            public Tuple2<String, String> map(String s) throws Exception {
                return new Tuple2<String, String>(s.split(",")[0], s.split(",")[1]);
            }
        }).keyBy(0).window(TumblingEventTimeWindows.of(Time.seconds(5)))
                .fold("Start:", new FoldFunction<Tuple2<String, String>, String>() {
                    @Override
                    public String fold(String s, Tuple2<String, String> o) throws Exception {
                        return s + " - " + o.f1;
                    }
                }).print();

        env.execute("MainWaterMark001");

    }
}

```

开启 9999 端口，并输入第一条数据：

那么，我先假设后续的数据 Event Time 间隔为 1 秒，推断一下 WaterMark 的设定，如下图所示

1\. 第一条数据的 Event Time 为 1553503185000，那么当前窗口时间为：1553503185000 -> 1553503189000，即下图中红色框线

2\. 第一条数据进来时，这条数据之前的 WaterMark 为 0，当第一条数据已经进入后，指定 Event Time 位置，并与现在的 WaterMark 比较，将两者中大的那个值设置为新的 WaterMark，那么当前数据的 WaterMark 为 1553503185000

3\. 第二条数据进来时，前一条数据的 WaterMark 为 1553503185000，第二条数据的 Event Time 比之前的 WaterMark 大，于是更新 WaterMark，将当前的 WaterMark 更新为 1553503186000，但还没到窗口触发时间，不进行计算

4\. 后面几个以此类推，直到 Event Time 为：1553503190000 的数据进来的时候，前一条数据的 WaterMark 为 1553503189000，于是更新当前的 WaterMark 为 155350390000，==Flink 认为 1553503190000 之前的数据都已经到达，且达到了窗口的触发条件，开始进行计算 ==

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-1%2014-40-00/69d41a2b-ef48-4d26-af58-cb5fca1245a1.jpeg?raw=true)

根据上面的推断，启动程序验证一下

先启动监听 9999 端口，再启动 Flink 程序，并向端口监听终端输入以下内容：

```text
hello,1553503185000
hello,1553503186000
hello,1553503187000
hello,1553503188000
hello,1553503189000
hello,1553503190000
```

Flink 输出结果：

```text
Key:hello,EventTime:1553503185000,水位线:0
Key:hello,EventTime:1553503186000,水位线:1553503185000
Key:hello,EventTime:1553503187000,水位线:1553503186000
Key:hello,EventTime:1553503188000,水位线:1553503187000
Key:hello,EventTime:1553503189000,水位线:1553503188000
Key:hello,EventTime:1553503190000,水位线:1553503189000
2> Start: - 1553503185000 - 1553503186000 - 1553503187000 - 1553503188000 - 1553503189000
```

通过结果可以发现，Flink 在指定 WaterMark 时，== 先调用 extractTimestamp 方法，再调用 getCurrentWatermark 方法 ==， 所以打印信息中的 WaterMark 为上一条数据的 WaterMark，并非当前的 WaterMark

为了验证这个结论，修改一下代码：

```text
@Nullable
@Override
public Watermark getCurrentWatermark() {
    currentWaterMark = currentTimeStamp - maxDelayAllowed;
    System.out.println("当前水位线:" + currentWaterMark);
    return new Watermark(currentWaterMark);
}

@Override
public long extractTimestamp(String s, long l) {
    String[] arr = s.split(",");
    long timeStamp = Long.parseLong(arr[1]);
    currentTimeStamp = Math.max(timeStamp, currentTimeStamp);
    System.out.println("Key:" + arr[0] + ",EventTime:" + timeStamp + ",前一条数据的水位线:" + currentWaterMark);
    return timeStamp;
}
```

在监听终端输入同一批数据：

```text
hello,1553503185000
hello,1553503186000
hello,1553503187000
hello,1553503188000
hello,1553503189000
hello,1553503190000
```

Flink 输出结果：

```text
Key:hello,EventTime:1553503185000,前一条数据的水位线:0
当前水位线:1553503185000

Key:hello,EventTime:1553503186000,前一条数据的水位线:1553503185000
当前水位线:1553503186000

Key:hello,EventTime:1553503187000,前一条数据的水位线:1553503186000
当前水位线:1553503187000

Key:hello,EventTime:1553503188000,前一条数据的水位线:1553503187000
当前水位线:1553503188000

Key:hello,EventTime:1553503189000,前一条数据的水位线:1553503188000
当前水位线:1553503189000

Key:hello,EventTime:1553503190000,前一条数据的水位线:1553503189000
当前水位线:1553503190000
2> Start: - 1553503185000 - 1553503186000 - 1553503187000 - 1553503188000 - 1553503189000
```

通过上面的结果，验证了之前的结论，在设置 WaterMark 方法中，== 先调用 extractTimestamp 方法，再调用 getCurrentWatermark 方法 ==

### 数据乱序

上面的实例，Event Time 是有序，现在来做一下数据乱序的场景模拟

启动程序，在监听终端中输入如下数据：

其中，在触发了了第一个窗口计算后，又来了两条迟到数据 hello,1553503187000，hello,1553503186000

```text
hello,1553503185000
hello,1553503186000
hello,1553503187000
hello,1553503188000
hello,1553503189000
hello,1553503190000
hello,1553503187000
hello,1553503186000
hello,1553503191000
hello,1553503192000
hello,1553503193000
hello,1553503194000
hello,1553503195000
```

Flink 结果：

```text
Key:hello,EventTime:1553503185000,前一条数据的水位线:0
当前水位线:1553503185000

Key:hello,EventTime:1553503186000,前一条数据的水位线:1553503185000
当前水位线:1553503186000

Key:hello,EventTime:1553503187000,前一条数据的水位线:1553503186000
当前水位线:1553503187000

Key:hello,EventTime:1553503188000,前一条数据的水位线:1553503187000
当前水位线:1553503188000

Key:hello,EventTime:1553503189000,前一条数据的水位线:1553503188000
当前水位线:1553503189000

Key:hello,EventTime:1553503190000,前一条数据的水位线:1553503189000
当前水位线:1553503190000
2> Start: - 1553503185000 - 1553503186000 - 1553503187000 - 1553503188000 - 1553503189000
当前水位线:1553503190000

Key:hello,EventTime:1553503187000,前一条数据的水位线:1553503190000
当前水位线:1553503190000

Key:hello,EventTime:1553503186000,前一条数据的水位线:1553503190000
当前水位线:1553503190000

Key:hello,EventTime:1553503191000,前一条数据的水位线:1553503190000
当前水位线:1553503191000

Key:hello,EventTime:1553503192000,前一条数据的水位线:1553503191000
当前水位线:1553503192000

Key:hello,EventTime:1553503193000,前一条数据的水位线:1553503192000
当前水位线:1553503193000

Key:hello,EventTime:1553503194000,前一条数据的水位线:1553503193000
当前水位线:1553503194000

Key:hello,EventTime:1553503195000,前一条数据的水位线:1553503194000
当前水位线:1553503195000
2> Start: - 1553503190000 - 1553503191000 - 1553503192000 - 1553503193000 - 1553503194000
```

从结果中可以看到，在第二个窗口中，那两条迟到数据并没有进行处理，这个就是迟到丢弃。

### 乱序时间的设置：

为了解决上面的问题，我们允许 Flink 处理延迟以 5 秒内的迟到数据

修改最大乱序时间

```text
long maxDelayAllowed = 5000l;
```

在监听终端中，输入数据

```text
hello,1553503185000
hello,1553503186000
hello,1553503187000
hello,1553503188000
hello,1553503189000
hello,1553503190000
hello,1553503187000
hello,1553503186000
hello,1553503191000
hello,1553503192000
hello,1553503193000
hello,1553503194000
hello,1553503195000
```

Flink 输出结果：

```text
Key:hello,EventTime:1553503185000,前一条数据的水位线:-5000
当前水位线:1553503180000

Key:hello,EventTime:1553503186000,前一条数据的水位线:1553503180000
当前水位线:1553503181000

Key:hello,EventTime:1553503187000,前一条数据的水位线:1553503181000
当前水位线:1553503182000

Key:hello,EventTime:1553503188000,前一条数据的水位线:1553503182000
当前水位线:1553503183000

Key:hello,EventTime:1553503189000,前一条数据的水位线:1553503183000
当前水位线:1553503184000

Key:hello,EventTime:1553503190000,前一条数据的水位线:1553503184000
当前水位线:1553503185000

Key:hello,EventTime:1553503187000,前一条数据的水位线:1553503185000
当前水位线:1553503185000

Key:hello,EventTime:1553503186000,前一条数据的水位线:1553503185000
当前水位线:1553503185000

Key:hello,EventTime:1553503191000,前一条数据的水位线:1553503185000
当前水位线:1553503186000

Key:hello,EventTime:1553503192000,前一条数据的水位线:1553503186000
当前水位线:1553503187000

Key:hello,EventTime:1553503193000,前一条数据的水位线:1553503187000
当前水位线:1553503188000

Key:hello,EventTime:1553503194000,前一条数据的水位线:1553503188000
当前水位线:1553503189000

Key:hello,EventTime:1553503195000,前一条数据的水位线:1553503189000
当前水位线:1553503190000
2> Start: - 1553503185000 - 1553503186000 - 1553503187000 - 1553503188000 - 1553503189000 - 1553503187000 - 1553503186000
```

可以看到，设置了最大允许乱序时间后，WaterMark 要比原来低 5 秒，可以对延迟 5 秒内的数据进行处理，窗口的触发条件也同样会往后延迟

关于延迟时间，请结合业务场景进行设置

至此，WaterMark 实例就写完了

### 总结

一开始，你先不要把 Windowing、WaterMark、Trigger 三者混在一起去考虑最终输出的结果是什么，建议独立考虑清楚这三者都做了什么，以及三者之间的依赖关系是什么：

1、**Windowing**：就是负责该如何生成 Window，比如 Fixed Window、Slide Window，当你配置好生成 Window 的策略时，Window 就会根据时间动态生成，最终得到一个一个的 Window，包含一个时间范围：\[起始时间, 结束时间)，它们是一个一个受限于该时间范围的事件记录的容器，每个 Window 会收集一堆记录，满足指定条件会触发 Window 内事件记录集合的计算处理。

2、**WaterMark**：它其实不太好理解，可以将它定义为一个函数 E=f(P)，当前处理系统的处理时间 P，根据一定的策略 f 会映射到一个事件时间 E，可见 E 在坐标系中的表现形式是一条曲线，根据 f 的不同曲线形状也不同。假设，处理时间 12:00:00，我希望映射到事件时间 11:59:30，这时对于延迟 30 秒以内（事件时范围 11:59:30~12:00:00）的事件记录到达处理系统，都指派到时间范围包含处理时间 12:00:00 这个 Window 中。事件时间超过 12:00:00 的就会由 Trigger 去做补偿了。

3、**Trigger**：为了满足实际不同的业务需求，对上述事件记录指派给 Window 未能达到实际效果，而做出的一种补偿，比如事件记录在 WaterMark 时间戳之后到达事件处理系统，因为已经在对应的 Window 时间范围之后，我有很多选择：选择丢弃，选择是满足延迟 3 秒后还是指派给该 Window，选择只接受对应的 Window 时间范围之后的 5 个事件记录，等等，这都是满足业务需要而制定的触发 Window 重新计算的策略，所以非常灵活。

> 本文由博客群发一文多发等运营工具平台 [OpenWrite](https://link.zhihu.com/?target=https%3A//openwrite.cn%3Ffrom%3Darticle_bottom) 发布
