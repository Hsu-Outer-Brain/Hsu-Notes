# (8条消息) ClickHouse主键索引最佳实践_大数据技术派的博客-CSDN博客
[(8 条消息) ClickHouse 主键索引最佳实践\_大数据技术派的博客 - CSDN 博客](https://blog.csdn.net/ddxygq/article/details/130437170?spm=1001.2101.3001.6650.3&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-3-130437170-blog-119976038.235%5Ev37%5Epc_relevant_anti_vip_base&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-3-130437170-blog-119976038.235%5Ev37%5Epc_relevant_anti_vip_base&utm_relevant_index=6) 

 在本文中，我们将深入研究[ClickHouse](https://so.csdn.net/so/search?q=ClickHouse&spm=1001.2101.3001.7020)索引。我们将对此进行详细说明和讨论：

-   ClickHouse 的索引与传统的关系数据库有何不同
-   ClickHouse 是怎样构建和使用主键稀疏索引的
-   ClickHouse 索引的最佳实践

这篇文章主要关注稀疏索引，clickhouse 主键使用的就是稀疏索引。

## 数据集

在本文中，我们将使用一个匿名的 web 流量数据集。

-   我们将使用样本数据集中的 887 万行 (事件) 的子集。
-   未压缩的数据大小为 887 万个事件和大约 700mb。当存储在 ClickHouse 时，压缩为 200mb。
-   在我们的子集中，每行包含三列，表示在特定时间 (EventTime 列) 单击 URL (URL 列)的互联网用户(UserID 列)。

通过这三个列，我们已经可以制定一些典型的 web 分析查询，如：

-   某个用户点击次数最多的前 10 个 url 是什么？
-   点击某个 URL 次数最多的前 10 名用户是谁？
-   用户点击特定 URL 的最频繁时间 (比如一周中的几天) 是什么？

## 测试环境

本文档中给出的所有运行时数据都是在带有 Apple M1 Pro 芯片和 16GB RAM 的[MacBook](https://so.csdn.net/so/search?q=MacBook&spm=1001.2101.3001.7020) Pro 上本地运行 ClickHouse 22.2.1。

## 全表扫描

为了了解在没有主键的情况下如何对数据集执行查询，我们通过执行以下 SQL [DDL 语句](https://so.csdn.net/so/search?q=DDL%E8%AF%AD%E5%8F%A5&spm=1001.2101.3001.7020)(使用 MergeTree 表引擎) 创建了一个表：

```null
CREATE TABLE hits_NoPrimaryKey
```

接下来，使用以下插入 SQL 将命中数据集的一个子集插入到表中。这个 SQL 使用 URL 表函数和类型推断从 clickhouse.com 加载一个数据集的一部分数据：

```null
INSERT INTO hits_NoPrimaryKey SELECT   intHash32(c11::UInt64) AS UserID,FROM url('https://datasets.clickhouse.com/hits/tsv/hits_v1.tsv.xz')
```

结果：

```null
0 rows in set. Elapsed: 145.993 sec. Processed 8.87 million rows, 18.40 GB (60.78 thousand rows/s., 126.06 MB/s.)
```

ClickHouse 客户端输出了执行结果，插入了 887 万行数据。

最后，为了简化本文后面的讨论，并使图表和结果可重现，我们使用 FINAL 关键字 optimize 该表：

```null
OPTIMIZE TABLE hits_NoPrimaryKey FINAL;
```

> NOTE

一般来说，不需要也不建议在加载数据后立即执行 optimize。对于这个示例，为什么需要这样做是很明显的。

现在我们执行第一个 web 分析查询。以下是用户 id 为 749927693 的互联网用户点击次数最多的前 10 个 url：

```null
SELECT URL, count(URL) as Count
```

结果：

```null
┌─URL────────────────────────────┬─Count─┐└────────────────────────────────┴───────┘10 rows in set. Elapsed: 0.022 sec.Processed 8.87 million rows,70.45 MB (398.53 million rows/s., 3.17 GB/s.)
```

ClickHouse 客户端输出表明，ClickHouse 执行了一个完整的表扫描！我们的表的 887 万行中的每一行都被加载到 ClickHouse 中，这不是可扩展的。

为了使这种 (方式) 更有效和更快，我们需要使用一个具有适当主键的表。这将允许 ClickHouse 自动 (基于主键的列) 创建一个稀疏的主索引，然后可以用于显著加快我们示例查询的执行。

## 包含主键的表

创建一个包含联合主键 UserID 和 URL 列的表：

```null
CREATE TABLE hits_UserID_URLPRIMARY KEY (UserID, URL)ORDER BY (UserID, URL, EventTime)SETTINGS index_granularity = 8192, index_granularity_bytes = 0;
```

> 为了简化本文后面的讨论，并使图和结果可重现，使用 DDL 语句有如下说明：

-   通过 ORDER BY 子句指定表的复合排序键
-   通过设置配置控制主索引有多少索引项：
-   -   index_granularity_bytes
    -   设置为 0 表示禁止
    -   如果 n 小于 8192，但 n 行的合并行数据大小大于或等于 10MB (index_granularity_bytes 的默认值) 或
    -   n 达到 8192
    -   index_granularity: 显式设置为其默认值 8192。这意味着对于每一组 8192 行，主索引将有一个索引条目，例如，如果表包含 16384 行，那么索引将有两个索引条目。
    -   自适应索引粒度

        。自适应索引粒度意味着 ClickHouse 自动为一组 n 行创建一个索引条目

上面 DDL 语句中的主键会基于两个指定的键列创建主索引。

插入数据：

```null
INSERT INTO hits_UserID_URL SELECT   intHash32(c11::UInt64) AS UserID,FROM url('https://datasets.clickhouse.com/hits/tsv/hits_v1.tsv.xz')
```

结果：

```null
0 rows in set. Elapsed: 149.432 sec. Processed 8.87 million rows, 18.40 GB (59.38 thousand rows/s., 123.16 MB/s.)
```

optimize 表：

```null
OPTIMIZE TABLE hits_UserID_URL FINAL;
```

我们可以使用下面的查询来获取关于表的元数据：

```null
    formatReadableQuantity(rows) AS rows,    formatReadableSize(data_uncompressed_bytes) AS data_uncompressed_bytes,    formatReadableSize(data_compressed_bytes) AS data_compressed_bytes,    formatReadableSize(primary_key_bytes_in_memory) AS primary_key_bytes_in_memory,    formatReadableSize(bytes_on_disk) AS bytes_on_diskWHERE (table = 'hits_UserID_URL') AND (active = 1)
```

结果：

```null
path:                        ./store/d9f/d9f36a1a-d2e6-46d4-8fb5-ffe9ad0d5aed/all_1_9_2/data_uncompressed_bytes:     733.28 MiBdata_compressed_bytes:       206.94 MiBprimary_key_bytes_in_memory: 96.93 KiBbytes_on_disk:               207.07 MiB1 rows in set. Elapsed: 0.003 sec.
```

客户端输出表明：

-   表数据以 wide format 存储在一个特定目录，每个列有一个数据文件和 mark 文件。
-   表有 887 万行数据。
-   未压缩的数据有 733.28 MB。
-   压缩之后的数据有 206.94 MB。
-   有 1083 个主键索引条目，大小是 96.93 KB。
-   在磁盘上，表的数据、标记文件和主索引文件总共占用 207.07 MB。

## 针对海量数据规模的索引设计

在传统的关系数据库管理系统中，每个表行包含一个主索引。对于我们的数据集，这将导致主索引——通常是一个 B(+)-Tree 的数据结构——包含 887 万个条目。

这样的索引允许快速定位特定的行，从而提高查找点查和更新的效率。在 B(+)-Tree 数据结构中搜索一个条目的平均时间复杂度为 O(log2n)。对于一个有 887 万行的表，这意味着需要 23 步来定位任何索引条目。

这种能力是有代价的: 额外的磁盘和内存开销，以及向表中添加新行和向索引中添加条目时更高的插入成本 (有时还需要重新平衡 B-Tree)。

考虑到与 B-Tee 索引相关的挑战，ClickHouse 中的表引擎使用了一种不同的方法。ClickHouseMergeTree Engine 引擎系列被设计和优化用来处理大量数据。

这些表被设计为每秒接收数百万行插入，并存储非常大 (100 pb) 的数据量。

数据被一批一批的快速写入表中，并在后台应用合并规则。

在 ClickHouse 中，每个数据部分（data part）都有自己的主索引。当他们被合并时，合并部分的主索引也被合并。

在大规模中情况下，磁盘和内存的效率是非常重要的。因此，不是为每一行创建索引，而是为一组数据行（称为颗粒（granule））构建一个索引条目。

之所以可以使用这种稀疏索引，是因为 ClickHouse 会按照主键列的顺序将一组行存储在磁盘上。

与直接定位单个行 (如基于 B-Tree 的索引) 不同，稀疏主索引允许它快速 (通过对索引项进行二分查找) 识别可能匹配查询的行组。

然后潜在的匹配行组 (颗粒) 以并行的方式被加载到 ClickHouse 引擎中，以便找到匹配的行。

这种索引设计允许主索引很小 (它可以而且必须完全适合主内存)，同时仍然显著加快查询执行时间：特别是对于数据分析用例中常见的范围查询。

下面详细说明了 ClickHouse 是如何构建和使用其稀疏主索引的。在本文后面，我们将讨论如何选择、移除和排序用于构建索引的表列 (主键列) 的一些最佳实践。

## 数据按照主键排序存储在磁盘上

上面创建的表有：

-   联合主键 (UserID, URL)
-   联合排序键 (UserID, URL, EventTime)。

> NOTE

-   如果我们只指定了排序键，那么主键将隐式定义为排序键。
-   为了提高内存效率，我们显式地指定了一个主键，只包含查询过滤的列。基于主键的主索引被完全加载到主内存中。
-   为了上下文的一致性和最大的压缩比例，我们单独定义了排序键，排序键包含当前表所有的列（和压缩算法有关，一般排序之后又更好的压缩率）。
-   如果同时指定了主键和排序键，则主键必须是排序键的前缀。

插入的行按照主键列 (以及排序键的附加 EventTime 列) 的字典序 (从小到大) 存储在磁盘上。

> NOTE

ClickHouse 允许插入具有相同主键列的多行数据。在这种情况下 (参见下图中的第 1 行和第 2 行)，最终的顺序是由指定的排序键决定的，这里是 EventTime 列的值。

如下图所示：ClickHouse 是列存数据库。

-   在磁盘上，每个表都有一个数据文件 (\*.bin)，该列的所有值都以压缩格式存储，并且
-   在这个例子中，这 887 万行按主键列 (以及附加的排序键列) 的字典升序存储在磁盘上

    -   UserID 第一位，
    -   然后是 URL，
    -   最后是 EventTime：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/40b3608a-e90b-47d1-8dc5-63d8f3e5056c.png?raw=true)
UserID.bin，URL.bin，和 EventTime.bin 是 UserID，URL，和 EventTime 列的数据文件。

> NOTE

-   因为主键定义了磁盘上行的字典顺序，所以一个表只能有一个主键。
-   我们从 0 开始对行进行编号，以便与 ClickHouse 内部行编号方案对齐，该方案也用于记录消息。

## 数据被组织成颗粒以进行并行数据处理

出于数据处理的目的，表的列值在逻辑上被划分为多个颗粒。颗粒是流进 ClickHouse 进行数据处理的最小的不可分割数据集。这意味着，ClickHouse 不是读取单独的行，而是始终读取 (以流方式并并行地) 整个行组（颗粒）。

> NOTE

列值并不物理地存储在颗粒中，颗粒只是用于查询处理的列值的逻辑组织方式。

下图显示了如何将表中的 887 万行 (列值) 组织成 1083 个颗粒，这是表的 DDL 语句包含设置 index_granularity(设置为默认值 8192)的结果。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/e6846230-f564-4ba8-a05a-eecab11e8504.png?raw=true)

img

第一个 (根据磁盘上的物理顺序)8192 行(它们的列值) 在逻辑上属于颗粒 0，然后下一个 8192 行 (它们的列值) 属于颗粒 1，以此类推。

> NOTE

-   最后一个颗粒（1082 颗粒）是少于 8192 行的。
-   我们在本指南开头的 “DDL 语句详细信息” 中提到，我们禁用了自适应索引粒度（为了简化本指南中的讨论，并使图表和结果可重现）。

    因此，示例表中所有颗粒（除了最后一个）都具有相同大小。
-   对于具有自适应索引粒度的表（默认情况下索引粒度是自适应的），某些粒度的大小可以小于 8192 行，具体取决于行数据大小。
-   我们将主键列 (UserID, URL) 中的一些列值标记为橙色。

    这些橙色标记的列值是每个颗粒中每个主键列的最小值。这里的例外是最后一个颗粒 (上图中的颗粒 1082)，最后一个颗粒我们标记的是最大的值。

    正如我们将在下面看到的，这些橙色标记的列值将是表主索引中的条目。
-   我们从 0 开始对行进行编号，以便与 ClickHouse 内部行编号方案对齐，该方案也用于记录消息。

## 每个颗粒对应主索引的一个条目

主索引是基于上图中显示的颗粒创建的。这个索引是一个未压缩的扁平数组文件 (primary.idx)，包含从 0 开始的所谓的数字索引标记。

下面的图显示了索引存储了每个颗粒的最小主键列值 (在上面的图中用橙色标记的值)。 例如：

-   第一个索引条目 (下图中的“mark 0”) 存储上图中颗粒 0 的主键列的最小值，
-   第二个索引条目 (下图中的“mark 1”) 存储上图中颗粒 1 的主键列的最小值，以此类推。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/41fcc035-f657-44be-86ac-ef5d336fba34.png?raw=true)

img

在我们的表中，索引总共有 1083 个条目，887 万行数据和 1083 个颗粒:

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/66ba6656-402b-4bd8-8042-40c000a1a390.png?raw=true)

