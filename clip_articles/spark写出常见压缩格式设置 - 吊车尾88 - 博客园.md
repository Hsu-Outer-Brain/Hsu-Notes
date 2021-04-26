# spark写出常见压缩格式设置 - 吊车尾88 - 博客园
1. Hadoop 之常见压缩格式以及性能对比

#### 1. 压缩的好处和坏处

好处

-   减少存储磁盘空间
-   降低 IO(网络的 IO 和磁盘的 IO)
-   加快数据在磁盘和网络中的传输速度，从而提高系统的处理速度

坏处

-   由于使用数据时，需要先将数据解压，加重 CPU 负荷。而且压缩的越狠，耗费的时间越多。

#### 2. 压缩格式

| 压缩格式 | 工具 | 算法 | 扩展名 | 是否支持分割 | Hadoop 编码 / 解码器 | hadoop 自带 |
| DEFLATE | N/A | DEFLATE | .deflate | No | org.apache.hadoop.io.compress.DefalutCodec | 是 |
| gzip | gzip | DEFLATE | .gz | No | org.apache.hadoop.io.compress.GzipCodec | 是 |
| bzip2 | bzip2 | bzip2 | .bz2 | yes | org.apache.hadoop.io.compress.Bzip2Codec | 是 |
| LZO | Lzop | LZO | .lzo | yes(建索引) | com.hadoop.compression.lzo.LzoCodec | 是 |
| LZ4 | N/A | LZ4 | .lz4 | No | org.apache.hadoop.io.compress.Lz4Codec | 否 |
| Snappy | N/A | Snappy | .snappy | No | org.apache.hadoop.io.compress.SnappyCodec | 否 |

压缩比：Snappy&lt;LZ4&lt;LZO&lt;GZIP&lt;BZIP2

**3. 优缺点**  
a. gzip

优点：压缩比在四种压缩方式中较高；hadoop 本身支持，在应用中处理 gzip 格式的文件就和直接处理文本一样；有 hadoop native 库；大部分 linux 系统都自带 gzip 命令，使用方便  
缺点：不支持 split  
b. lzo

优点：压缩 / 解压速度也比较快，合理的压缩率；支持 split，是 hadoop 中最流行的压缩格式；支持 hadoop native 库；需要在 linux 系统下自行安装 lzop 命令，使用方便  
缺点：压缩率比 gzip 要低；hadoop 本身不支持，需要安装；lzo 虽然支持 split，但需要对 lzo 文件建索引，否则 hadoop 也是会把 lzo 文件看成一个普通文件（为了支持 split 需要建索引，需要指定 inputformat 为 lzo 格式）  
c. snappy

优点：压缩速度快；支持 hadoop native 库  
缺点：不支持 split；压缩比低；hadoop 本身不支持，需要安装；linux 系统下没有对应的命令  
d. bzip2

优点：支持 split；具有很高的压缩率，比 gzip 压缩率都高；hadoop 本身支持，但不支持 native；在 linux 系统下自带 bzip2 命令，使用方便  
缺点：压缩 / 解压速度慢；不支持 native

**4.spark 输出压缩文件**

1）RDD 输出压缩文件

![](https://common.cnblogs.com/images/copycode.gif)

import org.apache.hadoop.io.compress.BZip2Codec // bzip2 压缩率最高，压缩解压速度较慢，支持 split。
rdd.saveAsTextFile("codec/bzip2",classOf\[BZip2Codec])

import org.apache.hadoop.io.compress.SnappyCodec //snappy json 文本压缩率 38.2%，压缩和解压缩时间短。
rdd.saveAsTextFile("codec/snappy",classOf\[SnappyCodec])

import org.apache.hadoop.io.compress.GzipCodec //gzip 压缩率高，压缩和解压速度较快，不支持 split，如果不对文件大小进行控制，下次分析可能可能会造成效率低下的问题。
rdd.saveAsTextFile("codec/gzip",classOf\[GzipCodec])

![](https://common.cnblogs.com/images/copycode.gif)

2）spark sql 输出压缩文件

parquet 文件压缩

parquet 为文件提供了列式存储，查询时只会取出需要的字段和分区，对 IO 性能的提升非常大，同时占用空间较小，即使是 parquet 的 uncompressed 存储方式也比普通的文本要小的多。

sparkConf.set("spark.sql.parquet.compression.codec","gzip")
dataset.write().parquet("path");

**parquet 存储提供了:**`**lzo gzip snappy uncompressed 四种方式 **。`

3) spark sql 的 csv 文件压缩设置

df.write.mode(SaveMode.Overwrite).option("compression", "gzip").csv(s"${path}") 
 [https://www.cnblogs.com/yyy-blog/p/12747133.html](https://www.cnblogs.com/yyy-blog/p/12747133.html)
