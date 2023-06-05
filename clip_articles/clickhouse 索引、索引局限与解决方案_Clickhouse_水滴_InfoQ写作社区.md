# clickhouse 索引、索引局限与解决方案_Clickhouse_水滴_InfoQ写作社区
[clickhouse 索引、索引局限与解决方案_Clickhouse_水滴\_InfoQ 写作社区](https://xie.infoq.cn/article/5cd5fa8d2a361d857a02949a2) 

## 索引

clickhouse 索引详细介绍可以阅读官网[ClickHouse Index Design](https://xie.infoq.cn/link?target=https%3A%2F%2Fclickhouse.com%2Fdocs%2Fen%2Fguides%2Fimproving-query-performance%2Fsparse-primary-indexes%2Fsparse-primary-indexes-design)，这里只概况陈述下 clickhouse 索引原理

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-32-43/d8711183-4347-4c45-81e1-c08338eb2f4d.webp?raw=true)

ClickHouse 会按照主键列的顺序将一组行存储在磁盘上，通过稀疏索引（不是为每一行创建索引，而是为一组数据行（称为颗粒（granule））构建一个索引条目），快速 (通过对索引项进行二分查找) 识别可能匹配查询的行组。然后潜在的匹配行组 (颗粒) 以并行的方式被加载到 ClickHouse 引擎中，以便找到匹配的行。clickhouse 索引局限分析与解决方案就是围绕着**识别可能匹配查询的行组**，因为识别到行组（block）越多，需要解压与加载到 ClickHouse 引擎 block 就越多，就会消耗更多 cpu、磁盘 io 与内存。如上图 clickhouse 索引主要包括 primary.idx、_.mrk、_.bin 文件。

## 索引局限分析

clickhouse 索引局限主要是在多个主键索引，如以下数据结构，通过几个场景观察 sql 执行**耗时**与所扫描**数据量**来对 clickhouse 索引局限进行探索。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-32-43/4743b68a-6b7d-42f7-a0c0-2dd8c31d2185.webp?raw=true)

先看看表数据量

    instance-vx60mbtu :) select count(1) from test.member;

    SELECT count(1)
    FROM test.member

    Query id: f8b62bf2-90b6-46d0-a1f7-fd8162a1a1ea

    ┌─count()─┐
    │ 1699891 │
    └─────────┘

    1 row in set. Elapsed: 0.002 sec

场景一：主键为会员手机号码 、服务门店时，对门店进行数据分析时 sql 耗时与所扫描数据量

    PRIMARY KEY (mobile_no, service_shop_name)

执行以下 sql，通过输出日志信息可以看出此 sql 扫描来所有 block，这是非常消耗性能（因为需要解压所有 block 并且将所有 block 加载到 clickhouse 引擎，很占用磁盘 io 与内存的），所以虽然使用了服务门店创建索引，但对查询性能并没有提升。

    SELECT
        sex_name,
        count(1) AS Count
    FROM test.member_index
    WHERE test.member_index.service_shop_name = 'xx门店'
    GROUP BY sex_name
    ORDER BY sex_name DESC

    Query id: 8d7570ad-2b7b-4900-91cb-0f39d00415d1

    ┌─sex_name─┬─Count─┐
    │ 男       │  2021 │
    │ 未知     │   101 │
    │ 女       │  4086 │
    └──────────┴───────┘

    3 rows in set. Elapsed: 0.067 sec. Processed 1.70 million rows, 74.67 MB (25.30 million rows/s., 1.11 GB/s.)

场景二：主键为所属品牌、会员手机号码，对会员进行数据分析时 sql 耗时与所扫描数据量

    PRIMARY KEY (brand_name、mobile_no)

执行以下 sql，通过输出日志信息可以看出场景二执行耗时与所扫描数据量比场景一减少了十几倍

    SELECT
        brand_name,
        count(1)
    FROM test.member_index
    WHERE test.member_index.mobile_no = 000000
    GROUP BY brand_name
    ORDER BY brand_name DESC

    Query id: 1a0c835b-bc58-4637-b382-375d67dce159

    ┌─brand_name─┬─count()─┐
    │ 品牌一     │       1 │
    │ 品牌二     │       1 │
    └───────────┴─────────┘

    2 rows in set. Elapsed: 0.004 sec. Processed 127.03 thousand rows, 582.59 KB (28.86 million rows/s., 132.35 MB/s.)

虽然场景一与场景二都是对联合主键的一部分但不是第一个键列进行过滤，但两个场景性能表现差异很大。这是因为场景一会员手机号码具有较高的基数，相同的员手机号码值不太可能分布在多个表行和颗粒上。这意味着索引标记的服务门店值不是单调递增的：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-32-43/1848705b-af24-4feb-b1c2-06eba7bb8cd5.webp?raw=true)

而场景二所属品牌具有较低的基数。在这种情况下，相同的所属品牌值很可能分布在多个表行和颗粒上，从而分布在索引标记上。对于具有相同所属品牌的索引标记，索引标记的会员手机号码值按升序排序 (因为表行首先按所属品牌排序，然后按会员手机号码排序)。这使得有效的过滤如下所述：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-32-43/1e6e432e-9a97-4aa6-ba32-e073d76894b6.webp?raw=true)

## 索引局限解决方案

解决索引局限导致场景一问题，可以有以下方案

### 新建一个不同主键的新表

新建一个不同主键的新表 sec_member_index，联合主键为服务门店、会员手机号码

    PRIMARY KEY (service_shop_name,mobile_no)