img

> NOTE

-   最后一个索引条目 (上图中的“mark 1082”) 存储了上图中颗粒 1082 的主键列的最大值。
-   索引条目 (索引标记) 不是基于表中的特定行，而是基于颗粒。例如，对于上图中的索引条目‘mark 0’，在我们的表中没有 UserID 为 240.923 且 URL 为 “goal://metry=10000467796a411…” 的行，相反，对于该表，有一个颗粒 0，在该颗粒中，最小 UserID 值是 240.923，最小 URL 值是“goal://metry=10000467796a411…”，这两个值来自不同的行。
-   主索引文件完全加载到主内存中。如果文件大于可用的空闲内存空间，则 ClickHouse 将发生错误。

主键条目称为索引标记，因为每个索引条目都标志着特定数据范围的开始。对于示例表:

-   UserID index marks: 主索引中存储的 UserID 值按升序排序。 上图中的‘mark 1’指示颗粒 1 中所有表行的 UserID 值，以及随后所有颗粒中的 UserID 值，都保证大于或等于 4.073.710。

    正如我们稍后将看到的, 当查询对主键的第一列进行过滤时，此全局有序使 ClickHouse 能够对第一个键列的索引标记使用二分查找算法。
-   URL index marks: 主键列 UserID 和 URL 有相同的基数，这意味着第一列之后的所有主键列的索引标记通常只表示每个颗粒的数据范围。 例如，‘mark 0’中的 URL 列所有的值都大于等于 goal://metry=10000467796a411...， 然后颗粒 1 中的 URL 并不是如此，这是因为‘mark 1‘与‘mark 0‘具有不同的 UserID 列值。

    稍后我们将更详细地讨论这对查询执行性能的影响。

