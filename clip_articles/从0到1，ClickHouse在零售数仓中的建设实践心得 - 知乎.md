# 从0到1，ClickHouse在零售数仓中的建设实践心得 - 知乎
[从 0 到 1，ClickHouse 在零售数仓中的建设实践心得 - 知乎](https://zhuanlan.zhihu.com/p/506379527?utm_id=0) 

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/734177a1-ca36-4f32-93b4-8e78df969126.jpeg?raw=true)

投稿作者：王春波 内容来源：作者授权发布 出品平台：DataFunTalk  

**导读：** 本次分享的是在国内某头部体育用品企业的主品牌数据仓库建设中积累的宝贵经验，项目背景是真实的，作者作为技术经理全程参与数据仓库设计、开发、测试和优化工作，由于身份是乙方，所以这里就不暴露企业名称了。项目基于 Hive on Spark 搭建数据仓库，完成数据的抽取、转换，最后通过 DataX 将数据同步到 ClickHouse 用于 BI 报表、自助分析和移动 Web 查询。

该企业在启动主品牌数据仓库前，分布在其它品牌数据仓库上，在上面尝试了多个技术方案，包括基于 Greenplum+Oacle 模式、Hive+Kylin 模式、Hive+Doirs 模式的方案，但是以上三个项目最后查询的数据量都是千万级别到亿级，而主品牌则是十亿级的。

根据已有项目经验发现：

① 基于 MPP 架构的 Greenplum 虽然查询高效，但不方便进行存算分离，并发查询也不高；

② 基于 Kylin 模式的查询虽然性能快，但模型构建不太灵活，针对零售 BI 场景，预先构建 Cube 的难度非常大；

③ 基于 Doirs 模式的查询引擎虽然性能足够快，但是集群节点要求比较多，项目选型的时候还不支持 DataX 插件 (2021 年 9 月底才发布 DorisWriter 插件)。

所以，我们最后决定选用业界最主流的 Hive + ClickHouse 架构。  

## 01 系统架构

本次项目基于集团自研大数据平台进行开发，由系统架构部提供大数据平台支撑，我公司完成数据仓库模型构建和集市层数据加工，基于 Hive SQL on Spark 完成数据加工，基于自研工具完成数据抽取，基于 DataX 完成数据从 Hive 到 ClickHouse 的同步。前端展现分为移动端和 PC 端，移动端采用自研平台，进行模块化开发，PC 端使用集团采购的商业化软件观远 BI 来展现。整体系统架构如下图：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/6acf7440-7fa3-46dc-9039-f2091390d270.jpeg?raw=true)

本次项目对接的下游系统包括新零售、欧宝、EBI、P60、P61、VOS 和 MDM 等系统，主要抽取销售模型、库存明细（包括在途）、店仓和商品主数据等。数据抽取有增量和全量两种方式，采用大数据平台提供的模板化功能完成数据的同步。零售仓库采用标准分层，按照 ODS、DWD、DWS、ADS 划分，数据的加工全部由 Hive 脚本来完成，通过 git 进行代码版本管理，Hive SQL 的执行由大数据平台自动提交到 Spark on Yarn 上。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/52af1f97-28b4-4239-9e24-f5e2b482ad37.jpeg?raw=true)

本次项目的集市层采用 ClickHouse（以下简称 CK）作为查询引擎，数据在 Hive 中完成加工后通过 DataX 同步到 ClickHouse。项目按照模块分为移动端和 PC 端，分别创建 zy_mbi 和 zy_pcbi 两个数据库，用于集市数据同步和查询。ClickHouse 集群由 4 个节点（320G 内存 84 核 CPU）组成物理集群，并划分成 2 个逻辑集群，其中，Nginx 或 CHProxy 用作数据读写时的负载均衡；ClickHouse 在数据副本的同步会依赖 Zookeeper 来存储元信息和协调。CK 集群的结构如下图：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/699ac3ef-7af9-4ae4-8ace-91e8731bfb41.jpeg?raw=true)

两个逻辑集群的设定如下：

-   **单分片集群**

单分片集群由 4 个节点组成对等服务，互为备份。当某张表存储在单分片集群，每个节点都存储该表的全量数据，各节点之间的数据会相互同步，往任何一个节点写入数据，都会同步到其它节点中。维表和事实表（10 亿以内），建议存储在单分片集群。

