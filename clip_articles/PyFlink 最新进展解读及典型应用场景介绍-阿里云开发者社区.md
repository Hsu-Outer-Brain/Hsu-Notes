# PyFlink 最新进展解读及典型应用场景介绍-阿里云开发者社区
[PyFlink 最新进展解读及典型应用场景介绍 - 阿里云开发者社区](https://developer.aliyun.com/article/1137389) 

> 摘要：本文整理自阿里巴巴高级技术专家付典，在 FFA 核心技术专场的分享。本篇内容主要分为四个部分：
>
> 1.  PyFlink 发展现状介绍
> 2.  PyFlink 最新功能解读
> 3.  PyFlink 典型应用场景介绍
> 4.  PyFlink 下一步的发展规划

**[点击查看直播回放 & 演讲 PPT](https://flink-learning.org.cn/activity/detail/f30571911e47478ddf4047eeb518d796?name=activity&tab=Meetup&page=1)**

## 一、PyFlink 发展现状介绍

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/181a5318-75ad-4268-9ecf-6835f7118a13.jpeg?raw=true)

很多 PyFlink 的新用户都会问这样一些问题，PyFlink 是否成熟？功能是否齐全？性能怎么样？在这里，我们针对用户的这样一些问题，进行一个详细的解读。

首先，在功能层面，PyFlink 已经对齐了 Flink Java API 中的绝大多数功能。用户使用 Java API 可以实现的功能，基本上都可以用 Python API 实现得出来。

同时，PyFlink 还面向 Python 用户提供了很多特有的能力，比如说 Python UDF、Pandas UDF 等，允许用户在 PyFlink 作业中使用各种 Python 三方库。

在部署模式上，PyFlink 支持各种常见的部署模式，比如说 YARN、kubernetes、Standalone 等，这意味着用户可以根据需要，灵活地选择作业的部署模式。

除了功能层面之外，性能也是很多用户非常关心的。在性能上，PyFlink 也做了很多优化，首先，在执行计划层面，PyFlink 做了一系列的优化，尽可能优化作业的物理执行计划，比如算子融合。

当作业的物理执行计划确定之后，在 Python 运行时，PyFlink 通过 Cython 实现了 Python 运行时中核心链路的代码，尽量降低 Python 运行时中框架部分的开销。对于 Cython 有所了解的同学应该知道，Cython 在执行时会被编译成 native 代码来执行，性能非常高。

同时，PyFlink 还在现有的进程模式的基础之上，引入了线程执行模式，以进一步提升 Python 运行时的性能。线程执行模式在 JVM 中执行用户的 Python 代码，通过这种方式，在一些典型应用场景中，性能甚至可以追平 Java，这一块后面我们还会详细介绍。

经过这一系列的优化之后，目前 PyFlink 无论在功能上还是在性能上，都已经基本完备，达到了生产可用的状态。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/e8f8b049-003b-4d99-b1a4-70833c6c1e5c.jpeg?raw=true)

PyFlink 达到目前这样一个状态，并不是一蹴而就的，从 Flink 1.9 开始引入 PyFlink，到目前为止，PyFlink 已经累计发布了 8 个大版本，20 多个小版本。

从 Flink 1.9 到 Flink 1.11 这几个版本中，我们重点在完善 Python Table API，基本上对齐了 Java Table API 中的绝大多数功能，同时也支持了 Python UDF、Pandas UDF 等功能。

在 Flink 1.12 至 Flink 1.14 版本中，社区主要是在完善 Python DataStream API，目前已经基本上对齐了 Python DataStream API 上的绝大部分常用功能。

在 Flink 1.15 至 Flink 1.16 版本中，PyFlink 的重点是在性能优化上，在原有的进程执行模式的基础之上，为 Python 运行时引入了线程执行模式，以进一步地提升 Python 运行时的性能。

随着 PyFlink 功能的逐渐完善，我们也看到 PyFlink 的用户数也在逐渐增长，PyPI 的日均下载量在过去一年也有了显著的增长，从最开始的日均 400 多次，已经增长到日均 2000 多次。

## 二、PyFlink 最新功能解读

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/96431aa6-541b-41aa-9769-bf94a6ac1a1f.jpeg?raw=true)

接下来，我们看一下 PyFlink 在 Flink 1.16 中的功能。PyFlink 在 Flink 1.16 中支持的功能主要是围绕使 PyFlink 在功能及性能上全面生产可用这样一个目的。为此，我们重点补齐了 PyFlink 在功能以及性能上的最后几处短板。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/e8555c80-c800-48fb-8a41-b8c5dcc6222d.jpeg?raw=true)