## 主索引被用来选择颗粒

现在，我们可以在主索引的支持下执行查询。

下面计算 UserID 749927693 点击次数最多的 10 个 url。

```null
SELECT URL, count(URL) AS Count
```

结果：

```null
┌─URL────────────────────────────┬─Count─┐└────────────────────────────────┴───────┘10 rows in set. Elapsed: 0.005 sec.Processed 8.19 thousand rows,740.18 KB (1.53 million rows/s., 138.59 MB/s.)
```

ClickHouse 客户端的输出显示，没有进行全表扫描，只有 8.19 千行流到 ClickHouse。

如果 trace logging 打开了，那 ClickHouse 服务端日志会显示 ClickHouse 正在对 1083 个 UserID 索引标记执行二分查找以便识别可能包含 UserID 列值为 749927693 的行的颗粒。这需要 19 个步骤，平均时间复杂度为 O(log2 n)：

```null
...Executor): Key condition: (column 0 in [749927693, 749927693])...Executor): Running binary search on index range for part all_1_9_2 (1083 marks)...Executor): Found (LEFT) boundary mark: 176...Executor): Found (RIGHT) boundary mark: 177...Executor): Found continuous range in 19 steps...Executor): Selected 1/1 parts by partition key, 1 parts by primary key,1/1083 marks by primary key, 1 marks to read from 1 ranges...Reading ...approx. 8192 rows starting from 1441792
```

