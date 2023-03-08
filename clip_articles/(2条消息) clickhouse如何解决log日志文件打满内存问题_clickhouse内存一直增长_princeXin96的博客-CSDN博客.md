# (2条消息) clickhouse如何解决log日志文件打满内存问题_clickhouse内存一直增长_princeXin96的博客-CSDN博客
[(2 条消息) clickhouse 如何解决 log 日志文件打满内存问题\_clickhouse 内存一直增长\_princeXin96 的博客 - CSDN 博客](https://blog.csdn.net/qq_40407087/article/details/128397872) 

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2009-32-57/6080831e-8b3c-4baf-a540-7f26b063d38b.png?raw=true)

于 2022-12-21 15:54:40 首次发布

版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。

今天早上想看看[clickhouse](https://so.csdn.net/so/search?q=clickhouse&spm=1001.2101.3001.7020)里面跑了多少数据了，一查询刺激了。。。。。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2009-32-57/f1f7ff1f-26a6-45ff-aec6-e5dcf1e30857.png?raw=true)

 抱着遇到问题解决问题的态度，根据报错那不就是内存空间不足了吗？

来到服务器 df -h 一跑果然 use% 达到 100%

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2009-32-57/063330a8-b15c-40bd-abdf-674dda8b7a3f.png?raw=true)

 没得说清内存吧，可是要怎么清呢，问了度娘一大圈，总结还是 clickhouse 一些 XX_log 日志文件太大了，那就删呗

重点来了。。。

1、先看是哪个日志文件把我的内存打满了

```null
    formatReadableSize(sum(data_uncompressed_bytes)) AS `原始大小`,    formatReadableSize(sum(data_compressed_bytes)) AS `压缩大小`,    round((sum(data_compressed_bytes) / sum(data_uncompressed_bytes)) * 100, 0) AS `压缩率`,FROM system.parts where database = 'system' group by `table`
```

2、找到后执行删除命令，注意 clickhouse 里面的日志文件都是按照日期分区存储的，query_thread_log 或者 query_log, 看具体哪个日志文件占的内存大

```null
alter table system.query_thread_log drop partition '202201';
```

3\. 如果日志文件的大小超过 50G，删除会报错，此时需要添加一个文件 / var/lib/clickhouse/flags/force_drop_table, 文件内容如下；添加完即可成功删除

```null
<max_table_size_to_drop>0</max_table_size_to_drop>
```

删干净之后再用 df 看，发现内存空闲多了

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2009-32-57/bfd92277-3b5e-4b46-b0d0-8e19ae70acc6.png?raw=true)

4、删除完之后并没有完全解决问题，要想完全解决问题，需要修改服务端的 config.xml（配置文件位置 /etc/clickhouse-server/config.xml）配置文件（设置每 15 天清一次日志文件）

```null
<database>system</database><table>query_thread_log</table><engine>Engine = MergeTree PARTITION BY event_date ORDER BY event_time TTL event_date + INTERVAL 15 day</engine><flush_interval_milliseconds>7500</flush_interval_milliseconds>
```

5、最最重要的是要重启一下 clickhouse，如果你是[docker 部署](https://so.csdn.net/so/search?q=docker%E9%83%A8%E7%BD%B2&spm=1001.2101.3001.7020)的直接，不重启还是会报错

```null
docker restart 容器id
```
