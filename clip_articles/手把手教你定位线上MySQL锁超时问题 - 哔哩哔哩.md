# 手把手教你定位线上MySQL锁超时问题 - 哔哩哔哩
[手把手教你定位线上 MySQL 锁超时问题 - 哔哩哔哩](https://www.bilibili.com/read/cv20199396/) 

 昨晚我正在床上睡得着着的，突然来了一条短信。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-23%2016-30-08/befe3c0e-75f7-401c-98ee-a685b4a3a4d0.webp?raw=true)

我赶紧登录线上系统，查看业务日志。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-23%2016-30-08/cdbce17b-1af9-4e90-ae50-dfc090f45764.webp?raw=true)

发现有**MySQL 锁超时**的错误日志。

不用想，肯定有另一个事务正在修改这条订单，持有这条订单的锁。

导致当前事务获取不到锁，一直等待，直到超过锁超时时间，然后报错。

既然问题已经清楚了，接下来就轮到怎么排查一下到底是哪个事务正在持有这条订单的锁。

好在 MySQL 提供了丰富的工具，帮助我们排查锁竞争问题。

现场复现一个这个问题：

创建一张用户表，造点数据：

``CREATE TABLE `user` (`id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键 ID',  `name` varchar(50) NOT NULL DEFAULT ''COMMENT'姓名',  PRIMARY KEY (`id`) ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;``

事务 1，更新 id=1 的用户姓名，不提交事务：

`begin;update user set name='一灯' where id=1;`

事务 2，删除 id=1 的数据，这时候会产生锁等待：

`begin;delete from user where id=1;`

接下来，我们就通过 MySQL 提供的锁竞争统计表，排查一下锁等待问题：

先查一下锁等待情况：

`select * from information_schema.innodb_lock_waits;`

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-23%2016-30-08/eef95c59-1056-442f-81b5-cf5294ca59eb.webp?raw=true)

可以看到有一个锁等待的事务。

然后再查一下正在竞争的锁有哪些？

`select * from information_schema.innodb_locks;`

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-23%2016-30-08/6263e495-096a-495b-b79d-77fb451db148.webp?raw=true)

可以看到，MySQL 统计的非常详细：

> lock_trx_id 表示事务 ID
>
> lock_mode 表示排它锁还是共享锁
>
> lock_type 表示锁定的记录，还是范围
>
> lock_table 锁的表名
>
> lock_index 锁定的是主键索引

`select * from information_schema.innodb_trx;`

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-23%2016-30-08/11f0f934-0f0b-4dc9-81c4-56b066efadce.webp?raw=true)

可以清楚的看到正在执行的事务有两个，一个状态是锁等待（`LOCK WAIT`），正在执行的 SQL 也打印出来了：

`delete from user where id=1;`

正是事务 2 的删除语句。

不用问，第二条，显示正在运行状态（`RUNNING`）的事务就是正在持有锁的事务 1，MySQL 线程 id（`trx_mysql_thread_id`）是 193。

我们用 MySQL 线程 id 查一下事务线程 id：

`select * from performance_schema.threads where processlist_id=193;`

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-23%2016-30-08/19ca15af-d56d-48f3-be8f-812ebc149e92.webp?raw=true)

找到对应的事务线程 id 是 218，然后再找一下这个线程正在执行的 SQL 语句：

`select THREAD_ID,CURRENT_SCHEMA,SQL_TEXT from performance_schema.events_statements_current where thread_id=218;`

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-23%2016-30-08/d91da7ca-c362-419b-819f-7c0f7cea871e.webp?raw=true)

可以清楚的看到这个线程正在执行的 SQL 语句就是事务 1 的 update 语句。

持有锁的 SQL 语句找到了，接下来再去找对应的业务代码也就轻而易举了。

以上是基于 MySQL5.7 版本，在 MySQL8.0 版本中有些命令已经删除了，替换成了其他命令，下篇文章再讲一下 MySQL8.0 怎么定位**MySQL 锁超时**问题。