我们可以在上面的跟踪日志中看到，1083 个现有标记中有一个满足查询。

```null
Mark 176 was identified (the 'found left boundary mark' is inclusive, the 'found right boundary mark' is exclusive), and therefore all 8192 rows from granule 176 (which starts at row 1.441.792 - we will see that later on in this article) are then streamed into ClickHouse in order to find the actual rows with a UserID column value of 749927693.
```

我们也可以通过使用 EXPLAIN 来重现这个结果：

```null
SELECT URL, count(URL) AS Count
```

结果如下：

```null
┌─explain───────────────────────────────────────────────────────────────────────────────┐│ Expression (Projection)                                                               ││   Limit (preliminary LIMIT (without OFFSET))                                          ││     Sorting (Sorting for ORDER BY)                                                    ││       Expression (Before ORDER BY)                                                    ││           Expression (Before GROUP BY)                                                ││               SettingQuotaAndLimits (Set limits and quota after reading from storage) ││                     Condition: (UserID in [749927693, 749927693])                     │└───────────────────────────────────────────────────────────────────────────────────────┘16 rows in set. Elapsed: 0.003 sec.
```

客户端输出显示，在 1083 个颗粒中选择了一个可能包含 UserID 列值为 749927693 的行。

> **CONCLUSION**
>
> 当查询对联合主键的一部分并且是第一个主键进行过滤时，ClickHouse 将主键索引标记运行二分查找算法。

正如上面所讨论的，ClickHouse 使用它的稀疏主索引来快速 (通过二分查找算法) 选择可能包含匹配查询的行的颗粒。

这是 ClickHouse 查询执行的**第一阶段 (颗粒选择)**。

在**第二阶段 (数据读取中)**, ClickHouse 定位所选的颗粒，以便将它们的所有行流到 ClickHouse 引擎中，以便找到实际匹配查询的行。

我们将在下一节更详细地讨论第二阶段。

## 标记文件用来定位颗粒

下图描述了上表主索引文件的一部分。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/7f4f19d9-a38c-4a47-9c6f-888e7d0ea6c3.png?raw=true)

img

如上所述，通过对索引的 1083 个 UserID 标记进行二分搜索，确定了第 176 个标记。因此，它对应的颗粒 176 可能包含 UserID 列值为 749.927.693 的行。

> **颗粒选择的具体过程**上图显示，标记 176 是第一个 UserID 值小于 749.927.693 的索引条目，并且下一个标记 (标记 177) 的颗粒 177 的最小 UserID 值大于该值的索引条目。因此，只有标记 176 对应的颗粒 176 可能包含 UserID 列值为 749.927.693 的行。

为了确认 (或排除) 颗粒 176 中的某些行包含 UserID 列值为 749.927.693，需要将属于此颗粒的所有 8192 行读取到 ClickHouse。

为了读取这部分数据，ClickHouse 需要知道颗粒 176 的物理地址。

在 ClickHouse 中，我们表的所有颗粒的物理位置都存储在标记文件中。与数据文件类似，每个表的列有一个标记文件。

下图显示了三个标记文件 UserID.mrk、URL.mrk、EventTime.mrk，为表的 UserID、URL 和 EventTime 列存储颗粒的物理位置。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/341eee1e-08da-4d88-bc32-1d94cd1577a9.png?raw=true)

img

我们已经讨论了主索引是一个扁平的未压缩数组文件 (primary.idx)，其中包含从 0 开始编号的索引标记。

类似地，标记文件也是一个扁平的未压缩数组文件 (\*.mrk)，其中包含从 0 开始编号的标记。

一旦 ClickHouse 确定并选择了可能包含查询所需的匹配行的颗粒的索引标记，就可以在标记文件数组中查找，以获得颗粒的物理位置。

每个特定列的标记文件条目以偏移量的形式存储两个位置:

-   第一个偏移量 (上图中的'block_offset') 是在包含所选颗粒的压缩版本的压缩列数据文件中定位块。这个压缩块可能包含几个压缩的颗粒。所定位的压缩文件块在读取时被解压到内存中。
-   标记文件的第二个偏移量 (上图中的“granule_offset”) 提供了颗粒在解压数据块中的位置。

定位到的颗粒中的所有 8192 行数据都会被 ClickHouse 加载然后进一步处理。

> 为什么需要 MARK 文件

为什么主索引不直接包含与索引标记相对应的颗粒的物理位置？

因为 ClickHouse 设计的场景就是超大规模数据，非常高效地使用磁盘和内存非常重要。

主索引文件需要放入内存中。

对于我们的示例查询，ClickHouse 使用了主索引，并选择了可能包含与查询匹配的行的单个颗粒。只有对于这一个颗粒，ClickHouse 才需定位物理位置，以便将相应的行组读取以进一步的处理。

而且，只有 UserID 和 URL 列需要这个偏移量信息。