如上图所示，PyFlink 在 Flink 1.16 中支持了 side output 功能。用户可以把一条数据流，切分成多条数据流。以机器学习为例，用户可以通过该功能，把一份数据集给切分成多份，分别用于模型训练和模型验证。

除此之外，用户也可以通过 side output 处理迟到数据或者脏数据。将迟到数据或者脏数据通过 side output 拆分出来，单独进行处理。用户也可以通过 side output 把迟到数据或脏数据，写入外部存储，进行离线分析。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/2ca98f1a-06c7-4854-ae76-2a6291bad3d0.jpeg?raw=true)

PyFlink 在 Flink 1.16 中，还支持了 broadcast state。通过该功能，用户可以将一条数据流中的数据，广播发送到另一条数据流算子中的多个并发实例上，并通过 broadcast state 保存广播流的状态，以确保作业在 failover 时，所有算子恢复的状态是一致的。

比如我们用 PyFlink 做近线预测，当模型更新后，可以将最新的模型文件地址，广播发送到所有的预测算子，来实现模型的热更新，并通过 broadcast state 确保在作业 failover 时，所有算子加载的模型文件是一致的。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/6a379357-6d7c-474c-ab28-d83572724c70.jpeg?raw=true)

PyFlink 在 Flink 1.16 中，对于 DataStream API 上 Window 的支持也做了很多完善，原生支持了各种窗口，比如滚动窗口、滑动窗口、会话窗口等等。Window 可以将无限流中的数据，划分成不同的时间窗口进行计算，在流计算中是非常重要的功能，有着非常丰富的应用场景。

比如机器学习用户可以使用 Window 来计算实时特征。在短视频应用中，可以通过 Window，计算用户最近五分钟的有效视频观看列表，也可以通过 Window，来计算最近 30 分钟，某个视频在各个人群中的点击分布等。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/3c01232a-9a92-4f74-a865-b8cb2aedff98.jpeg?raw=true)

除此之外，在 Flink 1.16 中，PyFlink 还新增了对于很多 DataStream API 上 connector 的支持，包括 Elasticsearch、Kinesis、Pulsar、Hybrid source 等。与此同时，也支持 Orc、Parquet 等 format。

有了这些 connector 以及 format 的支持，PyFlink 基本上已经对齐了所有 Table API 以及 DataStream API 上 Flink 官方所支持的 connector。

需要说明的是，对于 PyFlink 中没有原生提供支持的 connector，如果有对应的 Java 实现，也是可以在 PyFlink 作业中使用的，其中 Table API 以及 SQL 上的 connector，可以直接在 PyFlink 作业中使用，不需要任何开发。

对于 DataStream API connector，用户只需要非常少量的开发即可在 PyFlink 作业中使用。如果用户有需求的话，可以参考一下 PyFlink 中现有的 DataStream API connector 是如何支持的，基本上只需要一两个小时即可完成一个 connector 的支持。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/bd014517-01ef-4eec-9631-021692a6578e.jpeg?raw=true)

除了前面介绍的这些功能层面的增强之外，在性能层面，Flink 1.16 也做了很多工作，基本完成了 Python 运行时线程执行模式的支持。相比于进程执行模式，线程执行模式的性能更好。

线程执行模式通过 JNI 调用的方式，执行 Python 代码，节省序列化 / 反序列化开销及通信开销。特别是当单条数据比较大时，效果更加明显。

由于不涉及跨进程通信，线程执行模式目前采用同步执行的方式，不需要在算子中进行攒批操作，没有攒批延迟，适用于对延迟敏感的场景，比如量化交易。

与此同时，与其他 Java/Python 互调用方案相比，PyFlink 所采用的方案兼容性更好。很多 Java/Python 互调用方案对于所能支持的 Python 库，都有一定程度的限制。PyFlink 所采用的方案，对于用户在作业中所使用的 Python 库没有任何限制。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/cd9e4e73-5598-4c54-899d-550216334d84.jpeg?raw=true)

如上图左侧所示，展示了进程模式和线程模式架构的区别。进程模式需要启动一个独立的 Python 进程，用于执行用户的 Python 代码。线程执行模式在 JVM 中，通过 JNI 调用的方式执行 Python 代码。PEMJA 是 PyFlink 中 Java 代码和 Python 代码之间互调用的库。

