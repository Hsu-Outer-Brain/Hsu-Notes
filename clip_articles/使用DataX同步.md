# 使用DataX同步
[使用 DataX 同步](https://help.aliyun.com/document_detail/124751.htm?spm=a2c4g.11186623.0.0.23a29474Zg95iM#concept-1062540) 

 通过 DataX，您可以将 MySQL 数据库中的全量数据同步到表格存储（Tablestore）的数据表中。DataX 只支持同步全量数据，不支持同步增量数据。

## 前提条件

-   已创建表格存储实例并在实例详情页面获取实例的服务地址（Endpoint）。具体操作，请参见[创建实例](https://help.aliyun.com/document_detail/55211.htm#task472 "实例是表格存储资源管理的基础单元，表格存储对应用程序的访问控制和资源计量均在实例级别完成。通过控制台创建实例后，您可以在实例中创建和管理数据表。")。
-   已创建表格存储数据表，用于存放迁移数据。具体操作，请参见[创建数据表](https://help.aliyun.com/document_detail/55212.htm#concept-55212-zh)。

    **说明** 创建数据表时，建议使用 MySQL 原主键或唯一索引作为表格存储数据表的主键。
-   已获取 AccessKey（包括 AccessKey ID 和 AccessKey Secret），用于进行签名认证。具体操作，请参见[获取 AccessKey](https://help.aliyun.com/document_detail/175967.htm#task-354412 "您可以为阿里云账号（主账号）和 RAM 用户创建一个访问密钥（AccessKey）。在调用阿里云 API 时您需要使用 AccessKey 完成身份验证。")。

## 背景信息

DataX 通过 MySQL 驱动使用 Reader 中的 MySQL 连接串配置，直接发送 SQL 语句获取到查询数据，这些数据会缓存在本地 JVM 中，然后 Writer 线程将这些数据写入到表格存储的表中。更多信息，请参见[DataX](https://github.com/alibaba/DataX/blob/master/introduction.md)。

## 步骤一：下载 DataX

您可以选择下载 DataX 的源代码进行本地编译或者直接下载编译好的压缩包。

-   下载 DataX 的源代码并编译。
    1.  通过 Git 工具执行以下命令下载 DataX 源代码。
    2.  进入到下载的源代码目录后，执行以下命令进行 Maven 打包。

        **说明** 此步骤会在本地编译各种数据源的 Writer 和 Reader，会花费较长的时间，需要耐心等待。

        ```null
        mvn -U clean package assembly:assembly -Dmaven.test.skip=true
        ```

        编译完成后，进入 / target/datax/datax 目录，查看相应的目录。各目录说明请参见下表。

        | 目录     | 说明                                                                |
        | ------ | ----------------------------------------------------------------- |
        | bin    | 存放可执行的 datax.py 文件，是 DataX 工具的入口。                                 |
        | plugin | 存放支持各种类型数据源的 Reader 和 Writer。                                     |
        | conf   | 存放 core.json 文件，文件中定义了一些默认参数值，例如 channel 流控、buffer 大小等参数，建议使用默认值。 |
-   下载编译好的压缩包[DataX 压缩包](https://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/file-manage-files/zh-CN/20220527/jjqn/datax.tar.gz)。

## 步骤二：准备全量导出的 JSON 文件

在 DataX 中 mysqlreader 配置有 querySQL 模式和 table 模式两种模式，请根据实际选择。

-   querySQL 模式（单 task）

    一般用于有条件的数据导出。在此模式下，DataX 不会按照指定的 column、table 参数进行 SQL 的拼接，而是会略过这些配置（如果有）直接执行 querySQL 语句。task 数量固定为 1，因此在此模式下 channel 的配置无多线程效果。

    querySQL 模式的数据导出示例如下：

    ```null
    {
        "job": {

            "content": [
                {
                    "reader": {
                        "name": "mysqlreader", 
                        "parameter": {
                            "username": "username",
                            "password": "password",
                            "connection": [
                                {
                                    "querySql": [ 
                                        "select bucket_name, delta , timestamp ,cdn_in, cdn_out ,total_request from vip_quota where bucket_name='xxx' "
                                    ],
                                    "jdbcUrl": ["jdbc:mysql://192.168.0.8:3306/db1?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true" 
                                    ]
                                }
                            ]
                        }
                    },

                    "writer": {
                        "name": "otswriter",
                        "parameter": {
                            "endpoint":"https://smoke-test.xxxx.ots.aliyuncs.com",
                            "accessId":"xxxx",
                            "accessKey":"xxxx",
                            "instanceName":"smoke-test",
                            "table":"vip_quota",
                            
                            "requestTotalSizeLimitation": 1048576, 
                            "attributeColumnSizeLimitation": 2097152, 
                            "primaryKeyColumnSizeLimitation": 1024, 
                            "attributeColumnMaxCount": 1024, 

                            "primaryKey":[
                                {"name":"bucket_name", "type":"string"},
                                {"name":"delta", "type":"int"},
                                {"name":"timestamp", "type":"int"}
                            ],
                            "column":[
                                {"name":"cdn_in","type":"int"},
                                {"name":"cdn_out","type":"int"},
                                {"name":"total_request","type":"int"}
                            ],
                            "writeMode":"UpdateRow" 
                        }
                    }

                }
            ]
        }
    }
    ```
-   table 模式（多 task）

    在此模式下无需手动编写 select 语句，而是由 DataX 根据 JSON 中的 column、table、splitPk 配置项自行拼接 SQL 语句。观察执行日志如下：![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-30%2009-37-57/9a9e5a99-d614-4649-ad97-678850d93fc9.png?raw=true)

    table 模式的数据导出示例代码如下：

    ```null
    {
        "job": {
            "setting": {
                "speed": {
                     "channel": 3  
                },
                "errorLimit": {
                    "record": 0,
                    "percentage": 0.02
                }
            },
            "content": [
                {
                    "reader": {
                        "name": "mysqlreader",
                        "parameter": {
                            "username": "username",
                            "password": "password",
                            "column": [  
                                "bucket_name",      
                                "timestamp" ,
                                "delta" , 
                                "cdn_in", 
                                "cdn_out" ,
                                "total_request"
                            ],
                            "splitPk": "timestamp",
                            "connection": [
                                {

                                    "table": [
                                        "vip_quota"
                                     ],
                                    "jdbcUrl": ["jdbc:mysql://192.168.1.7:3306/db1"
                                    ]
                                }
                            ]
                        }
                    },

                    "writer": {
                        "name": "otswriter",
                        "parameter": {
                            "endpoint":"https://smoke-test.xxxx.ots.aliyuncs.com",
                            "accessId":"xxx",
                            "accessKey":"xxx",
                            "instanceName":"smoke-test",
                            "table":"vip_quota",
                            "primaryKey":[
                                {"name":"bucket_name", "type":"string"},
                                {"name":"delta", "type":"int"},
                                {"name":"timestamp", "type":"int"}
                            ],
                            "column":[
                                {"name":"haha","type":"int"},
                                {"name":"hahah","type":"int"},
                                {"name":"kengdie","type":"int"}
                            ],
                            "writeMode":"UpdateRow"
                        }
                    }

                }
            ]
        }
    }
    ```

    上述 JSON 文件中定义了一次数据导出导入的数据源信息和部分系统配置。配置主要包括如下两部分：

    -   setting：主要是 speed（与速率、并发相关）和 errorLimit（容错限制）。
        -   channel：个数决定了 reader 和 writer 的个数上限。
        -   splitPk：指定了 splitPk 字段，DataX 会将 MySQL 表中数据按照 splitPk 切分成 n 段。splitPk 的字段必须是整型或者字符串类型。

            由于 DataX 的实现方式是按照 splitPk 字段分段查询数据库表，那么 splitPk 字段的选取应该尽可能选择分布均匀且有索引的字段，例如主键 ID、唯一键等字段。如果不指定 splitPk 字段，则 DataX 将不会进行数据的切分，并行度会变为 1。

            **说明** 为了保证同步数据的一致性，要么不配置 splitPk 字段使用单线程迁移数据，要么确保数据迁移期间停止该 MySQL 数据库的服务。
    -   content：主要是数据源信息，包含 reader 和 writer 两部分。

    **说明** 配置中的 MySQL 应该确保执行 DataX 任务的机器能够正常访问。

## 步骤三：执行同步命令

执行以下命令同步数据。

```null
python datax.py  -j"-Xms4g -Xmx4g" mysql_to_ots.json
```

其中`-j"-Xms4g -Xmx4g"`可以限制占用 JVM 内存的大小。如果不指定，则 DataX 将使用`conf/core.json`中的配置，默认为 1 GB。

## 步骤四：全量同步加速

DataX 的数据同步涉及数据读取、数据交换和数据写入三部分，您可以对每个部分进行优化加速。

-   数据读取

    数据源读取有 table 模式和 querySQL 模式两种模式，选择 table 模式可以实现加速读取。具体操作，请参见[步骤二：准备全量导出的 JSON 文件](#section-2lz-y13-r01)。
-   数据交换

    在数据交换部分，您可以通过以下方面进行同步优化。

    -   JVM 的内存

        发送给 MySQL 数据库 SQL 语句后会得到查询的数据集，并缓存在 DataX 的 buffer 中。除此之外，每个 channel 也维护了自己的 record 队列。如果存在并发，则 channel 的个数越多，也会需要更多的内存。因此您可以考虑指定 JVM 的内存大小参数，即在[步骤三：执行同步命令](#section-82d-436-4zb)中通过`-j`参数来指定 JVM 的内存大小。
    -   channel 的个数和流控参数

        在 conf/core.json 中，控制 channel 的关键参数如下图所示。

        **说明**

        -   一般情况下，channel 队列本身配置的调整并不常见，但是在使用 DataX 时应该注意 byte 和 record 流控参数。这两个参数都是在 flowControlInterval 间隔中采样后根据采样值来决定是否进行流控。
        -   为了提升同步效率，您可以适当提高 channel 的个数来提高并发数，以及调高每个 channel 的 byte 和 record 限制来提高 DataX 的吞吐量。
        -   请综合考虑 channel 的个数和流控参数，保证理论峰值不会对服务器产生过高的压力。

        ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-30%2009-37-57/67f202a5-fcb6-4175-8f80-f28dd9dc417e.png?raw=true)

        详细参数说明请参见下表。

        | 参数           | 描述                                                             |
        | ------------ | -------------------------------------------------------------- |
        | capacity     | 限制 channel 中队列的大小，即最多缓存的 record 个数。                            |
        | byteCapacity | 限制 record 占用的内存大小，单位为字节。默认值为 64 MB，如果不指定此参数，则占用内存大小会被配置为 8 MB。 |
        | byte         | 控流参数，限制通道的默认传输速率，-1 表示不限制。                                     |
        | record       | 控流参数，限制通道的传输记录个数，-1 表示不限制。                                     |

        capacity 和 byteCapacity 两个参数决定了每个 channel 能缓存的记录数量和内存占用情况。如果要调整相应参数，您需要按照 DataX 实际的运行环境进行配置。例如 MySQL 中每个 record 都比较大，那么可以考虑适当调高 byteCapacity，调整该参数时还请同时考虑机器的内存情况。

        ```null
        {
            "core": {  
                "transport": {
                    "channel": {
                        "speed": {
                            "record": 5000,
                            "byte": 102400
                        }
                    }
                }
            },


            "job": {
                "setting": {
                    "speed": {  
                        "record": 10000,
                    },
                    "errorLimit": {
                        "record": 0,
                        "percentage": 0.02
                    }
                },
                "content": [
                    {
                        "reader": {
                              .....
                        },

                        "writer": {
                            .....
                        }

                    }
                ]
            }
        }
        ```
-   数据写入

    Tablestore 是基于 LSM 设计的高性能高吞吐的分布式数据库产品，每一张表都会被切分为很多数据分区，数据分区分布在不同的服务器上，拥有极强的吞吐能力。如果写入能够分散在所有服务器上，则能够利用所有服务器的服务能力，更高速地写入数据，即表分区数量和吞吐能力正相关。

    新建的表默认分区数量都是 1，分区数量会随着表不断写入数据而自动分裂不断增长，但是自动分裂的周期较长。对于新建表，如果需要马上进行数据导入，由于单分区很可能不够用导致数据导入不够顺畅，因此建议在新建表时对表进行预分区，在开始导入数据时即可获得极好的性能，而不用等待自动分裂。

    ```null
    {
        "job": {
            "setting": {
               ....
            },
            "content": [
                {
                    "reader": {
                          .....
                    },

                    "writer": {
                        "name": "otswriter",
                        "parameter": {
                                .......
                            "writeMode":"UpdateRow",
                            "batchWriteCount":100
                        }
                    }

                }
            ]
        }
    }
                
    ```