对于查询中不使用的列，例如 EventTime，不需要偏移量信息。

对于我们的示例查询，Clickhouse 只需要 UserID 数据文件 (UserID.bin) 中 176 颗粒的两个物理位置偏移，以及 URL 数据文件 (URL.data) 中 176 颗粒的两个物理位置偏移。

由 mark 文件提供的间接方法避免了直接在主索引中存储所有三个列的所有 1083 个颗粒的物理位置的条目：因此避免了在主内存中有不必要的 (可能未使用的) 数据。

下面的图表和文本说明了我们的查询示例，ClickHouse 如何在 UserID.bin 数据文件中定位 176 颗粒。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/42afd091-4d07-4fea-84ca-32a12212b2c4.png?raw=true)

img

我们在本文前面讨论过，ClickHouse 选择了主索引标记 176，因此 176 颗粒可能包含查询所需的匹配行。

ClickHouse 现在使用从索引中选择的标记号 (176) 在 UserID.mark 中进行位置数组查找，以获得两个偏移量，用于定位颗粒 176。

如图所示，第一个偏移量是定位 UserID.bin 数据文件中的压缩文件块，该数据文件包含颗粒 176 的压缩数据。

一旦所定位的文件块被解压缩到主内存中，就可以使用标记文件的第二个偏移量在未压缩的数据中定位颗粒 176。

ClickHouse 需要从 UserID.bin 数据文件和 URL.bin 数据文件中定位 (读取) 颗粒 176，以便执行我们的示例查询(UserID 为 749.927.693 的互联网用户点击次数最多的 10 个 url)。

上图显示了 ClickHouse 如何定位 UserID.bin 数据文件的颗粒。

同时，ClickHouse 对 URL.bin 数据文件的颗粒 176 执行相同的操作。这两个不同的颗粒被对齐并加载到 ClickHouse 引擎以进行进一步的处理，即聚合并计算 UserID 为 749.927.693 的所有行的每组 URL 值，最后以计数降序输出 10 个最大的 URL 组。

## 查询使用第二位主键的性能问题

当查询对复合键的一部分并且是第一个主键列进行过滤时，ClickHouse 将对主键列的索引标记运行二分查找。

但是，当查询对联合主键的一部分但不是第一个键列进行过滤时，会发生什么情况？

> NOTE

我们讨论了这样一种场景: 查询不是显式地对第一个主键列进行过滤，而是对第一个主键列之后的任何键列进行过滤。

当查询同时对第一个主键列和第一个主键列之后的任何键列进行过滤时，ClickHouse 将对第一个主键列的索引标记运行二分查找。

我们使用一个查询来计算最点击 "[http://public_search" 的最多的前 10 名用户：](http://public_search"的最多的前10名用户：)

```null
SELECT UserID, count(UserID) AS CountWHERE URL = 'http://public_search'
```

结果是：

```null
10 rows in set. Elapsed: 0.086 sec.Processed 8.81 million rows,799.69 MB (102.11 million rows/s., 9.27 GB/s.)
```

客户端输出表明，尽管 URL 列是联合主键的一部分，ClickHouse 几乎执行了一一次全表扫描！ClickHouse 从表的 887 万行中读取 881 万行。

如果启用了 trace 日志，那么 ClickHouse 服务日志文件显示，ClickHouse 在 1083 个 URL 索引标记上使用了通用的排除搜索，以便识别那些可能包含 URL 列值为 "[http://public_search" 的行。](http://public_search"的行。)

```null
...Executor): Key condition: (column 1 in ['http://public_search',...Executor): Used generic exclusion search over index for part all_1_9_2...Executor): Selected 1/1 parts by partition key, 1 parts by primary key,1076/1083 marks by primary key, 1076 marks to read from 5 ranges...Executor): Reading approx. 8814592 rows with 10 streams
```

我们可以在上面的跟踪日志示例中看到，1083 个颗粒中有 1076 个 (通过标记) 被选中，因为可能包含具有匹配 URL 值的行。

这将导致 881 万行被读取到 ClickHouse 引擎中 (通过使用 10 个流并行地读取)，以便识别实际包含 URL 值 "[http://public_search" 的行。](http://public_search"的行。)

然而，稍后仅仅 39 个颗粒包含匹配的行。

虽然基于联合主键 (UserID, URL) 的主索引对于加快过滤具有特定 UserID 值的行的查询非常有用，但对于过滤具有特定 URL 值的行的查询，索引并没有提供显著的帮助。

原因是 URL 列不是第一个主键列，因此 ClickHouse 是使用一个通用的排除搜索算法 (而不是二分查找) 查找 URL 列的索引标志，和 UserID 主键列不同，它的算法的有效性依赖于 URL 列的基数。

为了说明，我们给出通用的排除搜索算法的工作原理：

> 通用排除搜索算法

下面将演示当通过第一个列之后的任何列选择颗粒时，当前一个键列具有或高或低的基数时，ClickHouse 通用排除搜索算法 是如何工作的。

作为这两种情况的例子，我们将假设：

-   搜索 URL 值为 "W3" 的行。
-   点击表抽象简化为只有简单值的 UserID 和 UserID。
-   相同联合主键 (UserID、URL)。这意味着行首先按 UserID 值排序，具有相同 UserID 值的行然后再按 URL 排序。
-   颗粒大小为 2，即每个颗粒包含两行。

在下面的图表中，我们用橙色标注了每个颗粒的最小键列值。

**前缀主键低基数**

假设 UserID 具有较低的基数。在这种情况下，相同的 UserID 值很可能分布在多个表行和颗粒上，从而分布在索引标记上。对于具有相同 UserID 的索引标记，索引标记的 URL 值按升序排序 (因为表行首先按 UserID 排序，然后按 URL 排序)。这使得有效的过滤如下所述：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/e9925c86-4055-45ca-be77-704f6f300030.png?raw=true)