如上图右侧所示，在处理时延上，相比于进程模式，线程模式有显著的降低。因为进程模式不需要攒数据，来一条处理一条。与此同时，在处理性能上，线程模式相比进程模式也有较大的提升，在某些情况下，性能甚至可以追平 Java。

这里需要说明的是，Python UDF 的执行性能既取决于 PyFlink 执行框架的性能，也跟 Python UDF 的实现是否高效息息相关。通过各种优化手段，目前 PyFlink 执行框架的性能已经非常高效，开销非常小。用户的 PyFlink 作业的执行性能很大程度取决于，用户作业中的 Python UDF 实现得是否高效。

如果用户的 Python UDF 实现得足够高效，比如说实现的过程中针对一些耗时操作，有针对性地进行来一些优化或者利用一些高性能的 Python 三方库，那么 PyFlink 作业的性能其实是可以实现的非常好的。

## 三、PyFlink 典型应用场景介绍

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/bd83dd99-52a6-4f93-889f-1e9d3c25945b.jpeg?raw=true)

接下来，讲一讲 PyFlink 的应用场景。目前，实时机器学习是 PyFlink 用户的重点应用场景。以推荐系统为例，上图是实时推荐系统的一个典型架构。用户的行为日志，通过 APP 埋点等手段，实时采集到消息队列中，经过实时数据清洗，归一化处理之后，在特征生成、样本拼接等模块使用。实时的用户行为日志，可以用来计算实时特征。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/4cd073b3-c2b0-412f-a9ae-ae76262feec4.jpeg?raw=true)

首先，实时用户行为日志可以被用来计算实时特征。实时特征是 Flink 非常重要的应用场景。实时特征对于推荐效果的提升非常明显，建设难度相对来说比较小，是当前很多公司投入的重点。

比如在短视频应用中，用户最近 N 分钟的有效视频观看列表就是短视频应用中，非常重要的用户实时特征。这个特征可以通过一个 Flink 作业，实时分析用户的行为日志得到。

一般来说，用户的行为日志还会同步一份到离线存储中，用于生成离线特征。这块主要是用于计算一些复杂特征或者是说长周期特征。不管是离线特征还是实时特征，最终都会存储到特征库中供在线推荐系统使用。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/a49a5be2-e041-479d-a21d-8fd77e260e1a.jpeg?raw=true)

实时的用户行为日志不但可以用来构造实时特征，而且可以用来构造实时样本，用于模型训练。

通过分析用户的行为日志，可以自动完成对样本打标签。比如在推荐系统中，给用户推荐了 10 个 item，如果用户点击了某个 item，那么在行为日志中就会出现这个 item 的点击事件。有了这个点击事件，我们就可以得到一条正样本。同理，如果对于某个 item，只有曝光事件，没有点击事件，我们就可以将其看成是一条负样本。

除了区分正负样本之外，还需要拼接上用户的特征以及 item 的特征之后，才能得到一条完整的样本。这里需要注意的是，做样本拼接时所用的特征不是来自于实时特征库，而是来自于历史特征库。

由于实时特征库中的特征是不断更新的，比如在短视频应用中，用户最近 N 分钟的视频点击列表特征，随着时间的推移，在不断发生变化。因此在样本拼接时，我们希望拼接推荐发生时所用到的特征，而不是当前时刻的特征。样本拼接可能发生在推荐事件过去一段时间之后，此时在实时特征库中存储的特征可能已经发生了变化，因此这里拼接的是历史特征库。因为历史特征库中的数据来自于推荐发生时所用到的特征。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/e1d82e7d-069c-47c8-a329-0a0e494a39c0.jpeg?raw=true)

样本经过训练之后，最终生成模型。经过验证，如果没有问题，就可以把模型部署到线上，供在线推理服务使用。

在推荐系统中，在线推理服务包括召回、排序等多个环节。其中，在召回环节使用比较广泛的一种手段是多路召回技术，每一路召回使用不同的策略。比如说可以根据用户画像、当前的热点内容、运营策略等，分别生成不同的召回结果。对于这些召回结果，合并之后，再经过排序等环节之后展示给用户。

由此可见，多路召回的好处是显而易见的。通过多路召回，可以增强推荐结果的多样性。这里需要指出的是，由于推荐系统对于延时比较敏感，对于召回策略或者模型的性能要求非常高。

因此，召回中使用的模型或者策略一般都比较简单。目前一些公司也在探索，在多路召回系统中引入近线召回。近线召回可以预先计算召回结果，并将召回结果缓存，作为多路召回中的一路，供在线推理服务直接使用。因此近线召回没有时延约束，用户可以在近线召回中使用一些比较复杂的模型或者策略。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/25c3e7ea-e40b-42f6-a83a-9b0ef66d5487.jpeg?raw=true)

