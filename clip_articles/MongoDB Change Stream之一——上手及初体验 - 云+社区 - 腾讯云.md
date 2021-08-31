# MongoDB Change Stream之一——上手及初体验 - 云+社区 - 腾讯云
> **导语**：Change Stream 是 MongoDB 自 3.6 版本就推出的功能，顾名思义，“变更流” 可以对[数据库](https://cloud.tencent.com/solution/database?from=10680)建立一个监听（订阅）进程，一旦数据库发生变更，使用 change stream 的客户端都可以收到相应的通知。使用场景包括多个 MongoDB 集群之间的增量数据同步、高风险操作审计（删库删表）、将 MongoDB 的变更订阅到其他关联系统实现离线分析 / 计算等等。本文作为系列文章的第一篇，尝试简要介绍一下 change stream 以及实践。(未特殊说明，文中内容均基于 MongoDB4.0.3 版本)

## 一、什么是 Change Stream?

Change Stream 可以直译为 "**变更流**"，也就是说会将数据库中的所有变更以流式的方式呈现出来。用户可以很方便地对数据库建立一个监听（订阅）进程，一旦数据库发生变更，使用 change stream 的客户端都可以收到相应的通知。使用场景可以包括但不限于以下几种：

1）多个 MongoDB 集群之间的增量数据同步；

2）高风险操作的审计（删库删表）；

3）将 MongoDB 的变更订阅到其他关联系统实现离线分析 / 计算等等；

以下是一些 change stream 的基本特征：

-   change stream 对于**副本集**和**分片集群**都可用。副本集时，可以在副本集中任意一个成员上建立监听流；分片集群时则只能在 mongos 上建立监听流。
-   使用条件：1）WT 引擎；2）副本集协议为`pv1`；3）4.0 及以前的版本，要求支持[readConcern](https://docs.mongodb.com/v4.0/reference/read-concern/)为`“majority”`。
-   **粒度可调整**，可选择配置在单个表、单个库或者整个集群上。但是无法配置为`admin/local/config`库或者`system.xxx`表。
-   4.0 以后的版本可以指定`startAtOperationTime`来表示在某个特定的时间开始监听 change Stream。但是要求给定的时间点必须在所选择节点的有效 oplog 时间范围中。

## 二、MongoDB Change Stream 演进过程

**v3.6 版本：** 

1.  初期版本，仅支持 collection 维度的订阅，仅支持 insert/update/replace/delete4 种事件；
2.  支持故障恢复；支持 update 操作的全文档查看；

**v4.0 版本：** 

1.  支持更粗粒度的 change stream 订阅，比如 database / 集群粒度；
2.  resumeToken 的类型从`BinData`变为十六进制编码的字符串；
3.  新增了对 drop/rename/dropDatabase 事件的支持；

**v4.2 版本：** 

1.  加入了更多 pipeline 管道操作符的支持，比如`$replaceWith`、`$set`、`$unset`;
2.  如果 pipeline 修改了一个 change events 的`_id`，将会报错；
3.  新增了`startAfter`选项，可以开始一个新的 change stream 监听，与之前的`resumeAfter`互斥；
4.  使用 change stream 不再需要指定`read concern`为`majority`；

**v4.4 版本**(最新)：暂时未见优化

## 三、Change Stream 初体验

### 3.1 mongo shell

操作顺序：

1\. 第一个会话建立 change stream

```

\>db.watch(\[\],{maxAwaitTimeMS:60000})
```

> 注意：上述命令会阻塞整个会话，直到 1 分钟或者有相应的 change event 产生。如果不希望阻塞 shell 的话可以采用显示生成游标的方式： cursor = db.changestream.watch(\[],{maxAwaitTimeMS:60000})cursor.next()....

2\. 第二个会话做 CURD 操作

    \>db.test.insert({a:2})
    \>db.test.update({a:2}, {$set:{"msg":"hello world"}})
    \>db.test.insert({a:1})
    \>db.test.remove({a:1})

    \>db.test.renameCollection("test1")
    \>db.test1.drop()
    \>db.dropDatabase()

3\. 回到第一个会话观察监听结果

    { "\_id" : { "\_data" : "825F156B3F0000000229295A1004C982483732384D28AE57C6500C6018BF46645F696400645F156B3F0DE1FAAEF1B3DF830004" }, "operationType" : "insert", "clusterTime" : Timestamp(1595239231, 2), "fullDocument" : { "\_id" : ObjectId("5f156b3f0de1faaef1b3df83"), "a" : 2 }, "ns" : { "db" : "phoenix", "coll" : "test" }, "documentKey" : { "\_id" : ObjectId("5f156b3f0de1faaef1b3df83") } }
    { "\_id" : { "\_data" : "825F156B5F0000000129295A1004C982483732384D28AE57C6500C6018BF46645F696400645F156B3F0DE1FAAEF1B3DF830004" }, "operationType" : "update", "clusterTime" : Timestamp(1595239263, 1), "ns" : { "db" : "phoenix", "coll" : "test" }, "documentKey" : { "\_id" : ObjectId("5f156b3f0de1faaef1b3df83") }, "updateDescription" : { "updatedFields" : { "msg" : "hello world" }, "removedFields" : \[ \] } }
    { "\_id" : { "\_data" : "825F156B640000000129295A1004C982483732384D28AE57C6500C6018BF46645F696400645F156B640DE1FAAEF1B3DF840004" }, "operationType" : "insert", "clusterTime" : Timestamp(1595239268, 1), "fullDocument" : { "\_id" : ObjectId("5f156b640de1faaef1b3df84"), "a" : 1 }, "ns" : { "db" : "phoenix", "coll" : "test" }, "documentKey" : { "\_id" : ObjectId("5f156b640de1faaef1b3df84") } }
    { "\_id" : { "\_data" : "825F156B6B0000000129295A1004C982483732384D28AE57C6500C6018BF46645F696400645F156B640DE1FAAEF1B3DF840004" }, "operationType" : "delete", "clusterTime" : Timestamp(1595239275, 1), "ns" : { "db" : "phoenix", "coll" : "test" }, "documentKey" : { "\_id" : ObjectId("5f156b640de1faaef1b3df84") } }
    { "\_id" : { "\_data" : "825F156B960000000129295A1004C982483732384D28AE57C6500C6018BF04" }, "operationType" : "rename", "clusterTime" : Timestamp(1595239318, 1), "ns" : { "db" : "phoenix", "coll" : "test" }, "to" : { "db" : "phoenix", "coll" : "test1" }, }
    { "\_id" : { "\_data" : "825F156BC60000000129295A1004C982483732384D28AE57C6500C6018BF04" }, "operationType" : "drop", "clusterTime" : Timestamp(1595239366, 1), "ns" : { "db" : "phoenix", "coll" : "test1" } }{ "\_id" : { "\_data" : "825F156BD200000001292904" }, "operationType" : "dropDatabase", "clusterTime" : Timestamp(1595239378, 1), "ns" : { "db" : "phoenix" } }

    { "\_id" : { "\_data" : "825F156BD200000001292904" }, "operationType" : "invalidate", "clusterTime" : Timestamp(1595239378, 1) }

    no cursor

4\. 意外中止时的恢复

由于某些原因，我们还没消费已获取的 change event，那么可以通过指定`resumeAfter`来恢复对该 changeStream 的订阅。其中`resumeAfter`就是 change stream 结果里的`_id`字段（即 resume token，唯一标志一个 change stream 流中的位置）

    \>db.changestream.watch(\[\],{maxAwaitTimeMS:60000, resumeAfter:{"\_data":"825F156B3F0000000229295A1004C982483732384D28AE57C6500C6018BF46645F696400645F156B3F0DE1FAAEF1B3DF830004"}})

结果：

![](https://ask.qcloudimg.com/developer-images/article/3031654/bchp5gicux.jpg?imageView2/2/w/1620)

change stream 1.jpg

### 3.2 mongo-driver

只有[官方驱动](https://docs.mongodb.com/drivers/)才支持 change stream，使用诸如[mgo](https://gopkg.in/mgo.v2)等旧的第三方驱动是无法使用的。详情可以查看自己所使用的驱动版本及`README`文件。

这里以 mongo-driver go 版本为例，API 使用非常简单：

```
        pipeline :\= mongo.Pipeline{bson.D{{"$match", bson.D{{"$or",
            bson.A{
                bson.D{{"fullDocument.username", "alice"}},
                bson.D{{"operationType", "delete"}}}}},
        }}}
        cs, err :\= coll.Watch(ctx, pipeline)
        require.NoError(t, err)
        defer cs.Close(ctx)
        ok :\= cs.Next(ctx)
        next :\= cs.Current

```

> 其他语言的版本可以参考[open a change stream](https://docs.mongodb.com/manual/changeStreams/#open-a-change-stream)

## 四、Change Stream VS. Tailing Oplog

在`Change Stream`功能出现以前，我们只能通过不断`tailing oplog`的方式来拉取增量的 oplog。两种方式的对比如下：

\| 

对比项

 \| 

Change Stream

 \| 

Tailing Oplog

|  |
|  |

\| 

易用性

 \| 

简单易用，API 友好

 \| 

使用门槛高，需要知道 oplog 的各种格式变化，而且在对 oplog 不断拉取过程中需要使用`tailable`和`awaitData`两个选项

 \|
\| 

故障恢复

 \| 

简单，内核进行统一的进度管理，通过 resumeToken+API 实现故障恢复

 \| 

相对复杂，需要自行管理增量续传，故障时需要记录上一次拉取的 oplog 的 ts 字段并转换为下一次查询的过滤器

 \|
\| 

结果过滤

 \| 

支持多个维度（集群 / 库 / 集合）以及在 server 端的 pipeline 过滤，减少网络传输

 \| 

只能在拉取的 client 端过滤，而且过滤必须进行反序列化操作，带来一定的 CPU 和网络传输消耗

 \|
\| 

update 返回全文档

 \| 

支持，指定`fullDocument:"updateLookup"`即可

 \| 

不支持，对于`$set`的 update 操作需要根据 oplog 中的`_id`再次查询以获取到全文档

 \|
\| 

分片集群适配

 \| 

直接在 mongos 发起 change stream 即可订阅整个集群维度的变更，并且是全局有序的

 \| 

需要针对每个分片单独建立拉取进程，而且可能乱序

 \|
\| 

持久化

 \| 

返回的每一个 event 都是已提交到大多数的，遇到主节点切换的场景也可以保证数据的持久化

 \| 

无法保证 oplog 已提交到大多数节点

 \|
\| 

安全性

 \| 

用户只能在已授权访问的 db 上订阅变更

 \| 

需要有 local 库的读权限

 \|
\| 

**性能**

 \| 

类似

 \| 

类似

 \|

可以看到`change stream`在各个方面都要优于`tailing oplog`。但是注意到，副本集内节点间主从同步使用的依然是`tailing oplog`的方式。为什么呢？各位可以思考一下，欢迎评论区留言。

## 五、待优化

遗憾的是，当前版本（以及官方 4.4 版本）的 change stream 功能依然存在一些待优化的地方，比如：

**1**.**部分 DDL 操作不支持**，比如`create/createIndex/dropIndex/convertToCapped/collMod/emptycapped/`，无法覆盖到所有变更；（**这也是最主要的问题**）

> 其中`convertToCapped`会变成一个非预期的 rename event，并且 size 字段会丢失。

**2**. 如果将`fullDocument`设置为`"updateLookup"`时，会获取到已提交到大多数节点的已更新全文档版本，change stream 中是通过 update 操作中的`_id`来查找到文档当前内容。但是对同一文档短时间内频繁更新时，change stream 收到的`fullDocument`内容可能已经被后续的修改覆盖。换句话说，**这里的`fullDocument`中内容并不是`point-in-time`的**。

**3**. 对于分片集群的 change stream 需要将订阅建立在 mongos 上，为了保证全局有序的变更流结果，从各个分片返回的结果需要在 mongos 侧按时间戳进行排序和聚合处理。一方面在写入量比较大的场景下，change stream 的性能存在瓶颈；另一方面如果分片间写入不均（比如不合理的范围分区分片键导致写入都落到单个分片），会导致 change stream 的返回延迟大幅增加。

**4**. 所有 change stream 的返回文档也受到 **16MB 的**[**文档大小限制**](https://docs.mongodb.com/manual/reference/limits/#BSON-Document-Size)，考虑到指定了`fullDocument`选项会将全文档内容包含在返回文档内，可能会导致变更流返回失败。当然，如果文档本身的大小已经接近 16MB。对其的 insert/update 操作同样也会导致变更流返回失败。

**5**. 分片集群的变更流会将用户自定义的 pipeline 阶段（比如`$match`,`$project`等）放在 mongos 上执行，导致 mongos 到 mongod 之间的网络流量较大。其实这里可以考虑将一部分 pipeline 操作符下放到 mongod 上去执行，减少 mongos 侧合并排序的压力，提升 change stream 的性能。

**6**.**对事务的支持能力尚有欠缺**，尽管 change event 里面有`lsid`字段来标明所在的 transaction，但并不知道某个事件是否为事务中的最后一个操作，也不知道该事务的提交状态。

**7**.**不支持对 config 库的订阅**。但是有部分开发者认为订阅分片集群的元数据状态也是一种合理的变更流使用场景，比如希望知道集群中何时触发了 chunk split，何时触发了 balancer 等等。

至于为何官方没有实现对`create/createIndex/dropIndex`等 DDL 的支持，猜测可能是最初设计时考虑欠缺（比如`drop/rename/dropDatabase`等是 4.0 以版本后才支持的）

在**JIRA**上可以看到相关的提问：

[mongodb create/delete index should trigger change stream](https://jira.mongodb.org/browse/SERVER-46414)

镜像**stackoverflow**提问：

[mongodb create/delete index won't trigger change stream](https://stackoverflow.com/questions/60368424/mongodb-create-delete-index-wont-trigger-change-stream)

官方在 2 月末回应将转给相关的开发团队，但截至笔者撰写此文时，仍然未见对这些 DDL 操作的支持。

## 六、总结

-   Change Stream 提供了简单而强大的订阅集群中修改的能力。
-   对部分 DDL 操作仍然不支持。
-   对分片集群的订阅在写入量很高的情况下，由于为了保证全局有序的排序聚合阶段的存在，性能可能存在瓶颈。
-   Change Stream 的性能相较于 Tailing oplogs 差不多。
-   同一个集群上开启不同维度的多个 change stream，将不可避免地对源集群产生性能影响。

* * *

## 备注

### MongoDB 中的 DDL

DDL 的概念来自关系型数据库的核心语言——`SQL(Structure Query Language)`。按照定义，它分为四类：

-   数据定义语言 DDL
-   数据操纵语言 DML
-   数据查询语言 DQL
-   数据控制语言 DCL

由于习惯的原因，在 NoSQL 数据库中也沿用了上面的说法，以 DML 和 DDL 为主，主要为了区分一般的`insert/update/delete`和其他操作。

在 MongoDB 中，DDL 包括以下几种 (在 oplog 中，其`"op"`字段为`"c"`)：

-   `collMod` : 向集合添加选项或者修改视图定义，比如修改 TTL、指定验证规则等
-   `create`: 创建集合
-   `createIndexes`: 创建索引
-   `convertToCapped`: 变为 capped collection
-   `deleteIndex/deleteIndexes`: 删除索引
-   `dropIndex/dropIndexes`: 删除索引
-   `drop`: 删除集合
-   `dropDatabase`: 删除数据库
-   `emptycapped`: 清空一个 capped collection
-   `renameCollection`: 集合重命名
-   `applyOps`: 用于回放 oplog

### Change Events 解析

从 Change Streams 中能监听到的变更事件，具体字段信息和含义请参考[change events](https://docs.mongodb.com/manual/reference/change-events/#change-stream-output)。

    {
            \_id : { 
                "\_data" : <BinData|hex string\> 
            },
            "operationType" : "<operation>", 
            "fullDocument" : { <document\> }, 
            "ns" : { 
                "db" : "<database>",
                "coll" : "<collection"
            },
            "to" : { 
                "db" : "<database>",
                "coll" : "<collection"
            },
            "documentKey" : { "\_id" : <value\> }, 
            "updateDescription" : { 
                "updatedFields" : { <document\> }, 
                "removedFields" : \[ "<field>", ... \] 
            },
            "clusterTime" : <Timestamp\>, 
            "txnNumber" : <NumberLong\>, 
            "lsid" : { 
                "id" : <UUID\>,
                "uid" : <BinData\>
            }
        }

截止 4.4 版本，change events 主要分为以下几种类型（也就是上面`operationType`的取值范围）：

-   **insert 事件**
-   **update 事件**
-   **replace 事件**
-   **delete 事件**
-   **drop 事件**
-   **rename 事件**
-   **dropDatabase 事件**
-   **invalidate 事件**

invalidate 事件发生时将会关闭 change stream 的游标。需要使用`resumeAfter`选项从 invalidate 事件之前的 change events 来恢复变更流。

### Change Stream 性能

根据下面这个 jira [SERVER-46979](https://jira.mongodb.org/browse/SERVER-46979?)中官方的回复：

`$changeStream`的原始读取速率（不可避免地）比对 oplog 的简单查询要慢。其中 oplog 的`$match`和转换阶段是主要瓶颈，相较于基本查询而言，CPU 消耗更高。但是在典型操作条件和写入工作负载下对`$change stream`和`oplog-Tailing`进行的测试中，两者的性能相似。

`Change Stream`目前是串行执行的，即对每一个变更流只有一个线程来执行 oplog 的获取、过滤和转换工作。未来也不会考虑对每个单独的变更流进行多线程处理，而是考虑将多个不同的变更流工作分解并复用。 
 [https://cloud.tencent.com/developer/article/1711794](https://cloud.tencent.com/developer/article/1711794)