在上图中，我们的抽象样本数据的颗粒选择过程有三种不同的场景:

1.  如果索引标记 0 的 (最小)URL 值小于 W3，并且紧接索引标记的 URL 值也小于 W3，则可以排除索引标记 0，因为标记 0、标记 1 和标记 2 具有相同的 UserID 值。注意，这个排除前提条件确保颗粒 0 和下一个颗粒 1 完全由 U1 UserID 值组成，这样 ClickHouse 就可以假设颗粒 0 中的最大 URL 值也小于 W3 并排除该颗粒。
2.  如果索引标记 1 的 URL 值小于 (或等于)W3，并且后续索引标记的 URL 值大于 (或等于)W3，则选择索引标记 1，因为这意味着粒度 1 可能包含 URL 为 W3 的行)。
3.  可以排除 URL 值大于 W3 的索引标记 2 和 3，因为主索引的索引标记存储了每个颗粒的最小键列值，因此颗粒 2 和 3 不可能包含 URL 值 W3。

**前缀主键高基数**

当 UserID 具有较高的基数时，相同的 UserID 值不太可能分布在多个表行和颗粒上。这意味着索引标记的 URL 值不是单调递增的：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/6e9fcce8-e204-422e-a208-4057901822a5.png?raw=true)

img

正如在上面的图表中所看到的，所有 URL 值小于 W3 的标记都被选中，以便将其关联的颗粒的行加载到 ClickHouse 引擎中。

这是因为虽然图中的所有索引标记都属于上面描述的场景 1，但它们不满足前面提到的排除前提条件，即两个直接随后的索引标记都具有与当前标记相同的 UserID 值，因此不能被排除。

例如，考虑索引标记 0，其 URL 值小于 W3，并且其直接后续索引标记的 URL 值也小于 W3。这不能排除，因为两个直接随后的索引标记 1 和 2 与当前标记 0 没有相同的 UserID 值。

请注意，随后的两个索引标记需要具有相同的 UserID 值。这确保了当前和下一个标记的颗粒完全由 U1 UserID 值组成。如果仅仅是下一个标记具有相同的 UserID，那么下一个标记的 URL 值可能来自具有不同 UserID 的表行——当您查看上面的图表时，确实是这样的情况，即 W2 来自 U2 而不是 U1 的行。

这最终阻止了 ClickHouse 对颗粒 0 中的最大 URL 值进行假设。相反，它必须假设颗粒 0 可能包含 URL 值为 W3 的行，并被迫选择标记 0。

同样的情况也适用于标记 1、2 和 3。

> 结论

当查询对联合主键的一部分列 (但不是第一个键列) 进行过滤时，ClickHouse 使用的通用排除搜索算法 (而不是二分查找) 在前一个键列基数较低时最有效。

在我们的示例数据集中，两个键列 (UserID、URL) 都具有类似的高基数，并且，如前所述，当 URL 列的前一个键列具有较高基数时，通用排除搜索算法不是很有效。

**看下跳数索引**

因为 UserID 和 URL 具有较高的基数，根据 URL 过滤数据不是特别有效，对 URL 列创建二级跳数索引同样也不会有太多改善。

例如，这两个语句在我们的表的 URL 列上创建并填充一个 minmax 跳数索引。

```null
ALTER TABLE hits_UserID_URL ADD INDEX url_skipping_index URL TYPE minmax GRANULARITY 4;ALTER TABLE hits_UserID_URL MATERIALIZE INDEX url_skipping_index;
```

ClickHouse 现在创建了一个额外的索引来存储—每组 4 个连续的颗粒 (注意上面 ALTER TABLE 语句中的 GRANULARITY 4 子句)—最小和最大的 URL 值：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/78ceebd6-dfe5-43d8-99ec-afbe43c5a208.png?raw=true)

img

第一个索引条目 (上图中的 mark 0) 存储属于表的前 4 个颗粒的行的最小和最大 URL 值。

第二个索引条目 (mark 1) 存储属于表中下一个 4 个颗粒的行的最小和最大 URL 值，依此类推。

(ClickHouse 还为跳数索引创建了一个特殊的标记文件，用于定位与索引标记相关联的颗粒组。)

由于 UserID 和 URL 的基数相似，在执行对 URL 的查询过滤时，这个二级跳数索引不能帮助排除选择的颗粒。

正在寻找的特定 URL 值 ('[http://public_search'](http://public_search')) 很可能是索引为每组颗粒存储的最小值和最大值之间的值，导致 ClickHouse 被迫选择这组颗粒 (因为它们可能包含匹配查询的行)。

因此，如果我们想显著提高过滤具有特定 URL 的行的示例查询的速度，那么我们需要使用针对该查询优化的主索引。

此外，如果我们想保持过滤具有特定 UserID 的行的示例查询的良好性能，那么我们需要使用多个主索引。

下面是实现这一目标的方法。

## 使用多个主键索引进行调优

如果我们想显著加快我们的两个示例查询——一个过滤具有特定 UserID 的行，一个过滤具有特定 URL 的行——那么我们需要使用多个主索引，通过使用这三个方法中的一个：

-   新建一个不同主键的新表。
-   创建一个物化视图。
-   增加 projection。

这三个方法都会有效地将示例数据复制到另一个表中，以便重新组织表的主索引和行排序顺序。

然而，这三个选项的不同之处在于，附加表对于查询和插入语句的路由对用户的透明程度。

当创建有不同主键的第二个表时，查询必须显式地发送给最适合查询的表版本，并且必须显式地插入新数据到两个表中，以保持表的同步：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/2729c83b-9e81-42ee-ab92-d8d0c0886ded.png?raw=true)

