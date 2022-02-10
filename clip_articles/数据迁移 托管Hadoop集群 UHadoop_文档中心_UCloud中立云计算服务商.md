# 数据迁移 托管Hadoop集群 UHadoop_文档中心_UCloud中立云计算服务商
## [1. HDFS 迁移](https://docs.ucloud.cn/uhadoop/migration?id=_1-hdfs%e8%bf%81%e7%a7%bb)

#### [拷贝单个目录或文件](https://docs.ucloud.cn/uhadoop/migration?id=%e6%8b%b7%e8%b4%9d%e5%8d%95%e4%b8%aa%e7%9b%ae%e5%bd%95%e6%88%96%e6%96%87%e4%bb%b6)

网络互通的两个 Hadoop 集群中，可执行如下命令，将 nn1 节点所在集群 A 上目录 a.dir 拷贝到 nn2 节点所在集群 B 目的 b.dir 上

    hadoop distcp -i hdfs://nn1:8020/a.dir hdfs://nn2:8020/b.dir

详情参考：[http://hadoop.apache.org/docs/r1.0.4/cn/distcp.html](http://hadoop.apache.org/docs/r1.0.4/cn/distcp.html)

## [2. Hive 迁移](https://docs.ucloud.cn/uhadoop/migration?id=_2-hive%e8%bf%81%e7%a7%bb)

**step1: 设置默认需要导出的 hive 数据库为 defaultDatabase**

在原集群中的任意节点上，新建 “.hiverc” 文件，加入如下内容：

    vi ~/.hiverc
    use defaultDatabase; 

> defaultDatabase 可修改为需要迁移的其它名称

**step2: 创建数据临时目录**

    hdfs dfs -mkdir /tmp/hive-export

**step3: 生成数据导出脚本**

执行如下命令生成数据导出脚本：

    hive -e "show tables" | awk '{printf "export table %s to @/tmp/hive-export/%s@;\n",$1,$1}' | sed "s/@/'/g" > export.sql

**step4: 手工导出数据到 HDFS**

执行脚本导出数据

    hive -f export.sql

**step5: 下载数据**

下载 HDFS 数据到本地，并传送到目标集群（targetDir 为目标集群地址）的 / tmp/hive-export 目录：

    hdfs dfs -get /tmp/hive-export/
    scp -r hive-export/ export.sql root@targetDir
    hdfs dfs -put hive-export/ /tmp/hive-export

**step6: 生成数据导入脚本**

执行如下命令，复制导出脚本，并将脚本修改为导入脚本：

    cp export.sql import.sql
    sed -i 's/export table/import table/g' import.sql
    sed -i 's/ to / from /g' import.sql

**step7: 导入数据**

    hive -f import.sql 

更多内容请参考： [https://cwiki.apache.org/confluence/display/Hive/LanguageManual+ImportExport#LanguageManualImportExport-Examples](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+ImportExport#LanguageManualImportExport-Examples)

## [3. HBase 迁移](https://docs.ucloud.cn/uhadoop/migration?id=_3-hbase%e8%bf%81%e7%a7%bb)

HBase 迁移参考：[http://www.tuicool.com/articles/QJFn22E](http://www.tuicool.com/articles/QJFn22E)

设置主从备份，只拷贝新增数据

**step1: 开启 replication：** 

主从集群 hbase-site.xml 添加（待确认是否默认就为 true，如果默认 true，可省略）

    <property>
        <name>hbase.replication</name>
        <value>true</value>
    </property>

**step2: 在 new cluster 和 old cluster 所有节点加上对方的 hosts**

**step3: 在 new Cluster 中建表和簇名**

**step4: 修改表定义，开启复制功能**

    disable 'your_table'
    alter 'your_table', {NAME => 'family_name', REPLICATION_SCOPE => '1'}
    enable 'your_table'

**step5: 添加 peer**

    add_peer 'ID' 'zk1,zk2,zk3:2181:/hbase'

此 ID 应和上述 REPLICATION_SCOPE 相同

-   可用如下命令停止自动复制功能

    disable_peer 'peer_id'

**step6: 停止原 HBase**

**step7: 迁移原 HDFS 的 HBase 数据**

迁移 HDFS 中 HBase 数据的两种方案如下：

-   利用 HDFS 的 distcp 工具

    hadoop distcp -overwrite hdfs://sourceIP:8020/hbaseDir hdfs://targetIP:8020/hbase
-   通过 scp 拷贝方式传输

    hadoop fs -copyToLocal /hbase hbase-data scp 拷贝到 target 集群 hadoop fs -copyFromLocal hbase-data /hbase

**step8: 恢复元数据与数据**

在新集群执行

    hbase hbck -fixMeta
    hbase hbck -repair

> 详情：hbase hbck -h

## [4. 将数据导入到 hive 并自动分区](https://docs.ucloud.cn/uhadoop/migration?id=_4-%e5%b0%86%e6%95%b0%e6%8d%ae%e5%af%bc%e5%85%a5%e5%88%b0hive%e5%b9%b6%e8%87%aa%e5%8a%a8%e5%88%86%e5%8c%ba)

-   1.  创建目标分区的表。

        hive> create table t_part (id int, name string) partitioned by (stat_date string , province string);
-   2\. 导入外部表。 创建外部表（创建外部表的开销几乎为 0）：


    create external table ext( id int, name string, stat_date string, province string) row format delimited fields terminated by ' ' lines terminated by '\n' stored as textfile location '/data/x.txt';

-   3\. 将分区信息到入到目标表中

    hive> set hive.exec.dynamic.partition=true; hive> set hive.exec.dynamic.partition.mode=nostrict; hive> insert into t_part partition(stat_date, province) select \* from ext;

> 原理：partition 只是相当于对原来的表额外增加了列，所以只要把原始数据按照对应的列拼好一行数据就可以实现自动分区了。

    性能统计：
    环境：2cpu/6G core节点
    数据量：28G   1,138,816,944 行
    时间：
    Total MapReduce CPU Time Spent: 0 days 1 hours 19 minutes 32 seconds 330 msec

## [5. Flume 导数据到 Hive](https://docs.ucloud.cn/uhadoop/migration?id=_5-flume%e5%af%bc%e6%95%b0%e6%8d%ae%e5%88%b0hive)

以下步骤以 flume-1.6.0 为例，[点此下载](http://uhadoop.ufile.ucloud.com.cn/flume/apache-flume-1.6.0-bin.tar.gz)

\*\* 限制说明 \*\*

-   只支持 orc 存储格式的 hive 表
-   支持带有 buckets 的表


```
CREATE EXTERNAL TABLE stocks (
    date STRING,
    open DOUBLE,
    high DOUBLE,
    low DOUBLE,
    close DOUBLE,
    volume BIGINT,
    adj_close DOUBLE)
PARTITIONED BY(year STRING)
CLUSTERED BY (date) into 3 buckets
STORED AS ORC；

```

-   partition 是可选项
-   数据源只支持 csv 和 json 两种格式
-   hivesink 使兼容的版本是 hive1.0.0
-   metastore 增加下面的配置, 然后重启 metastore

    hive.txn.manager = org.apache.hadoop.hive.ql.lockmgr.DbTxnManager hive.compactor.initiator.on = true hive.compactor.worker.threads = 5

\*\* Flume 配置 \*\*

-   flume.conf

    a1.sources = src1 a1.channels = chan1 a1.sinks = sink1 a1.sources.src1.type = spooldir a1.sources.src1.channels = chan1 a1.sources.src1.spoolDir = /root/stk a1.sources.src1.interceptors = skipHeadI dateI a1.sources.src1.interceptors.skipHeadI.type = regex_filter a1.sources.src1.interceptors.skipHeadI.regex = ^Date.\* a1.sources.src1.interceptors.skipHeadI.excludeEvents = true a1.sources.src1.interceptors.dateI.type = regex_extractor a1.sources.src1.interceptors.dateI.regex = ^(\\d+)-.\* a1.sources.src1.interceptors.dateI.serializers = y a1.sources.src1.interceptors.dateI.serializers.y.name = year

    a1.channels.chan1.type = memory a1.channels.chan1.capacity = 1000 a1.channels.chan1.transactionCapacity = 100

    a1.sinks.sink1.type = hive  
    a1.sinks.sink1.channel = chan1

    a1.sinks.sink1.hive.metastore = thrift://ip1:9083,thrift://ip2:9083 a1.sinks.sink1.hive.database = default a1.sinks.sink1.hive.table = stocks a1.sinks.sink1.hive.partition = year a1.sinks.sink1.hive.txnsPerBatchAsk = 2 a1.sinks.sink1.batchSize = 10 a1.sinks.sink1.serializer = delimited a1.sinks.sink1.serializer.delimiter = , a1.sinks.sink1.serializer.fieldnames = date,open,high,low,close,volume,adj_close

> 注意：
>
> a1.sinks.sink1.hive.metastore = thrift:\*ip1:9083,thrift:\*ip2:9083 中的 ip1,ip2 需要修改成具体的 ip

\*\* 下载依赖包 \*\*

[http://mirrors.ucloud.cn/ucloud/udata/hivesink.gz](http://mirrors.ucloud.cn/ucloud/udata/hivesink.gz)

把 jar 包解压到 flume 的 lib 下，启动 flume

示例文件

        Date,Open,High,Low,Close,Volume,Adj Close
    2006-07-21,75.489998,75.50,74.50,74.860001,8372500,59.86873
    2006-07-20,75.730003,75.879997,75.199997,75.480003,12214900,60.364573
    2006-07-19,76.00,77.059998,76.00,76.07,14536900,60.836418

启动命令

    /bin/flume-ng agent -n a1 -f conf/flume-conf

> \-n 指定的是 config 文件中启动的 agent 的名字
>
> \-f 指定了配置文件

最近更新时间：2021-07-07 17:52:46 
 [https://docs.ucloud.cn/uhadoop/migration](https://docs.ucloud.cn/uhadoop/migration)
