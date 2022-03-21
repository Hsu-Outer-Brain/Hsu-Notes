# (11条消息) Flink学习6-常见问题_迷路剑客的博客-CSDN博客
-   更多 Flink 系列文章请点击[Flink 系列文章](https://blog.csdn.net/baichoufei90/article/details/105674793)
-   更多大数据文章请点击[大数据好文推荐](https://blog.csdn.net/baichoufei90/article/details/90264272)

## flink no implicits found for parameter evidence9

-   解释：缺少隐式转换。
-   解决：在代码上加入`import org.apache.flink.api.scala._`即可

## Error:(72, 8) value build is not a member of ?0

使用 flink 1.10.0 时报错，代码如下：

```java
val sink: StreamingFileSink[String] = StreamingFileSink
      .forRowFormat(new Path(parameter.get("hdfs-path", "hdfs://xxx"))
        , new SimpleStringEncoder[String]("UTF-8"))
      .withBucketAssigner(new DateTimeBucketAssigner)
      .withRollingPolicy(OnCheckpointRollingPolicy.build())
      .build

```

报错详细信息：

```
Error:(72, 8) value build is not a member of ?0
possible cause: maybe a semicolon is missing before `value build'?
      .build

```

参见[Change to StreamingFileSink in Flink 1.10](http://apache-flink-user-mailing-list-archive.2336050.n4.nabble.com/Change-to-StreamingFileSink-in-Flink-1-10-td34472.html#a34505)

这是一个 bug，已经在 flink 1.10.1 中解决。

## 2.1 ClassCastException: cannot assign instance

使用`bin/flink run -m yarn-cluster ...`方式提交 flink 作业时，报错如下：

```java
Caused by: java.lang.ClassCastException: cannot assign instance of org.apache.commons.collections.map.LinkedMap to field org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumerBase.pendingOffsetsToCommit of type org.apache.commons.collections.map.LinkedMap in instance of org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer010

```

经查，是因为  
。我们提交的 Flink 作业，相关的 class 被动态加载。

可参考[X cannot be cast to X exceptions](https://ci.apache.org/projects/flink/flink-docs-release-1.8/monitoring/debugging_classloading.html#x-cannot-be-cast-to-x-exceptions)

是因为多个不同版本的`org.apache.commons.collections.map.LinkedMap`被不同的 CalssLoader 加载，而这些类被尝试转换为对方。

解决方法是编辑`conf/flink-conf.yaml`，设置`classloader.resolve-order: parent-first`（flink 默认逆置了 ClassLoader，使用 ChildClassLoader 即 user code ClassLoader 来动态加载类。这个选项就关闭了逆置，即优先使用 ParentClassLoader Java Application ClassLoader 动态加载）来关闭逆置或设置`classloader.parent-first-patterns-additional`来单独设置使用`parent-first`的 package。

## 2.2 ClassNotFoundException，NoSuchMethodError,NoClassDefoundError 相关错误

1.  检查 maven scope
2.  检查`$FLINK_HOME/lib`
3.  检查 jobmanager 启动日志，看 jar 包加载顺序，是否有其他相关 jar 被优先加载，导致类冲突。如果是，把该包放到`$FLINK_HOME/lib`，会优先加载

## 3.1 Jobmanager 内存超限被 Kill

之前我们设置的 Jobmanager 总内存大小为`-yjm 2048m`，实际 JobManager Heap 总大小为 1399MB，其他内存还包括元空间、Direct Memory 、 栈、本地栈、程序计数器等部分。

在运行 DataStream 的 StreamingFileSink（On [Yarn](https://so.csdn.net/so/search?q=Yarn&spm=1001.2101.3001.7020)）过程中，突然遇到整个应用直接挂掉的情况。具体在 Yarn 的`FAILED`中可以找到该应用记录，可以看到`Diagnostics`日志如下：

```
Application application_1575939577711_199999 failed 1 times (global limit =2; local limit is =1) due to Attempt recovered after RM restartAM Container for appattempt_1575939577711_199999_000001 exited with exitCode: -104

Failing this attempt.Diagnostics: Container [pid=994899,containerID=container_e28_1575939577711_199999_01_000001] is running beyond physical memory limits. Current usage: 2.0 GB of 2 GB physical memory used; 7.5 GB of 16 GB virtual memory used. Killing container.

Dump of the process-tree for container_e28_1575939577711_199999_01_000001 :
|- PID PPID PGRPID SESSID CMD_NAME USER_MODE_TIME(MILLIS) SYSTEM_TIME(MILLIS) VMEM_USAGE(BYTES) RSSMEM_USAGE(PAGES) FULL_CMD_LINE
|- 994935 994899 994899 994899 (java) 21450780 17798162 7969161216 524011 /usr/local/jdk//bin/java -Xms1448m -Xmx1448m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/test.bin -Dlog.file=/yarn/logs/application_1575939577711_199999/container_e28_1575939577711_199999_01_000001/jobmanager.log -Dlogback.configurationFile=file:logback.xml -Dlog4j.configuration=file:log4j.properties org.apache.flink.yarn.entrypoint.YarnSessionClusterEntrypoint
|- 994899 994897 994899 994899 (bash) 0 0 115818496 297 /bin/bash -c /usr/local/jdk//bin/java -Xms1448m -Xmx1448m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/test.bin -Dlog.file=/yarn/logs/application_1575939577711_199999/container_e28_1575939577711_199999_01_000001/jobmanager.log -Dlogback.configurationFile=file:logback.xml -Dlog4j.configuration=file:log4j.properties org.apache.flink.yarn.entrypoint.YarnSessionClusterEntrypoint 1> /yarn/logs/application_1575939577711_199999/container_e28_1575939577711_199999_01_000001/jobmanager.out 2> /yarn/logs/application_1575939577711_199999/container_e28_1575939577711_199999_01_000001/jobmanager.err

Container killed on request. Exit code is 143

Container exited with a non-zero exit code 143

For more detailed output, check the application tracking page: http://192.168.1.1:1111/cluster/app/application_1575939577711_199999 Then click on links to logs of each attempt.
. Failing the application.

```

随后点击该应用`Logs`查看：  
![](https://img-blog.csdnimg.cn/20200325181602185.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

最后，我们查了监控一直没找到原因（内存一直平稳并未达到 2GB）。网上搜了很久也没找到出错原因。

于是我们从最后一次成功 Checkpoint 处恢复，调大了 JobManager 内存，继续观察。如果还发现问题回来补充。

## 4.1 DataStream

### 4.1.1 StreamingFileSink

可以参考 [Flink 学习 - DataStream-HDFSConnector(StreamingFileSink)](https://blog.csdn.net/baichoufei90/article/details/104009350)的`3.5 常见问题`章节，有关于少类、找不到方法、中文乱码等问题描述

## 4.2 Table & Sql API

### 4.2.1 Flink 1.10 关键字 time 解析报错问题

按照[reserved-keywords](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/table/sql/index.html#reserved-keywords)的说法，我们已经对关键字字段`time`加上了反引号\`\`，但还是有问题，描述如下：

1.  ddl 里面源表包含该字段 time bigint（13 位毫秒时间戳），然后是 ts_field AS TO_TIMESTAMP(FROM_UNIXTIME(time1 / 1000))，WATERMARK FOR ts_field AS ts_field - INTERVAL ‘10’ SECOND
2.  再定义目标表。
3.  insert into 目标表 select TUMBLE_START(ts_field, INTERVAL ‘5’ SECOND) as tumble_start, TUMBLE_END(ts_field, INTERVAL ‘5’ SECOND) as tumble_end from 源表 GROUP BY TUMBLE(ts_field, INTERVAL ‘5’ SECOND);

类似如上，用 ide 调试代码会发现执行时会生成一个临时表，然后 select time from 临时表，这个地方 time 已经不再带反引号了，会导致 calcite 解析报错，说遇到了`time`字段。。。

解决方案是自己改`org.apache.flink.table.planner.calcite.CalciteParser#parse`方法。

### 4.2.2 Flink 1.10 SQL API 时区问题

我在使用时，发现时区全部加了 8。。。导致我的时间戳全部变成了 + 8 小时。

具体原因是 Flink 会大量使用`org.apache.flink.table.dataformat.SqlTimestamp`来转换时间。

比如我 DDL 中有字段如下：

```sql
time1 BIGINT,
ts_field AS TO_TIMESTAMP(FROM_UNIXTIME(time1 / 1000)),
WATERMARK FOR ts_field AS ts_field - INTERVAL '10' SECOND

```

则当消费一条 time1 为 1587047085000 的数据，则会调用`SqlTimestamp#fromLocalDateTime(LocalDateTime dateTime)`方法。

这个时候，时间已经被转为了不带时区的 LocalDateTime，值为`2020-04-16T22:24:45`。看起来貌似还对，但是接下来就出问题了。看看这个 fromLocalDateTime 方法：

```java
public static SqlTimestamp fromLocalDateTime(LocalDateTime dateTime) {
	
	long epochDay = dateTime.toLocalDate().toEpochDay();
	
	long nanoOfDay = dateTime.toLocalTime().toNanoOfDay();
	
	
	long millisecond = epochDay * MILLIS_PER_DAY + nanoOfDay / 1_000_000;
	int nanoOfMillisecond = (int) (nanoOfDay % 1_000_000);

	return new SqlTimestamp(millisecond, nanoOfMillisecond);
}

```

通过上述代码，我们知道，flink 内部将 LocalDateTime 转为了 epochDay 和 nanoOfDay 两部分。其中`epochDay`是日期部分减去 1970 年 1 月 1 日的天数。问题就在于，是当做 UTC 时间处理的，没有考虑我们现在的时区为东八区！

这样一来，日期部分的时间转换后就是`2020-04-16 00:00:00`（UTC），也就是`2020-04-16 08:00:00`（东八区时间）。自然，毫秒时间戳就比时间时间多了八小时（28800000 毫秒）。这就是时区问题根源所在！

这个问题很难解决，个人觉得这个时间就应该将考虑时区后转换得到的时间戳作为统一的衡量标准来转换。

## 4.3 DataSet

-   [Flink error](https://www.jianshu.com/p/da207f9f0467) 
    [https://blog.csdn.net/baichoufei90/article/details/102718487](https://blog.csdn.net/baichoufei90/article/details/102718487)