img

在物化视图中，额外的表被隐藏，数据自动在两个表之间保持同步：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/aa36aeb1-7778-4a06-b1f3-97191129f8aa.png?raw=true)

img

projection 方式是最透明的选项，因为除了自动保持隐藏的附加表与数据变化同步外，ClickHouse 还会自动选择最有效的表版本进行查询：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/152fd4a3-f1eb-45c3-a8fa-554423e3a195.png?raw=true)

img

下面我们使用真实的例子详细讨论下这三种方式。

## 通过辅助表使用联合主键索引

我们创建一个新的附加表，其中我们在主键中切换键列的顺序 (与原始表相比)：

```null
CREATE TABLE hits_URL_UserIDPRIMARY KEY (URL, UserID)ORDER BY (URL, UserID, EventTime)SETTINGS index_granularity = 8192, index_granularity_bytes = 0;
```

写入 887 万行源表数据：

```null
INSERT INTO hits_URL_UserIDSELECT * from hits_UserID_URL;
```

结果：

```null
0 rows in set. Elapsed: 2.898 sec. Processed 8.87 million rows, 838.84 MB (3.06 million rows/s., 289.46 MB/s.)
```

最后 optimize 下：

```null
OPTIMIZE TABLE hits_URL_UserID FINAL;
```

因为我们切换了主键中列的顺序，插入的行现在以不同的字典顺序存储在磁盘上 (与我们的原始表相比)，因此该表的 1083 个颗粒也包含了与以前不同的值：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/dcb99b60-5ea6-4088-8eaa-9a8133f000f8.png?raw=true)

img

主键索引如下：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/7bd94455-32d7-4863-bb1e-862b0d6f6202.png?raw=true)

img

现在计算最频繁点击 URL"[http://public_search" 的前 10 名用户，这时候的查询速度是明显加快的：](http://public_search"的前10名用户，这时候的查询速度是明显加快的：)

```null
SELECT UserID, count(UserID) AS CountWHERE URL = 'http://public_search'
```

结果：

```null
10 rows in set. Elapsed: 0.017 sec.Processed 319.49 thousand rows,11.38 MB (18.41 million rows/s., 655.75 MB/s.)
```

现在没有全表扫描了，ClickHouse 执行高效了很多。

对于原始表中的主索引 (其中 UserID 是第一个键列，URL 是第二个键列)，ClickHouse 在索引标记上使用了通用排除搜索来执行该查询，但这不是很有效，因为 UserID 和 URL 的基数同样很高。

将 URL 作为主索引的第一列，ClickHouse 现在对索引标记运行二分搜索。ClickHouse 服务器日志文件中对应的跟踪日志：

```null
...Executor): Key condition: (column 0 in ['http://public_search',...Executor): Running binary search on index range for part all_1_9_2 (1083 marks)...Executor): Found (LEFT) boundary mark: 644...Executor): Found (RIGHT) boundary mark: 683...Executor): Found continuous range in 19 steps...Executor): Selected 1/1 parts by partition key, 1 parts by primary key,39/1083 marks by primary key, 39 marks to read from 1 ranges...Executor): Reading approx. 319488 rows with 2 streams
```

ClickHouse 只选择了 39 个索引标记，而不是使用通用排除搜索时的 1076 个。

请注意，辅助表经过了优化，以加快对 url 的示例查询过滤的执行。

像之前我们查询过滤 URL 一样，如果我们现在对辅助表查询过滤 UserID，性能同样会比较差，因为现在 UserID 是第二主索引键列，所以 ClickHouse 将使用通用排除搜索算法查找颗粒，这对于类似高基数的 UserID 和 URL 来说不是很有效。

点击下面了解详情：

```null
SELECT URL, count(URL) AS Count┌─URL────────────────────────────┬─Count─┐└────────────────────────────────┴───────┘10 rows in set. Elapsed: 0.024 sec.Processed 8.02 million rows,73.04 MB (340.26 million rows/s., 3.10 GB/s.)...Executor): Key condition: (column 1 in [749927693, 749927693])...Executor): Used generic exclusion search over index for part all_1_9_2...Executor): Selected 1/1 parts by partition key, 1 parts by primary key,980/1083 marks by primary key, 980 marks to read from 23 ranges...Executor): Reading approx. 8028160 rows with 10 streams
```

现在我们有了两张表。优化了对 UserID 和 URL 的查询过滤，分别:

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/15132bc8-b05e-4d13-ae64-07303981df93.png?raw=true)

img

## 通过物化视图使用联合主键

在原表上创建物化视图：

```null
CREATE MATERIALIZED VIEW mv_hits_URL_UserIDPRIMARY KEY (URL, UserID)ORDER BY (URL, UserID, EventTime)AS SELECT * FROM hits_UserID_URL;
```

结果：

```null
0 rows in set. Elapsed: 2.935 sec. Processed 8.87 million rows, 838.84 MB (3.02 million rows/s., 285.84 MB/s.)
```

> NOTE