执行以下 sql，耗时与所扫描数据量明显减少

    SELECT
        sex_name,
        count(1) AS Count
    FROM test.sec_member_index
    WHERE test.sec_member_index.service_shop_name = 'xxx门店'
    GROUP BY sex_name
    ORDER BY sex_name DESC

    Query id: 75be637e-3750-405b-bfee-b5a6b5df383d

    ┌─sex_name─┬─Count─┐
    │ 男       │  2021 │
    │ 未知     │   101 │
    │ 女       │  4086 │
    └──────────┴───────┘

    3 rows in set. Elapsed: 0.004 sec. Processed 16.38 thousand rows, 714.15 KB (3.94 million rows/s., 171.58 MB/s.)

### 创建一个物化视图

创建一个物化视图 mv_member_index，联合主键为服务门店、会员手机号码

    CREATE MATERIALIZED VIEW mv_member_index
    ENGINE = MergeTree()
    PRIMARY KEY (service_shop_name,mobile_no)
    POPULATE
    AS SELECT * FROM member_index;

执行以下 sql，耗时与所扫描数据量明显减少

    SELECT
        sex_name,
        count(1) AS Count
    FROM test.mv_member_index
    WHERE test.mv_member_index.service_shop_name = 'xxx门店'
    GROUP BY sex_name
    ORDER BY sex_name DESC

    Query id: 56bc8eb8-c0ef-40ed-b26b-d19210179620

    ┌─sex_name─┬─Count─┐
    │ 男       │  6063 │
    │ 未知     │   303 │
    │ 女       │ 12258 │
    └──────────┴───────┘

    3 rows in set. Elapsed: 0.004 sec. Processed 24.58 thousand rows, 1.08 MB (5.58 million rows/s., 244.85 MB/s.)

### 增加 projection

创建一个 projection，ORDER BY 服务门店、会员手机号码

    ALTER TABLE member_index
        ADD PROJECTION prj_shopname_mobileno
        (
            SELECT *
            ORDER BY (service_shop_name,mobile_no)
        );

执行以下 sql，耗时与所扫描数据量明显减少

    SELECT
        sex_name,
        count(1) AS Count
    FROM test.member_index
    WHERE test.member_index.service_shop_name = '广州东方宝泰aojo'
    GROUP BY sex_name
    ORDER BY sex_name DESC

    Query id: 04e4d44c-992d-4d52-b86f-d5f421c15944

    ┌─sex_name─┬─Count─┐
    │ 男       │  2021 │
    │ 未知     │   101 │
    │ 女       │  4086 │
    └──────────┴───────┘

    3 rows in set. Elapsed: 0.005 sec. Processed 16.38 thousand rows, 714.15 KB (3.07 million rows/s., 133.93 MB/s.)

> 三个方案区别详细可以阅读官网[Using multiple primary indexes](https://xie.infoq.cn/link?target=https%3A%2F%2Fclickhouse.com%2Fdocs%2Fen%2Fguides%2Fimproving-query-performance%2Fsparse-primary-indexes%2Fsparse-primary-indexes-multiple)不同之处在于，附加表对于查询和插入语句的路由对用户的透明程度。
>
> -   新建一个不同主键的新表查询必须显式地发送给最适合查询的表版本，并且必须显式地插入新数据到两个表中，以保持表的同步
> -   创建一个物化视图，额外的表被隐藏，数据自动在两个表之间保持同步
> -   projection 方式是最透明的选项，因为除了自动保持隐藏的附加表与数据变化同步外，ClickHouse 还会自动选择最有效的表版本进行查询

### 增加 skip index

此方案有一定局限性，需要主键和目标列之间具有很强的相关性，场景一的会员手机号码与服务门店就没有强相关性，就无法使用 skip index。但如果是以下场景，性能提升就很明显

场景三：联合主键为会员手机号码、会员名称，对会员进行数据分析时 sql 耗时与所扫描数据量

    PRIMARY KEY (mobile_no,member_name)

执行以下 sql，性能没有明显变化

    SELECT
        brand_name,
        count(1) AS Count
    FROM test.member_skip_index
    WHERE test.member_skip_index.member_name = 'xxx会员名称'
    GROUP BY brand_name
    ORDER BY brand_name DESC

    Query id: 5d7520a6-a290-4f59-9baf-aa4d8e65766f

    ┌─brand_name─┬─Count─┐
    │ 品牌一     │     1 │
    └────────────┴───────┘

    1 row in set. Elapsed: 0.063 sec. Processed 1.70 million rows, 27.85 MB (27.06 million rows/s., 443.30 MB/s.)

增加 skip index

    ALTER TABLE member_skip_index ADD INDEX bloom_filter_index_1 member_name TYPE bloom_filter GRANULARITY 1

    ALTER TABLE member_skip_index MATERIALIZE INDEX bloom_filter_index_1;

执行以下 sql，耗时与所扫描数据量明显减少

    SELECT
        brand_name,
        count(1) AS Count
    FROM test.member_skip_index
    WHERE test.member_skip_index.member_name = 'xxx会员名称'
    GROUP BY brand_name
    ORDER BY brand_name DESC

    Query id: 29729f86-c632-492a-b133-75d226c33997

    ┌─brand_name─┬─Count─┐
    │ 品牌一     │     1 │
    └────────────┴───────┘

    1 row in set. Elapsed: 0.008 sec. Processed 65.54 thousand rows, 1.07 MB (8.23 million rows/s., 134.46 MB/s.)