接下来，介绍一下在上述步骤中，如何使用 PyFlink 完成各项功能的开发。在实时数据清洗部分，机器学习应用中，输入数据中往往包含很多列。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/419476cc-943a-4b63-bd5c-c510e6d3eae3.jpeg?raw=true)

Flink 的其他功能也可以用于数据清洗，比如 SQL。SQL 本身也是非常方便的，那么和 SQL 相比，PyFlink 可以提供哪些附加价值呢？

首先，在机器学习场景中，普遍有一个共性的特点，数据中的列非常多，可能有几十列甚至上百列。在这种情况下 SQL 语句可能写起来非常长，比如在这个例子中，用户可能希望对第 9 列和第 10 列进行一个合并操作，其他列保留。

但是在 SELECT 语句中，需要把所有的其他的无关列都写出来，如果数据中的列非常多，写起来非常繁琐，同时可读性、可维护性也会变得比较差，PyFlink 对于这块提供了完善的支持。

另外，机器学习用户通常对于 Pandas 比较熟悉，习惯于使用 Pandas 进行数据处理，很多机器学习相关的库的数据结构都是采用 Pandas 或者 Numpy 的数据结构，PyFlink 在这块也提供了很好的支持，支持用户在 Python UDF 的实现中使用 Pandas 库。

接下来，我们通过几个具体的例子看一下如何在 PyFlink 中使用上述功能。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/a327bc4a-f259-40ea-9211-2cafe688f01b.jpeg?raw=true)

首先，PyFlink 提供了行操作和列操作的 API，从而简化用户的代码逻辑。通过列操作，用户可以非常方便的增加列、删除列或替换列。列操作适用于输入数据的列很多，且只有个别列发生变化的场景。

比如在上述例子中，我们通过 add_or_replace_columns 操作，对数据中的 item_id 一列进行归一化后，替换原有的 item_id 列，数据中的其他列不需要再显式列出来。

除了列操作之外，PyFlink 还支持行操作，可以以行为单位对数据进行变换。在行操作的 UDF 中，可以直接通过列名引用对应列，使用起来非常方便，适用于输入数据中的列很多，且需要对多个列进行处理的场景。

在行操作中，不需要在 UDF 的输入参数中，把所有用到的列都显式列出来，而是把一行数据都作为输入传进来，供 UDF 使用。

在上述例子中，我们通过 map 操作对数据进行变换，map 的输入是一个 Python 函数，Python 函数的输入输出类型都是 Row 类型，Row 是 PyFlink 中定义的一个数据结构，在 Python 函数的实现中可以通过列名引用输入数据中对应列的值，使用起来非常方便。

除了 map 之外，PyFlink 中还提供了多个行操作相关的 API，如果有需要的话，大家可以从 PyFlink 的官方文档中了解详细信息。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/401bdefb-b4b8-498a-9748-b726483c181f.jpeg?raw=true)

如果用户熟悉 Pandas 库，也可以在 Python 函数中使用 Pandas 库。用户只需要将 Python 函数的类型，标记成 Pandas 即可。

在这种情况下，Python 函数的输入类型是 Pandas 的 DataFrame。PyFlink 运行时框架会在调用用户的 Python 函数之前，将输入数据转换成 Pandas 的 DataFrame 结构，方便用户使用。除此之外，Python 函数的输出类型也需要是 Pandas DataFrame。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/08fe2006-594e-4333-9363-5f62cd3e3140.jpeg?raw=true)

接下来，我们看一下实时特征计算部分。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/d165772d-fec4-41c7-a27f-509f75c31581.jpeg?raw=true)

当前有很多公司，开发一个实时特征任务的流程通常是这样的：

首先，算法团队的同学通过数据挖掘等手段，发现某个特征会比较有用。然后，找到数据开发团队的同学，进行需求沟通。将特征的详细描述信息，甚至 Python 代码参考实现，提给数据开发团队的同学。

然后，数据开发团队的同学进行需求排期并实现。在实现的过程中，算法团队的同学和数据开发团队的同学可能还需要进行多轮沟通，确保数据开发团队的同学的理解和实现没有问题。

另外，算法团队同学提供的 Python 参考实现，有可能不太容易翻译成 Java 代码。比如里面用到了一些 Python 三方库，找不到合适的 Java 实现等等。