-   **多分片集群**

多分片集群，将 4 个节点切分成 2 个主节点 2 个备份节点，每个节点存储单表 1/2 的数据。当某张表存储在多分片集群时，相当于将该表做 sharding，每个分片存储该表的一部分数据，每个分片内的节点数据会自动同步。事实表（10 亿以上）的数据可以存储在多分片集群中。逻辑集群的划分根据配置文件 metrika.xml 进行配置：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/3fca22b7-f8ee-4ceb-ae60-b06b4c6980c2.jpeg?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/ef5668b7-a2f0-447c-8610-04217103be0f.jpeg?raw=true)

在实际的项目应用和性能测试中，我们发现多分片集群在 10 亿以下数据量并且有关联查询的场景下比单分片集群慢很多（10 倍以上的性能差异），因此本次项目的所有的表都部署在单分片集群上，并且由于单分片集群有 4 个节点，通过 Nginx 进行负载均衡，并发查询可以达到四倍单节点的效果，在一定程度上缓解了 CK 并发能力不强的问题。

## **02** **建表规范**

在实际项目中，我们在建表方面遵循以下要求：

①每张需要同步的表，我们都创建三张对应表，一张 local 表、一张 local_tmp 表、一张跨节点的视图；

②表名和字段名均采用小写；

③视图命名以\_v 结尾；

④字段类型尽快简单统一，项目主要采用 Date、Timestamp、String、Int、Decimal(38,4) 五种数据类型，分别对应 Hive 的 date、timestamp、string、int 和 decimal(38,4) 类型；

⑤ local 表和 local_temp 表采用 ReplicatedMergeTree 引擎，跨节点的视图采用 Distributed 引擎。

⑥所有的表分为单分区表和多分区表两种，多分区表采用按月分区或者按日分区，即 toYYYYMM(order_dt) 和 toYYYYMMDD(inv_dt) 两种，根据数据量大小来确定；

⑦ORDER BY 的字段必须包含该表的业务日期，第二个字段优先选择 CMS_CODE；

⑧index_granularity 统一设置为 8192。

基于以上共识，我们就有了标准建表模板，所有的新增表和修改表操作都参照该目标进行，可以极大的提高生产效率。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/410565dd-01e8-4b78-a348-25519a41d36b.jpeg?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/9ed694cf-c97d-48cd-8042-ecec523d331d.jpeg?raw=true)

## **03\*\***数据同步 \*\*

前面说到，我们的建表模板需要创建三张表，可能有人会好奇，为什么需要创建 local_tmp 表。看完这一部分，读者就明白了。由于数据同步可能会失败，并且有可能用户正在使用相关报表，这个时候删除数据再插入数据是有一个时间间隔的，为了避免用户那边出现查询不到数据或者查询的数据不完整的情况，我们就需要用到 local_tmp 表了。即先将 Hive 数据同步到 local_tmp 表然后在库内将数据迁移到 local 表，而页面查询则是基于 local 表之上的跨节点视图。库存数据迁移还分为两种情况，一种是全量替换，我们可以直接通过 rename 表来实现。前置和后置 SQL 如下：  

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/ce4175bc-1157-4f3f-b966-74b462aad938.jpeg?raw=true)

平台自动生成的 DataX 同步配置如下：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/04cdce6a-0c9b-49a7-99bc-7959e4535e97.jpeg?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/fe773ecd-4e1c-4e7a-8a69-76f766cbabdd.jpeg?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/7be43419-63c6-49aa-8d14-409ad9362632.jpeg?raw=true)

第二种是部分数据替换，早期我们尝试过采用先 delete 后插入的方式，后面统一调整成为分区替换的方式。我们利用 ClickHouse 支持分区的特性，将 local_tmp 表和 local 表创建成为多分区的表，按月或者按日创建分区，然后通过 ClickHouse 的分区替换功能交换两张表的分区数据，这样就实现了数据的快速更新。分区替换模式需要在前置 SQL 里面清空目标表。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/54cd68b4-2ebd-485f-a174-eef98171944b.jpeg?raw=true)

然后通过其它程序实现分区替换。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/6f49ba6d-7d6e-44aa-802b-ab6e25c94341.jpeg?raw=true)

