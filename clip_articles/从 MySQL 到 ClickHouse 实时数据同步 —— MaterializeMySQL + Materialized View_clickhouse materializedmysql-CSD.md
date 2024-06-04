# 从 MySQL 到 ClickHouse 实时数据同步 —— MaterializeMySQL + Materialized View_clickhouse materializedmysql-CSDN博客
[从 MySQL 到 ClickHouse 实时数据同步 —— MaterializeMySQL + Materialized View_clickhouse materializedmysql-CSDN 博客](https://wxy0327.blog.csdn.net/article/details/137959646?spm=1001.2101.3001.6650.3&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EYuanLiJiHua%7EPosition-3-137959646-blog-113210489.235%5Ev43%5Epc_blog_bottom_relevance_base4&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EYuanLiJiHua%7EPosition-3-137959646-blog-113210489.235%5Ev43%5Epc_blog_bottom_relevance_base4&utm_relevant_index=4) 

 **目录**

[一、总体架构](#t0)

[二、安装配置 MySQL](#t1)

[1. 创建 mysql 用户](#t2)

[2. 建立 MySQL 使用的目录](#t3)

[3. 解压安装包](#t4)

[4. 配置环境变量](#t5)

[5. 创建 MySQL 配置文件](#t6)

[6. MySQL 系统初始化](#t7)

[7. 启动 mysql 服务器](#t8)

[8. 创建 dba 用户](#t9)

[三、配置 MySQL 主从复制](#t10)

[四、在 ClickHouse 中创建 MySQL 引擎数据库](#t11)

[五、在 ClickHouse 中创建物化视图](#t12)

[六、物化视图数据刷新](#t13)

[1. 初始数据装载](#t14)

[2. 增量数据刷新](#t15)

[参考：](#t16)

* * *

        本篇演示使用 ClickHouse 的 MaterializeMySQL 数据库引擎和物化视图，实时将 [MySQL](https://so.csdn.net/so/search?q=MySQL&spm=1001.2101.3001.7020) 库表中的数据同步到 ClickHouse 的库表中。相关软件版本如下：

-   MySQL：8.0.16
-   ClickHouse：24.1.8

        这种方案的好处是操作简单，几乎不需要额外配置即可实现。

## 一、总体架构

        总体结构如下图所示。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-6-4%2010-40-35/58cadcac-bad6-4fa3-be0f-eaa88c7d91cf.png?raw=true)

        ClickHouse 是由四个实例构成的两分片、没分片两副本集群，票选和协调器使用 ClickHouse 自带的 keeper 组件。分片、副本、keeper 节点部署如下表所示。

\| 

**IP**

 \| 

**主机名**

 \| 

**实例角色**

 \| 

**ClickHouse Keeper**

 \|
\| 

172.18.4.126

 \| 

node1

 \| 

分片 1 副本 1

 \| 

-    \|
    \| 

172.18.4.188

 \| 

node2

 \| 

分片 1 副本 2

 \| 

-    \|
    \| 

172.18.4.71

 \| 

node3

 \| 

分片 2 副本 1

 \| 

-    \|
    \| 

172.18.4.86

 \| 

node4

 \| 

分片 2 副本 2

 \|  \|

        ClickHouse 集群部署过程参见 “[ClickHouse 集群部署（不需要 Zookeeper）](https://blog.csdn.net/wzy0623/article/details/137809306 "ClickHouse 集群部署（不需要 Zookeeper）")”。另外在 172.18.16.156 上安装 MySQL，并启动两个实例做主从复制，主库实例用 3306 端口，从库实例用 3307 端口。

## 二、[安装配置](https://so.csdn.net/so/search?q=%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE&spm=1001.2101.3001.7020) MySQL

        安装配置 MySQL 一主一从双实例。

### 1. 创建 mysql 用户

### 2. 建立 MySQL 使用的目录

```null
# 创建数据目录，确保数据目录 mysqldata 为空mkdir -p /data/3306/mysqldatamkdir -p /data/3306/dblogchown -R mysql:mysql /data
```

### 3. 解压[安装包](https://so.csdn.net/so/search?q=%E5%AE%89%E8%A3%85%E5%8C%85&spm=1001.2101.3001.7020)

```null
tar xvf mysql-8.0.16-linux-glibc2.12-x86_64.tar.xzln -s mysql-8.0.16-linux-glibc2.12-x86_64 mysql-8.0.16
```

### 4. 配置环境变量

```null
PATH=$PATH:$HOME/.local/bin:$HOME/bin:/home/mysql/mysql-8.0.16/bin
```

### 5. 创建 MySQL [配置文件](https://so.csdn.net/so/search?q=%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6&spm=1001.2101.3001.7020)

```null
vim /home/mysql/my_3306.cnfbinlog_transaction_dependency_tracking  = WRITESETtransaction_write_set_extraction        = XXHASH64binlog_expire_logs_seconds=259200log_bin_trust_function_creators=oncharacter-set-server = utf8mb4default_authentication_plugin=mysql_native_passwordbasedir=/home/mysql/mysql-8.0.16-linux-glibc2.12-x86_64datadir=/data/3306/mysqldatasocket=/data/3306/mysqldata/mysql.sockinnodb_buffer_pool_size = 16Gdefault-time-zone = '+8:00'innodb_print_all_deadlocks=1log-bin=/data/3306/dblog/mysql-binlog-bin-index = /data/3306/dblog/mysql-bin.index innodb_data_file_path = ibdata1:1G:autoextendinnodb_data_home_dir = /data/3306/mysqldatainnodb_log_buffer_size = 16Minnodb_log_file_size = 1Ginnodb_log_files_in_group = 3innodb_log_group_home_dir=/data/3306/dbloginnodb_max_dirty_pages_pct = 90innodb_lock_wait_timeout = 120enforce_gtid_consistency=truelog_error='/data/3306/mysqldata/master.err'
```

        以下这三个参数必须设置：

```null
default_authentication_plugin=mysql_native_passwordenforce_gtid_consistency=true
```

        如果不设置 default_authentication_plugin，在 ClickHouse 中创建 MySQL 引擎数据库会报以下错误：

```null
Received exception from server (version 24.1.8):Code: 695. DB::Exception: Received from localhost:9000. DB::Exception: There was an error on [node2:9000]: Code: 695. DB::Exception: Load job 'startup MaterializedMySQL database test_mysql' failed: Code: 537. DB::Exception: Illegal MySQL variables, the MaterializedMySQL engine requires default_authentication_plugin='mysql_native_password'. (ILLEGAL_MYSQL_VARIABLE),. (ASYNC_LOAD_FAILED) (version 24.1.8.22 (official build)). (ASYNC_LOAD_FAILED)
```

        如果不启用 GTID，在 ClickHouse 中创建 MySQL 引擎数据库会报以下错误：

```null
Received exception from server (version 24.1.8):Code: 1002. DB::Exception: Received from localhost:9000. DB::Exception: The replication sender thread cannot start in AUTO_POSITION mode: this server has GTID_MODE = OFF instead of ON.. ()
```

### 6. MySQL 系统初始化

```null
mysqld --defaults-file=/home/mysql/my_3306.cnf --initialize
```

### 7. 启动 mysql 服务器

```null
mysqld_safe --defaults-file=/home/mysql/my_3306.cnf &
```

### 8. 创建 dba 用户

```null
mysql -u root -p -S /data/3306/mysqldata/mysql.sockalter user user() identified by "123456";create user 'dba'@'%' identified with mysql_native_password by '123456';grant all on *.* to 'dba'@'%' with grant option;
```

        重复执行 2 - 8 步，将 3306 换成 3307，创建从库实例。

## 三、配置 MySQL 主从复制

        3306 主库实例执行：

```null
create user 'repl'@'%' identified with mysql_native_password by '123456';grant replication client,replication slave on *.* to 'repl'@'%';  id bigint(20) not null auto_increment,  remark varchar(32) default null comment '备注',  createtime timestamp not null default current_timestamp comment '创建时间',insert into test.t1 (remark) values ('第一行：row1'),('第二行：row2'),('第三行：row3');
```

        输出：

```null
mysql> show master status;+------------------+----------+--------------+------------------+------------------------------------------+| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                        |+------------------+----------+--------------+------------------+------------------------------------------+| mysql-bin.000001 |      977 |              |                  | ba615057-e11c-11ee-b80e-246e961c91f8:1-3 |+------------------+----------+--------------+------------------+------------------------------------------+mysql> create user 'repl'@'%' identified with mysql_native_password by '123456';Query OK, 0 rows affected (0.01 sec)mysql> grant replication client,replication slave on *.* to 'repl'@'%';Query OK, 0 rows affected (0.00 sec)mysql> create database test;Query OK, 1 row affected (0.00 sec)mysql> create table test.t1 (    ->   id bigint(20) not null auto_increment,    ->   remark varchar(32) default null comment '备注',    ->   createtime timestamp not null default current_timestamp comment '创建时间',Query OK, 0 rows affected (0.01 sec)mysql> insert into test.t1 (remark) values ('第一行：row1'),('第二行：row2'),('第三行：row3');Query OK, 3 rows affected (0.00 sec)Records: 3  Duplicates: 0  Warnings: 0
```

        3307 从库实例执行：

```null
master_host='172.18.16.156',master_password='123456',master_log_file='mysql-bin.000001',select user,host from mysql.user;
```

        输出：

```null
    -> master_host='172.18.16.156',    -> master_password='123456',    -> master_log_file='mysql-bin.000001',Query OK, 0 rows affected, 2 warnings (0.00 sec)Query OK, 0 rows affected (0.01 sec)mysql> show slave status\G*************************** 1. row ***************************               Slave_IO_State: Waiting for master to send event                  Master_Host: 172.18.16.156              Master_Log_File: mysql-bin.000001Read_Master_Log_Pos: 2431               Relay_Log_File: vvgg-z2-music-mysqld-relay-bin.000002        Relay_Master_Log_File: mysql-bin.000001  Replicate_Wild_Ignore_Table:           Exec_Master_Log_Pos: 2431Master_SSL_Verify_Server_Cert: No  Replicate_Ignore_Server_Ids:              Master_Server_Id: 1563306                  Master_UUID: ba615057-e11c-11ee-b80e-246e961c91f8             Master_Info_File: mysql.slave_master_info          SQL_Remaining_Delay: NULL      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates           Master_Retry_Count: 86400Last_SQL_Error_Timestamp:            Retrieved_Gtid_Set: ba615057-e11c-11ee-b80e-246e961c91f8:4-8            Executed_Gtid_Set: ba615057-e11c-11ee-b80e-246e961c91f8:4-8,c2df1946-e11c-11ee-8026-246e961c91f8:1-3mysql> select user,host from mysql.user;+------------------+-----------++------------------+-----------+| mysql.infoschema | localhost || mysql.session    | localhost || mysql.sys        | localhost |+------------------+-----------+mysql> select * from test.t1;+----+------------------+---------------------+| id | remark           | createtime          |+----+------------------+---------------------+|  1 | 第一行：row1     | 2024-04-19 08:46:25 ||  2 | 第二行：row2     | 2024-04-19 08:46:25 ||  3 | 第三行：row3     | 2024-04-19 08:46:25 |+----+------------------+---------------------+
```

        MySQL 主从复制相关配置参见 “[配置异步复制](https://wxy0327.blog.csdn.net/article/details/90081518#t6 "配置异步复制")”。

## 四、在 ClickHouse 中创建 MySQL 引擎数据库

```null
set allow_experimental_database_materialized_mysql=1;create database test_mysql on cluster cluster_2S_2Rengine = MaterializeMySQL('172.18.16.156:3307', 'test', 'dba', '123456');
```

        如果不设置 allow_experimental_database_materialized_mysql=1 会报如下错误：

```null
Received exception from server (version 24.1.8):Code: 336. DB::Exception: Received from localhost:9000. DB::Exception: There was an error on [node3:9000]: Code: 336. DB::Exception: MaterializedMySQL is an experimental database engine. Enable allow_experimental_database_materialized_mysql to use it. (UNKNOWN_DATABASE_ENGINE) (version 24.1.8.22 (official build)). (UNKNOWN_DATABASE_ENGINE)
```

        输出：

```null
vvml-yz-hbase-test.172.18.4.188 :) set allow_experimental_database_materialized_mysql=1;SET allow_experimental_database_materialized_mysql = 1Query id: 7ce08dff-8d1e-496f-a1af-39c5bec416430 rows in set. Elapsed: 0.001 sec. vvml-yz-hbase-test.172.18.4.188 :) create database test_mysql on cluster cluster_2S_2Rengine = MaterializeMySQL('172.18.16.156:3307', 'test', 'dba', '123456');CREATE DATABASE test_mysql ON CLUSTER cluster_2S_2RENGINE = MaterializeMySQL('172.18.16.156:3307', 'test', 'dba', '123456')Query id: 610e3c86-c5b6-477c-b4a3-33624809d05c┌─host──┬─port─┬─status─┬─error─┬─num_hosts_remaining─┬─num_hosts_active─┐│ node4 │ 9000 │      0 │       │                   3 │                0 ││ node3 │ 9000 │      0 │       │                   2 │                0 ││ node2 │ 9000 │      0 │       │                   1 │                0 ││ node1 │ 9000 │      0 │       │                   0 │                0 │└───────┴──────┴────────┴───────┴─────────────────────┴──────────────────┘4 rows in set. Elapsed: 0.086 sec. vvml-yz-hbase-test.172.18.4.188 :)
```

        现在可以查询 MySQL 的库表数据：

```null
vvml-yz-hbase-test.172.18.4.188 :) select * from test_mysql.t1;Query id: bae55b87-6e80-4e7f-a2c7-b9312ffef999┌─id─┬─remark───────┬──────────createtime─┐│  1 │ 第一行：row1 │ 2024-04-19 08:46:25 ││  2 │ 第二行：row2 │ 2024-04-19 08:46:25 ││  3 │ 第三行：row3 │ 2024-04-19 08:46:25 │└────┴──────────────┴─────────────────────┘3 rows in set. Elapsed: 0.002 sec. vvml-yz-hbase-test.172.18.4.188 :) 
```

## 五、在 ClickHouse 中创建物化视图

```null
create database db1 on cluster cluster_2S_2R;create table db1.t1 on cluster cluster_2S_2Rengine = ReplicatedMergeTree('/clickhouse/tables/{shard}/t1',create table db1.t1_replica_all ON CLUSTER 'cluster_2S_2R'engine = Distributed(cluster_2S_2R, db1, t1, rand());create materialized view db1.t1_mv on cluster cluster_2S_2Rselect * from test_mysql.t1;
```

        注意创建本地表时的数据类型及其是否允许为空的属性，都要与 MySQL 表的数据类型匹配，否则会报类似下面的错误：

```null
Received exception from server (version 24.1.8):Code: 53. DB::Exception: Received from localhost:9000. DB::Exception: Type mismatch for column id. Column has type Int64, got type UInt64. (TYPE_MISMATCH)Received exception from server (version 24.1.8):Code: 53. DB::Exception: Received from localhost:9000. DB::Exception: Type mismatch for column remark. Column has type Nullable(String), got type String. (TYPE_MISMATCH)
```

        数据类型的对应如下图所示：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-6-4%2010-40-35/fb99eada-b78b-49a1-8bc2-55d73ee1774f.png?raw=true)

## 六、物化视图数据刷新

### 1. 初始数据装载

        ClickHouse 物化视图创建时缺省不会进行初始数据装载。初始装载的方法有两个，一是在创建物化视图时使用 POPULATE。POPULATE 关键字决定了物化视图的更新策略：

-   若有 POPULATE 则在创建视图的过程会将源表已经存在的数据一并导入，类似于 create table ... as
-   若无 POPULATE 则物化视图在创建之后没有数据，只会同步物化视图创建之后写入源表的数据

        ClickHouse 官方并不推荐使用 POPULATE，因为在创建物化视图的过程中同时写入的数据不能被插入物化视图。本例使用 TO \[db].\[table] 语法，物化视图创建后手工执行初始数据装载。

```null
insert into db1.t1_mv(id,remark,createtime) select * from test_mysql.t1;
```

        这么简单的一句却是实现初始数据装载的关键所在。从库停止复制，不影响主库的正常使用，也就不会影响业务。此时从库的数据处于静止状态，不会产生变化，这使得获取存量数据变得轻而易举。然后执行普通的 insert ... select 语句向物化视图插入数据，数据实际是被写入 db1.t1_replica_all 表。之后在 ClickHouse 集群中的任一实例上，都能从物化视图中查询到一致的 MySQL 存量数据。

```null
vvml-yz-hbase-test.172.18.4.86 :) select * from db1.t1_replica_all;Query id: 5063cd12-6eba-4a81-9c75-d892cb152b17┌─id─┬─remark───────┬──────────createtime─┐│  1 │ 第一行：row1 │ 2024-04-19 08:46:25 ││  2 │ 第二行：row2 │ 2024-04-19 08:46:25 ││  3 │ 第三行：row3 │ 2024-04-19 08:46:25 │└────┴──────────────┴─────────────────────┘3 rows in set. Elapsed: 0.004 sec. vvml-yz-hbase-test.172.18.4.86 :) select * from db1.t1_mv;Query id: e319f96b-f299-41a1-b730-737a42ceedb0┌─id─┬─remark───────┬──────────createtime─┐│  1 │ 第一行：row1 │ 2024-04-19 08:46:25 ││  2 │ 第二行：row2 │ 2024-04-19 08:46:25 ││  3 │ 第三行：row3 │ 2024-04-19 08:46:25 │└────┴──────────────┴─────────────────────┘3 rows in set. Elapsed: 0.006 sec. vvml-yz-hbase-test.172.18.4.86 :) 
```

### 2. 增量数据刷新

        ClickHouse 的物化视图能够在底层数据更新后，自动更新自己的数据。数据更新包括两个方面的变化：基础表的数据修改和基础表的数据新增。

-   基础表的数据修改

        如果基础表的数据修改，物化视图会自动更新。这是通过 ClickHouse 的引擎和存储方式来实现的。当基础表的一行记录被修改，ClickHouse 会将这个修改转化为一个新的 INSERT 语句，并且将其发送到物化视图中。这样，物化视图就能够自动更新自己的数据。

-   基础表的数据新增

        如果基础表的数据新增，物化视图同样会自动更新。这是通过设置物化视图的刷新机制来实现的。刷新机制有两种类型：定时刷新和自动刷新。

        需要注意的是，ClickHouse 的物化视图虽然能够自动更新数据，但是会带来一些性能上的损失，尤其是在基础表数据量较大的情况下。因此，在设计物化视图时，需要考虑这个因素，同时选择合适的刷新机制来平衡性能和数据实时性的需求。

        当然既然选择了 ClickHouse，使用场景就应该是数据新增比较多，而极少去修改或删除。对于一般对实时要求不高的业务场景，定时刷新完全够用了。

```null
insert into test.t1 (remark) values ('第四行：row4');update test.t1 set remark = '第五行：row5' where id = 4;delete from test.t1 where id =1;insert into test.t1 (remark) values ('第六行：row6');mysql> select * from test.t1;+----+------------------+---------------------+| id | remark           | createtime          |+----+------------------+---------------------+|  2 | 第二行：row2     | 2024-04-19 08:46:25 ||  3 | 第三行：row3     | 2024-04-19 08:46:25 ||  4 | 第五行：row5     | 2024-04-19 11:24:33 ||  5 | 第六行：row6     | 2024-04-19 11:56:20 |+----+------------------+---------------------+
```

        ClickHouse 查询数据，所有实例上查询物化视图返回相同的数据：

```null
vvml-yz-hbase-test.172.18.4.86 :) select * from db1.t1_mv order by id;Query id: 1e3f8bb3-6fdd-4f3e-a494-af8c72e3dab2┌─id─┬─remark───────┬──────────createtime─┐│  1 │ 第一行：row1 │ 2024-04-19 08:46:25 │└────┴──────────────┴─────────────────────┘┌─id─┬─remark───────┬──────────createtime─┐│  1 │ 第一行：row1 │ 2024-04-19 08:46:25 ││  1 │ 第一行：row1 │ 2024-04-19 08:46:25 ││  1 │ 第一行：row1 │ 2024-04-19 08:46:25 ││  1 │ 第一行：row1 │ 2024-04-19 08:46:25 ││  2 │ 第二行：row2 │ 2024-04-19 08:46:25 ││  3 │ 第三行：row3 │ 2024-04-19 08:46:25 │└────┴──────────────┴─────────────────────┘┌─id─┬─remark───────┬──────────createtime─┐│  4 │ 第四行：row4 │ 2024-04-19 11:24:33 │└────┴──────────────┴─────────────────────┘┌─id─┬─remark───────┬──────────createtime─┐│  4 │ 第五行：row5 │ 2024-04-19 11:24:33 │└────┴──────────────┴─────────────────────┘┌─id─┬─remark───────┬──────────createtime─┐│  4 │ 第四行：row4 │ 2024-04-19 11:24:33 ││  4 │ 第五行：row5 │ 2024-04-19 11:24:33 ││  4 │ 第四行：row4 │ 2024-04-19 11:24:33 ││  4 │ 第四行：row4 │ 2024-04-19 11:24:33 ││  4 │ 第五行：row5 │ 2024-04-19 11:24:33 ││  4 │ 第五行：row5 │ 2024-04-19 11:24:33 │└────┴──────────────┴─────────────────────┘┌─id─┬─remark───────┬──────────createtime─┐│  5 │ 第六行：row6 │ 2024-04-19 11:56:20 │└────┴──────────────┴─────────────────────┘┌─id─┬─remark───────┬──────────createtime─┐│  5 │ 第六行：row6 │ 2024-04-19 11:56:20 │└────┴──────────────┴─────────────────────┘17 rows in set. Elapsed: 0.005 sec. 
```

        查询本地表，同一分片的副本返回相同的结果，不同分片的数据不同：

```null
vvml-yz-hbase-test.172.18.4.126 :) select * from db1.t1 order by id;Query id: c4d50038-f73f-4989-948e-031d6ff5d5ee┌─id─┬─remark───────┬──────────createtime─┐│  1 │ 第一行：row1 │ 2024-04-19 08:46:25 ││  1 │ 第一行：row1 │ 2024-04-19 08:46:25 ││  1 │ 第一行：row1 │ 2024-04-19 08:46:25 ││  1 │ 第一行：row1 │ 2024-04-19 08:46:25 ││  2 │ 第二行：row2 │ 2024-04-19 08:46:25 ││  3 │ 第三行：row3 │ 2024-04-19 08:46:25 ││  4 │ 第四行：row4 │ 2024-04-19 11:24:33 ││  4 │ 第五行：row5 │ 2024-04-19 11:24:33 ││  4 │ 第四行：row4 │ 2024-04-19 11:24:33 ││  4 │ 第四行：row4 │ 2024-04-19 11:24:33 ││  4 │ 第五行：row5 │ 2024-04-19 11:24:33 ││  4 │ 第五行：row5 │ 2024-04-19 11:24:33 │└────┴──────────────┴─────────────────────┘┌─id─┬─remark───────┬──────────createtime─┐│  5 │ 第六行：row6 │ 2024-04-19 11:56:20 │└────┴──────────────┴─────────────────────┘13 rows in set. Elapsed: 0.002 sec. vvml-yz-hbase-test.172.18.4.126 :) vvml-yz-hbase-test.172.18.4.71 :) select * from db1.t1 order by id;Query id: e91e72b7-508a-48a2-b08f-779e49b7cd01┌─id─┬─remark───────┬──────────createtime─┐│  1 │ 第一行：row1 │ 2024-04-19 08:46:25 │└────┴──────────────┴─────────────────────┘┌─id─┬─remark───────┬──────────createtime─┐│  4 │ 第四行：row4 │ 2024-04-19 11:24:33 │└────┴──────────────┴─────────────────────┘┌─id─┬─remark───────┬──────────createtime─┐│  4 │ 第五行：row5 │ 2024-04-19 11:24:33 │└────┴──────────────┴─────────────────────┘┌─id─┬─remark───────┬──────────createtime─┐│  5 │ 第六行：row6 │ 2024-04-19 11:56:20 │└────┴──────────────┴─────────────────────┘4 rows in set. Elapsed: 0.002 sec. vvml-yz-hbase-test.172.18.4.71 :) 
```

        MySQL 中只有三行数据，ClickHouse 却有 17 行。ID=1 的行在 MySQL 中被删除，而在 ClickHouse 中并没有删除，而是变为了 5 行。虽然定义本地表时指定了 id 字段为主键，但 ClickHouse 中自定义的主键并不保证唯一性，即便本地表也是如此。ID=2 和 ID=3 的行在 MySQL 中没有变化，在 ClickHouse 中也分别是唯一的一行。ID=4 的行在 MySQL 中先新增后修改，在 ClickHouse 中都是新增数据。ID=6 的行在 MySQL 中新增一行，在 ClickHouse 中却增加了两行。分布式表的分片规则用的是随机，为什么 MySQL 端新增一条数据，到 ClickHouse 中两个分片都写了呢？

        实验到此实现了数据实时同步，但 ClickHouse 中的数据明显多了很多行，这与选择的表引擎、使用的分片规则都有关系，比较复杂，对数据的解释也变得很重要。所以这里得出的结论是，要用 ClickHouse，最好还是定期从源端导入数据比较靠谱，而且源端最好是只新增数据。

## 参考：

-   [MySQL](https://clickhouse.com/docs/zh/engines/database-engines/mysql "MySQL")
-   [Materialized views](https://clickhouse.com/docs/en/guides/developer/cascading-materialized-views "Materialized views")
-   [\[experimental\] MaterializedMySQL](https://clickhouse.com/docs/en/engines/database-engines/materialized-mysql "\[experimental] MaterializedMySQL")
-   [基于 HBase & Phoenix 构建实时数仓（5）—— 用 Kafka Connect 做实时数据同步](https://blog.csdn.net/wzy0623/article/details/136870007 "基于 HBase & Phoenix 构建实时数仓（5）—— 用 Kafka Connect 做实时数据同步")
-   [Greenplum 实时数据仓库实践（5）——实时数据同步](https://wxy0327.blog.csdn.net/article/details/121973100 "Greenplum 实时数据仓库实践（5）——实时数据同步")