最后，当特征任务开发好之后，算法同学经过一系列的验证，很有可能发现这个特征可能并没有预期中的效果那么好，这样的一个特征任务很可能就废弃了。数据开发团队的同学也白忙活了。

从这个过程中，我们看到特征的开发成本是非常高的，涉及到跨团队的沟通、开发语言的转换，特征上线的周期也非常长，通常以周甚至月为单位。

而 PyFlink 可以显著降低实时特征任务的开发门槛、缩短实时特征的上线周期。有了 PyFlink，算法团队的同学完全可以自己来开发实时特征任务。同时，在特征任务开发的过程中，可以使用各种 Python 库，没有任何限制。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/fb583c1b-64e9-4a16-8397-1771388b0fb0.jpeg?raw=true)

在推荐场景中，可能会用到这样一个特征，计算用户最近 5 分钟的访问物品列表。

为此，PyFlink 提供了多种实现手段。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/3925a0eb-169f-4af9-91c4-3feae014b277.jpeg?raw=true)

首先，用户可以通过 SQL+Pandas UDAF 的方式来实现上述功能。

上述 SQL 语句定义了一个长度为 5 分钟、步长为 30 秒的滑动窗口，针对窗口中的数据，定义了一个 Pandas UDAF 来计算用户在这个窗口中的访问序列。Pandas UDAF 的主要逻辑是对窗口中用户访问的 item 进行排序，并使用｜作为分隔符，生成访问序列字符串。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/490eb2f9-17cc-4733-b261-48ef8414a207.jpeg?raw=true)

除此之外，用户可以通过 DataStream API 计算序列特征，实现上述功能。通过使用 DataStream API，定义了一个窗口大小为 5 分钟、步长为 30 秒的滑动窗口，并定义了一个聚合函数来处理每一个窗口中的数据。聚合函数需要实现 create_accumulator、add、get_result、merge，定义如何针对窗口中的数据进行聚合运算。

针对窗口中的每一条数据，框架会依次调用聚合函数的 add 方法，当窗口中所有的数据都处理完后，框架会调用聚合函数的 get_result 方法来获得聚合值，因此用户只需要根据业务逻辑的需要实现这几个方法即可。

在 add 方法中，我们将数据缓存起来，在 get_result 中对于所有数据进行排序，并以｜作为分隔符，生成访问序列字符串。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/43c2c283-1efd-49af-bb24-3058671acd94.jpeg?raw=true)

接下来，我们来看一下实时样本生成部分。在实时样本生成部分，主要有正负样本判断和特征拼接。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/4173bcd7-c232-476a-8366-0a4513c5d22a.jpeg?raw=true)

首先，我们看一下正负样本的构造。在推荐场景中，当给用户推荐了一批 item 之后，如果用户点击了某个 item，就会成为一个正样本，而如果用户没有点击，则成为一个负样本。

在离线场景中，判别正负样本是非常容易的，而在实时场景中，就不那么容易了。给用户展现了某个 item 之后，用户有可能不点击，也有可能隔了很久之后才点击。

在 Flink 中，用户可以通过定时器解决正负样本问题。针对每条曝光事件，用户可以注册一个定时器。定时器的时间间隔，可以根据业务的需要确定。

在这个例子中，我们定义了一个 10 分钟的定时器。在 10 分钟内，如果收到了这条曝光事件对应的点击事件，则可以将其看成是一个正样本，否则，如果在 10 分钟内还没有收到对应的点击事件，则可以将其看成一个负样本。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/79856aea-ee1c-4713-a70a-108407eeccad.jpeg?raw=true)

正负样本的问题解决了之后，样本的标签也就确定了，接下来，还需要拼接上推荐发生时所用到的特征，才能成为一条完整的样本。

这里，我们可以使用 Flink 中的维表 Join 功能，来进行特征的拼接。为了解决特征穿越问题，也就是说在拼接用户的实时特征时，用户的实时特征相比推荐发生时可能已经发生了变化，前面我们提到，在线推理服务在推荐时，可以将所用到的实时特征保存到历史特征库中。为了便于区分特征的版本，可以给特征加一个唯一的标识，比如 trace_id，然后在做特征拼接时，通过 trace_id 来定位推荐发生时所使用的特征，解决特征穿越的问题。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/2fc65902-c2ce-4b6a-8495-d9eef1aa3203.jpeg?raw=true)

接下来，我们来看一下近线推理。近线推理是非常典型的应用场景。目前，很多用户在用 PyFlink 做近线推理。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/12c9b005-542d-4079-8c7d-a78deace3ec6.jpeg?raw=true)