实现分区替换的逻辑其实非常简单，主要包括三个步骤：

第一步，查询表的分区信息。

SELECT partition,name,part_type,active,rows,bytes_on_disk,data_compressed_bytesfrom system.parts where table='dm_zy_pcbi_offline_stock_sale_local_tmp';

第二步，检查不同节点分区的数据是否一致。由于数据同步到 ClickHouse 以后，ClickHouse 还需要进行节点间的数据复制，如果数据未复制完成就进行分区替换，结果会出现异常。这里就需要连上每一个实例节点，查询对应表的分区记录数。

select \`partition\`,sum(\`rows\`) from \`system\`.partswhere table ='dm_zy_pcbi_offline_stock_sale_local_tmp'--and \`partition\` ='202202'group by \`partition\`

第三步，从小到大执行分区替换。

ALTER TABLE dm_zy_pcbi_offline_stock_sale_local REPLACE PARTITION 202011 FROM dm_zy_pcbi_offline_stock_sale_local_tmp

总结一下，通过 local_tmp 表替换数据，有以下好处：

①清空 local_tmp 表对用户使用数据无影响，数据同步过程无感知；  

②重命名表或者分区替换操作属于库内操作，时间非常短，用户无感知；

③数据同步失败的情况下，不影响报表使用上一个版本的数据；

④通过分区替换的方式，支持增量同步数据。

## **04\*\***实时数据 \*\*

利用 ClickHouse 来完成实时数据接入，有两种方案分别对应 ClickHouse 提供的两种功能。

方案一：直接通过 Flink 写入 ClickHouse。

从技术实现上也有两种方法，第一种是改写 jdbc connector 源码，增加 ck 方言，可以参考阿里云的文档：

[https://help.aliyun.com/document_detail/175749.html](https://link.zhihu.com/?target=https%3A//help.aliyun.com/document_detail/175749.html)。

第二种是直接引入 flink-clickhouse-sink 包，对应的 github 项目地址为：[https://github.com/ivi-ru/flink-clickhouse-sink](https://link.zhihu.com/?target=https%3A//github.com/ivi-ru/flink-clickhouse-sink)。

本次我们项目采用是 FlinkSQL 写入 ClickHouse，采用的是第一种方法。通过 FlinkSQL 读取 kafka 数据，完成双流 jion 后直接写入 ClickHouse。这里省略读取 kafka 的过程和双流 join 的代码，只保留写入 ClickHouse 的模板，简化代码如下：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/31a546ba-7424-436a-855f-9eabc976ac15.jpeg?raw=true)

由于 Interval Join 要数据库支持 update 和 detele，ClickHouse 虽然支持但是语法不正常，所以这个功能没能用起来，后面需要进一步优化。

**方案二：直接通过 ClickHouse 读取 kafka 数据到内部表。** 

第一步，创建 Kafka 引擎表。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/a3950402-3c05-4da5-ad57-fbac11c9a12b.jpeg?raw=true)

第二步，创建 Kafka 引擎表。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/b898db92-efbd-4ca2-8a8e-531bda0aa8ec.jpeg?raw=true)

第三步，创建物化视图，持续不断地从 Kafka 收集数据并通过 SELECT 将数据转换为所需要的格式写入实体存储表。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/378dcb31-b692-44c0-a08a-26b84f53ec6e.jpeg?raw=true)

数据接入以后，还需要对数据进行去重查询。为什么数据接入的时候不去重呢，因为 ClickHouse 唯一支持数据去重的引擎 ReplacingMergeTree 是以分区为单位删除重复数据的。只有在相同的数据分区内重复的数据才可以被删除，而不同数据分区之间的重复数据依然不能被剔除。

这个时候就用到了 ClickHouse 一个比较好用的语法 order by limit 1 by ，等于在一个语句里面实现了 row \_number 去重（row \_numbe 去重需要嵌套两层查询）。

根据实际的业务场景，我们还需要在 ClickHouse 里面进行一些维度表关联，例如关联从 hive 同步过来的店铺和商品映射表，这些逻辑由于比较固定，并且维表数据量也比较大（在 FlinkSQL 中关联出现过内存溢出），所以我们选择创建 ClickHouse 视图来封装这些处理，举例如下：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/99cfbc98-a044-419a-9062-49efd1c5789c.jpeg?raw=true)

目前系统按照方案一构建实时数据，且仅需当日的实时数据，页面查询结果的延时在 1 分钟以内。但是由于不能支持 Interval Join，数据没办法保证 100% 准确性。如果能实现 Interval Join，然后保证数据在 24 小时内可以多次修改，那么数据的准确性会进一步提高。  

另外，根据实际应用情况，在接入实时数据时，Doris 的优势比较明显。首先 Doris 对 FlinkSQL 的支持比较好，可以删除和修改数据；其次 Doris 的 UNIQUE KEY 可以自动完成数据去重；第三，Doirs 对大表 join 支持更好，查询速度比 ClickHouse 更快。

## **05\*\***数据查询 \*\*

至此，所有离线数据和实时数据都已经进入了 ClickHouse，接下来就是 BI 报表和移动 web 查询了。

在 PCBI 方面，我们是通过数据集的方式构建查询结果集的。这里特别需要说到的一个场景是基于任意日期的本同期查询。

针对这个场景，我们做了以下优化：

①本同期数据只保留一份，即按照日期插入目标表的明细数据，通过 union all 查询 + 字段错位来获取本同期数据；

②在 order_dt 字段上增加分区和排序索引，并且 order_dt 作为 order by 字段的第一位；

③明细数据集字段尽可能精简，减少数据存储；

④维度表数据通过 join 来获取，页面会根据查询要求自动裁剪字段。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/cb30ce83-8f66-441c-a103-a15983821f50.jpeg?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/1fa35eb9-9091-49c0-a867-ea71582eab67.jpeg?raw=true)

在移动 BI 方面主要是基于 mybatis 语法完成 SQL 语句的封装和参数的替换。这里的特殊场景就是基于任意日期的日、周、月、年、累计查询。

对此场景做的查询优化有：

①本同期数据只保留一份，即按照日期插入目标表的明细数据，通过 union all 查询 + 字段错位来获取本同期数据；

②通过 mybatis 的条件判断确定数据的过滤条件，增加代码复用率，并且保持都用 biz_dt 过滤；

③多个数据来源的数据拆分成不同的表，都保留相同的数据粒度，分别进行数据同步；

④所有底表的 biz_dt 字段上增加分区和排序索引，并且 biz_dt 作为 order by 字段的第一位；

⑤过滤条件也通过 mybatis 的判断语句添加，实现多种筛选和下钻复用同一个数据查询服务。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/384b3558-8b34-4a95-9592-892628b2c2e4.jpeg?raw=true)

此外，还有实时数据和离线数据的联合查询，这个在 PCBI 和移动 BI 都有应用。

对此场景做的查询优化有：

①通过视图把离线数据和实时数据 union all 到一起；

②查询实时的时候通过参数条件直接跳过离线数据，查询离线数据的时候通过参数条件跳过实时数据；

③离线数据的开始日期、结束日期通过子查询来获取；

④在离线数据的 order_dt 字段上增加分区和排序索引，并且 order_dt 作为 order by 字段的第一位；

⑤在查询的外围嵌套维度表 join，减少 join 的主表数据量。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/d59f9ba8-757c-480d-b05a-9bf1c27d3028.jpeg?raw=true)

在以上 CK 查询优化的基础上，目前 80% 的复杂查询可以在 1s 内返回，95% 的查询在 3s 内完成，剩余少量需要 count distinct 的场景（无法提前预聚合）会稍慢一点。

经过半年的努力，项目完成各项开发目标，跑批和查询性能都达到了项目预期目标，满足上线条件并于 2022 年 3 月底正式上线。这篇 “热气腾腾” 的项目总结，既是对项目的一次复盘，也给想要使用 Hive+ClickHouse 构建数据仓库的朋友一些经验分享。

最后特别感谢 Kenny.Wang 和尘埃在项目过程中给予的指导和帮助，正是双方的密切配合和集思广益，才产出了这么多最佳实践经验，才有了项目的成功交付。

**今天的分享就到这里，谢谢大家。最后，欢迎大家关注 “数据中台研习社” 公众号或者购买《高效使用 Greenplum：入门、进阶与数据中台》。** 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-17%2017-11-36/d006a921-d868-4215-9009-02edc006d6f1.jpeg?raw=true)