-   我们在视图的主键中切换键列的顺序 (与原始表相比)
-   物化视图由一个隐藏表支持，该表的行顺序和主索引基于给定的主键定义
-   我们使用 POPULATE 关键字，以便用源表 hits_UserID_URL 中的所有 887 万行立即导入新的物化视图
-   如果在源表 hits_UserID_URL 中插入了新行，那么这些行也会自动插入到隐藏表中
-   实际上，隐式创建的隐藏表的行顺序和主索引与我们上面显式创建的辅助表相同:

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/313da234-8e03-4dd2-9914-357d5e279f70.png?raw=true)

img

ClickHouse 将隐藏表的列数据文件 (.bin)、标记文件(.mrk2) 和主索引 (primary.idx) 存储在 ClickHouse 服务器的数据目录的一个特殊文件夹中：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/6fa330b7-90cc-4d30-9c07-b135b2e9182e.png?raw=true)

img

物化视图背后的隐藏表 (和它的主索引) 现在可以用来显著加快我们在 URL 列上查询过滤的执行速度：

```null
SELECT UserID, count(UserID) AS CountWHERE URL = 'http://public_search'
```

结果：

```null
10 rows in set. Elapsed: 0.026 sec.Processed 335.87 thousand rows,13.54 MB (12.91 million rows/s., 520.38 MB/s.)
```

物化视图背后隐藏表 (及其主索引) 实际上与我们显式创建的辅助表是相同的，所以查询的执行方式与显式创建的表相同。

ClickHouse 服务器日志文件中相应的跟踪日志确认了 ClickHouse 正在对索引标记运行二分搜索：

```null
...Executor): Key condition: (column 0 in ['http://public_search',...Executor): Running binary search on index range ......Executor): Selected 4/4 parts by partition key, 4 parts by primary key,41/1083 marks by primary key, 41 marks to read from 4 ranges...Executor): Reading approx. 335872 rows with 4 streams
```

## 通过 projections 使用联合主键索引

Projections 目前是一个实验性的功能，因此我们需要告诉 ClickHouse：

```null
SET allow_experimental_projection_optimization = 1;
```

在原表上创建 projection：

```null
ALTER TABLE hits_UserID_URL    ADD PROJECTION prj_url_userid
```

物化 projection：

```null
ALTER TABLE hits_UserID_URL    MATERIALIZE PROJECTION prj_url_userid;
```

> NOTE

-   该 projection 正在创建一个隐藏表，该表的行顺序和主索引基于该 projection 的给定 order BY 子句
-   我们使用 MATERIALIZE 关键字，以便立即用源表 hits_UserID_URL 的所有 887 万行导入隐藏表
-   如果在源表 hits_UserID_URL 中插入了新行，那么这些行也会自动插入到隐藏表中
-   查询总是 (从语法上) 针对源表 hits_UserID_URL，但是如果隐藏表的行顺序和主索引允许更有效地执行查询，那么将使用该隐藏表
-   实际上，隐式创建的隐藏表的行顺序和主索引与我们显式创建的辅助表相同：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/c8ba2627-48f0-4dae-a418-685b9b969ca5.png?raw=true)

img

ClickHouse 将隐藏表的列数据文件 (.bin)、标记文件(.mrk2) 和主索引 (primary.idx) 存储在一个特殊的文件夹中(在下面的截图中用橙色标记)，紧挨着源表的数据文件、标记文件和主索引文件：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-6-5%2018-19-41/5459a701-5a43-4a39-a689-2f6d371a85bd.png?raw=true)

img

由投影创建的隐藏表 (以及它的主索引) 现在可以 (隐式地) 用于显著加快 URL 列上查询过滤的执行。注意，查询在语法上针对投影的源表。

```null
SELECT UserID, count(UserID) AS CountWHERE URL = 'http://public_search'
```

结果：

```null
10 rows in set. Elapsed: 0.029 sec.Processed 319.49 thousand rows, 11.38 MB (11.05 million rows/s., 393.58 MB/s.)
```

因为由投影创建的隐藏表 (及其主索引) 实际上与我们显式创建的辅助表相同，所以查询的执行方式与显式创建的表相同。

ClickHouse 服务器日志文件中跟踪日志确认了 ClickHouse 正在对索引标记运行二分搜索：

```null
...Executor): Key condition: (column 0 in ['http://public_search',...Executor): Running binary search on index range for part prj_url_userid (1083 marks)...Executor): Choose complete Normal projection prj_url_userid...Executor): projection required columns: URL, UserID...Executor): Selected 1/1 parts by partition key, 1 parts by primary key,39/1083 marks by primary key, 39 marks to read from 1 ranges...Executor): Reading approx. 319488 rows with 2 streams
```

## 移除无效的主键列

带有联合主键 (UserID, URL) 的表的主索引对于加快 UserID 的查询过滤非常有用。但是，尽管 URL 列是联合主键的一部分，但该索引在加速 URL 查询过滤方面并没有提供显著的帮助。

反之亦然：具有复合主键 (URL, UserID) 的表的主索引加快了 URL 上的查询过滤，但没有为 UserID 上的查询过滤提供太多支持。

由于主键列 UserID 和 URL 的基数同样很高，过滤第二个键列的查询不会因为第二个键列位于索引中而受益太多。

因此，从主索引中删除第二个键列 (从而减少索引的内存消耗) 并使用多个主索引是有意义的。

但是，如果复合主键中的键列在基数上有很大的差异，那么查询按基数升序对主键列进行排序是有益的。

主键键列之间的基数差越大，主键键列的顺序越重要。我们将在以后的文章中对此进行演示。请继续关注。