首先，用户可以通过 Table API 做近线推理。在 Table API 里，用户可以通过 Select 语句做近线推理。推理逻辑可以封装在用户的自定义函数中。在自定义函数里，用户可以通过 open 方法，加载机器学习模型。

open 方法只会在作业启动阶段调用一次，因此可以确保机器学习模型只 load 一次。实际的预测逻辑可以定义在 eval 方法中。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/01fa0210-b42c-41cc-9e00-18c43240e5b1.jpeg?raw=true)

除了 Table API 之外，用户也可以通过 DataStream API 做近线推理。跟 Table API 类似，DataStream API 中的自定义函数中也提供了一个 open 方法。用户可以在 open 方法里，加载机器学习模型。使用方式跟 Table API 比较像，用户可以根据自己的需求，选择使用 Table API，还是 DataStream API。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/01efa817-3741-46b0-b491-4377eedbd2b8.jpeg?raw=true)

除此之外，用户可以通过 timer 提升推理的时效性。在某些场景中，为了提高时效性，可以通过定时器来做周期性推理。该方法适用于活跃用户的范围比较确定，且用户访问比较频繁的场景。

在这些场景中，可以针对活跃用户，或者圈选一批重点用户，周期性地进行近线推理，以进一步提升推荐效果。在这里，我们每 5 分钟对于活跃用户进行一次近线推理。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/f4a5c969-3959-48cb-a7ca-3c05fa1c9415.jpeg?raw=true)

某些公司可能是 Java 技术栈。算法团队训练出模型之后，由开发团队再去负责部署使用。在这种情况下，用户可能会倾向于使用 Flink 的 Java API 进行预测。

在 PyFlink 支持线程模式的过程中，抽象出了一个 library PEMJA，支持 Java 和 Python 之间的互调用，跟其他的 Java/Python 互调用库相比，PEMJA 的性能更好，并且对于各种 Python 库的支持也比较好，兼容所有的 Python 库。

这个例子展示了如何利用 PEMJA 提供的 API，在 Flink Java 作业中加载机器学习模型、并进行预测。从这个例子可以看出，通过 PEMJA，用户可以在 JAVA 代码中调用并执行 Python 代码。

## 四、PyFlink 下一步发展规划

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/a47c8464-6d01-4e98-af40-945d17c1ef41.jpeg?raw=true)

接下来，PyFlink 的建设重点，会逐步从功能以及性能，转向易用性、稳定性以及文档，帮助用户更好的使用 PyFlink。我们接下来会重点完善以下几个方面。

首先，由于当前 PyFlink 的端到端示例相对来说还比较少，不利于新用户快速上手。接下来，我们会建设一个独立的 PyFlink 网站，结合具体场景，展示更多的端到端使用示例。

其次，在易用性方面，接下来会重点优化作业执行过程中的报错提示，让报错信息更友好，使用户在开发作业的过程中更容易定位问题。

与此同时，我们也在重构当前 Python API 的文档。这块主要是参考一些其他成熟的 Python 项目的经验，比如 Pandas。使得用户在 Python API 文档中，更容易找到 PyFlink 中各个 API 的使用方式。

最后，作业运行的稳定性也非常重要。我们也会持续改进并加强 PyFlink 作业运行的稳定性，比如降低 PyFlink 作业在进程模式下 checkpoint 的耗时等。

最后，也欢迎大家加入 【PyFlink 交流群】交流和反馈 PyFlink 相关的问题和想法。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/fdbf1315-525f-4fc7-8587-c6391b6e083b.png?raw=true)

**[点击查看直播回放 & 演讲 PPT](https://flink-learning.org.cn/activity/detail/f30571911e47478ddf4047eeb518d796?name=activity&tab=Meetup&page=1)**

* * *

### 更多内容

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/c6efa55a-1c0d-4520-8c62-529293141b9b.png?raw=true)

* * *

**活动推荐**

阿里云基于 Apache Flink 构建的企业级产品 - 实时计算 Flink 版现开启活动：  
99 元试用 [实时计算 Flink 版](https://www.aliyun.com/product/bigdata/sc)（包年包月、10CU）即有机会获得 Flink 独家定制卫衣；另包 3 个月及以上还有 85 折优惠！  
了解活动详情：[https://www.aliyun.com/product/bigdata/sc](https://www.aliyun.com/product/bigdata/sc)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-13%2016-26-33/ac9aa94d-91ff-4bb1-9c1b-fd56fb9493a3.jpeg?raw=true)
