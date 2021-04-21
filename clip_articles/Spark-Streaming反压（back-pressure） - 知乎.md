# Spark-Streaming反压（back-pressure） - 知乎
```text
写在前面：
本文主要针对kafka的Direct消费方式进行参数，其他的消费方式的反压逻辑这里不涉及。但是原理相差不多
```

## 一 作用

Spark Streaming 在处理不断流入的数据是通过每间隔一段时间 (batch interval)，将这段时间内的流入的数据积累为一个 batch，然后以这个 batch 内的数据作为 job DAG 的输入 RDD 提交新的 job 运行。当一个 batch 的处理时间大于 batch interval 时，意味着数据处理速度跟不上数据接收速度，此时在数据接收端(即 Receiver 一般数据接收端都运行在 executor 上) 就会积累数据，而数据是通过 BlockManager 管理的，如果数据存储采用 MEMORY_ONLY 模式就会导致 OOM，采用 MEMORY_AND_DISK 多余的数据保存到磁盘上反而会增加数据读取时间。

为了解决这一问题 Spark Streaming 从 v1.5 开始引入反压机制 (back-pressure)。通过动态控制数据控制数据接收速率来适配集群数据处理能力。

## 二 涉及参数

-   spark.streaming.backpressure.enabled 设置为 true 开启反压
-   spark.streaming.kafka.maxRatePerPartition 每个 partition 每秒最多消费条数
-   spark.streaming.backpressure.rateEstimator 速率估算器类，默认值为 pid ，目前 Spark 只支持这个。

**以下参数仅针对 PID 速率估算器设置**

-   spark.streaming.backpressure.pid.proportional：用于响应错误的权重（最后批次和当前批次之间的更改）。默认值为 1，只能设置成非负值。
-   spark.streaming.backpressure.pid.integral：错误积累的响应权重，具有抑制作用（有效阻尼）。默认值为 0.2 ，只能设置成非负值。
-   spark.streaming.backpressure.pid.derived：对错误趋势的响应权重。 这可能会引起 batch size 的波动，可以帮助快速增加 / 减少容量。默认值为 0，只能设置成非负值。
-   spark.streaming.backpressure.pid.minRate：可以估算的最低费率是多少。默认值为 100，只能设置成非负值。

## 三 实现过程

反压主要分为三个流程

1. 启动服务时注册 rateCotroller

2. 监听到批次结束事件后采样计算新的消费速率

3. 提交 job 时利用消费速率计算每个分区消费的数据条数

以下为详细流程及各流程的说明：

## 1. 启动服务时注册 rateCotroller

![](https://pic3.zhimg.com/v2-b9f9d3c4bab2f9dc2e8fbd5de0d1155a_b.jpg)

启动服务时注册 rateCotroller 过程

在程序运行到 StreamingContext 的 start 方法时会调用 JobScheduler 的 start 方法，在这里会根据消费者的不同生成不同的 RateController，在 kafka 中生成的是 DirectKafkaRateController 实例。接下来会把生成的 RateController 注册到 StreamingListenerBus 中。

## 2. 监听到批次结束事件后采样计算新的消费速率

![](https://pic4.zhimg.com/v2-4140d3061850bfa76efe49a22a36bcf3_b.jpg)

计算新的消费速率流程

(1) StreamingListener 的 doPostEvent 监听到批次结束事件后会调用 RateController 的 onBatchCompleted 方法, 在此方法中会获取 processingEnd(处理结束时间)、workDelay(处理耗时)、waitDelay(调度延迟)、elems(消息条数) 着四个参数用于计算新的消费速率

(2) computedAndPublish 方法会获取新的消费速率。

(3) computedAndPublish 方法会调用 RateEstimator 接口的 compute 方法，现在 spark 支持的唯一实现类是 PIDRateEstimator。compure 方法返回的消费速率值会随着不断采样速率值趋向稳定。

(4) computedAndPublish 方法调用 compute 方法获取到新的速率后把新的消费速率赋值到 rateLimit 属性中。

## 3. 提交 job 时利用消费速率计算每个分区消费的数据条数

![](https://pic1.zhimg.com/v2-ffed8cd8a0a493cf15d8f87bf8ae72b0_b.jpg)

计算消费条数流程

(1) 在提交 Job 时会调用 DirectKafkainputDStream 类的 compute 方法获取这一批次处理 KafkaRDD。

(2) compute 方法会调用 clamp 方法获取这一批次每个 partitions 要消费的截止 offset(取 kafka 最新的 offset 和反压计算后的 offset 的最小值)。

(3) clamp 方法会调用 maxMessagesPerPartition 方法通过消费速率计算出每个 partition 消费的截止 offset。

(4) maxMessagesPerPartition 方法调用 getLatestRate，获取消费速率。

(5) 通过获取的消费速率计算每个分区消费的记录数。计算方式如下：

-   所有分区滞后 offset 总和为 totalLag
-   某一分区滞后 offset 为 lagPerPartition
-   消费速率为 lastestRage
-   每个分区消费条数为 Match.round(为 lagPerPartition/totalLag\*lastestRage)

## 四 小结

以上为 Kafka 的 Direc 消费方式的反压逻辑，其中 spark 消费方式还有 Receiver 方式，部分逻辑会有些许不同。

## 参考文章

-   [Spark Streaming(4) - 反压](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/2b4643dec7a4)
-   [Spark 的 PIDController 源码赏析及 backpressure 详解](https://link.zhihu.com/?target=https%3A//blog.csdn.net/qq_42950313/article/details/81610199)
-   [Spark Streaming Backpressure 分析](https://link.zhihu.com/?target=http%3A//www.cnblogs.com/barrenlake/p/5349949.html)
-   [Spark Streaming 反压（Back Pressure）机制介绍](https://link.zhihu.com/?target=https%3A//www.iteblog.com/archives/2323.html) 
    [https://zhuanlan.zhihu.com/p/45954932](https://zhuanlan.zhihu.com/p/45954932)
