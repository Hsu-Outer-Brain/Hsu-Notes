# FlinkCDC-Hudi:Mysql数据实时入湖全攻略三：探索实现FlinkCDC mysql 主从库同步高可用 - 知乎
![](https://pic3.zhimg.com/v2-86ef646d037a6efbeb00974dbfeace72_b.jpg)

**前序：Hudi 系列文章：** 

[FlinkCDC-Hudi:Mysql 数据实时入湖全攻略一：Hudi 快速部署与验证](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzU2OTkyNzU0MA%3D%3D%26mid%3D2247483754%26idx%3D1%26sn%3D94166ea1f6302827573a8e26cfdf78a3%26chksm%3Dfcf67371cb81fa6723a45496d07a464a488ed117f3e2edeab81826e75e9baaf923edcc8625e2%26scene%3D21%23wechat_redirect)  
[FlinkCDC-Hudi:Mysql 数据实时入湖全攻略二：Hudi 与 Spark 整合时所遇异常与解决方案](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzU2OTkyNzU0MA%3D%3D%26mid%3D2247483755%26idx%3D1%26sn%3D32e20b2e07d9b5c1364d503f67d53eb9%26chksm%3Dfcf67370cb81fa6636e83e965350a730c6ee835955e84301168176ac30582c9a188f27f4a174%26scene%3D21%23wechat_redirect)  

## **一、背景**

在生产环境中，mysql 一般会配备主从库，以实现数据备份、服务容灾、读写分离等需要。使用 FlinkCdc 进行 mysql 数据入湖时，就不可避免地要和主从库打交道。FlinkCDC 对 mysql 主从库的切换支撑到什么程度、数据库需要怎么配置、同步程序要怎么配合操作和开发，是 FlinkCDC 投入生产应用前必验项目。

本文记录了使用 FlinkCDC 进行 Mysql 主从数据同步的主要验证过程，以为后鉴。

## **二、验证前环境准备**

## **2.1 FlinkCDC+Hudi 环境准备**

FlinkCDC+Hudi 环境准备具体过程参见 [FlinkCDC-Hudi:Mysql 数据实时入湖全攻略一：Hudi 快速部署与验证](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzU2OTkyNzU0MA%3D%3D%26mid%3D2247483754%26idx%3D1%26sn%3D94166ea1f6302827573a8e26cfdf78a3%26chksm%3Dfcf67371cb81fa6723a45496d07a464a488ed117f3e2edeab81826e75e9baaf923edcc8625e2%23rd)，本文不再赘述。

## **2.2 Mysql 主从库环境准备**

准备 Mysql 主从库各一台。  
主库：192.168.2.100  
从库 1：192.168.2.101  
从库 2：192.168.2.102

\*mysql 简易安装命令如下：

```bash
#ubuntu安装命令
sudo apt install mysql-server -y
#linux安装命令
yum install mysql-server
```

更多的安装应用可以参考：  
Ubuntu18.04 安装 MySQL  
Linux 安装 Mysql\*

## **2.3 主从库配置**

mysql 安装完成后，进行主从库配置。mysql 的默认配置文件在 / etc/mysql/my.cnf。主从库相应配置如下。

**注意：这里仅提供了实现主从库的最简配置，仅供测试使用。生产环境切勿直接应用些配置。** 

### **2.3.1 主库配置**

```text
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/

[client]
default-character-set = utf8mb4

[mysql]
default-character-set = utf8mb4

[mysqld]
collation-server = utf8mb4_unicode_ci
init-connect='SET NAMES utf8mb4'
character-set-server = utf8mb4

bind-address = 0.0.0.0
server_id = 1
log-bin = /var/lib/mysql/mysql-bin
#binlog-do-db = *
log-slave-updates
sync_binlog = 1
auto_increment_offset = 1
auto_increment_increment = 1
log_bin_trust_function_creators = 1
```

### **2.3.2 从库安装与配置**

```text
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/
[client]
default-character-set = utf8mb4

[mysql]
default-character-set = utf8mb4

[mysqld]
collation-server = utf8mb4_unicode_ci
init-connect='SET NAMES utf8mb4'
character-set-server = utf8mb4

bind-address = 0.0.0.0
server_id = 2
log-bin = /var/lib/mysql/mysql-bin
log-slave-updates
sync_binlog = 0     
##指定slave要复制哪个库
replicate-do-db = flink_cdc        
##MySQL主从复制的时候，当Master和Slave之间的网络中断，但是Master和Slave无法察觉的情况下（比如防火墙或者路由问题）。Slave会等待slave_net_timeout设置的秒数后，才能认为网络出现故障，然后才会重连并且追赶这段时间主库的数据
slave-net-timeout = 60                    
log_bin_trust_function_creators = 1
read_only = 1
```

### **2.3.3 从库数据初始化**

新安装的主从库由于还没有数据，不需要进行从库数据初始化。  
如果是已存在主库，然后新增从库，则需要进行数据初始化。  
1、在主库中给要同步的数据库加锁

```text
mysql> use flink_cdc;
mysql> flush tables with read lock; 
```

2、在主库 dump 所有数据

```text
mysqldump -uroot -proot_password flink_cdc> flink_cdc.sql
```

3、在第 2 步完成数据 dump 之后，释放锁

4、在从库导入所有数据

```text
mysql> source /path/flink_cdc.sql;
```

### **2.3.4 主库授权**

```text
#创建slave账号user_test，密码user_test_password
mysql> grant select,replication slave,replication client on *.* to 'user_test'@'%' identified by 'user_test_password';
#更新数据库权限
mysql> flush privileges;
#查看master binlog点位
### 如要进行从库初始化，需要在unlock tables之前执行。
mysql> show master status\G;
*************************** 1. row ***************************
             File: mysql-bin.000037   //当前binlog文件
         Position: 700                //当前binlog文件位移
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
1 row in set (0.00 sec)
```

### **2.3.5 开启从库同步**

```text
#执行同步命令，设置主服务器ip，同步账号密码，同步位置
mysql> change master to master_host='192.168.2.100',master_user='user_test',master_password='user_test_password',master_log_file='mysql-bin.000037',master_log_pos=700;
#开启同步功能
mysql> start slave;
#查看slave同步状态
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Connecting to master
                  Master_Host: 192.168.2.100
                  Master_User: user_test
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000037
          Read_Master_Log_Pos: 700
               Relay_Log_File: node-142-relay-bin.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File: mysql-bin.000037
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: flink_cdc
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 700
              Relay_Log_Space: 154
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 0
                  Master_UUID: 
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: 
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61167
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)
```

show slave status 中，Slave_IO_Running 及 Slave_SQL_Running 进程必须是 Yes 状态，才是正常开启了同步。异常情况 Last_IO_Errno，Last_IO_Error， Last_SQL_Errno，Last_SQL_Error 会有相应的提示。设置完成后可以在主库增删数据，在从库验证同步状态。这里便不再展开。

## **三、FlinkCDC 从库同步入湖验证**

## **3.1 FlinkCDC-Hudi 运行环境**

这里假定读者已经搭建好 FlinkCDC+Hudi 环境，本文操作只记录验证 FlinkCDC 同步 mysql 主从数据的必要步骤。

FlinkCDC+Hudi 环境准备具体过程参见 [FlinkCDC-Hudi:Mysql 数据实时入湖全攻略一：Hudi 快速部署与验证](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzU2OTkyNzU0MA%3D%3D%26mid%3D2247483754%26idx%3D1%26sn%3D94166ea1f6302827573a8e26cfdf78a3%26chksm%3Dfcf67371cb81fa6723a45496d07a464a488ed117f3e2edeab81826e75e9baaf923edcc8625e2%26scene%3D21%23wechat_redirect)

### **3.1.1 mysql 测试表准备**

在主库中创建测试表 1

```text
mysql> use flink_cdc;
mysql> CREATE TABLE `test_1` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `data` varchar(10) DEFAULT NULL,
  `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4; 
## 重复执行，插入多条数据
mysql> insert into test_1(data) values('data');
```

### **3.1.2 使用 FlinkSql 启动同步作业**

**3.1.2.1 启动 FlinkSql**

```text
FLINK_HOME/bin/yarn-session.sh -s 4 -jm 1024 -tm 2048 -nm flink-hudi-0.10 -d
FLINK_HOME/bin/sql-client.sh embedded -s yarn-session -j ./lib/hudi-flink-bundle_2.11-0.10.0.jar shell
```

**3.1.2.2 在 FlinkSql 中启动作业**

```text
Flink SQL> set execution.checkpointing.interval=30sec;

##创建flinksql mysqlcdc表
Flink SQL> create table mysql_test_1(
id bigint primary key not enforced,
data String,
create_time Timestamp(3)
) with (
'connector'='mysql-cdc',
'hostname'='192.168.2.101',    --从从库1开始同步，切换为主库或从库2时修改为对应的ip
'port'='3306',
'server-id'='5600-5604',
'username'='user_test',
'password'='user_test_password',
'server-time-zone'='Asia/Shanghai',
'debezium.snapshot.mode'='initial',
'database-name'='flink_cdc',
'table-name'='test_1'
)  

##创建flinksql hudi表
Flink SQL> create table hudi_test_1(
id bigint,
data String,
create_time Timestamp(3),
PRIMARY KEY (`id`) NOT ENFORCED
)
with(
'connector'='hudi',
'path'='hdfs:///tmp/flink/cdcata/hudi_test_1',
'hoodie.datasource.write.recordkey.field'='id',
'hoodie.parquet.max.file.size'='268435456',
'write.precombine.field'='create_time',
'write.tasks'='1',
'write.bucket_assign.tasks'='1',
'write.task.max.size'='1024',
'write.rate.limit'='30000',
'table.type'='MERGE_ON_READ',
'compaction.tasks'='1',
'compaction.async.enabled'='true',
'compaction.delta_commits'='1',
'compaction.max_memory'='500',
'changelog.enabled'='true',
'read.streaming.enabled'='true',
'read.streaming.check.interval'='3',
'hive_sync.enable'='true',
'hive_sync.mode'='hms',
'hive_sync.metastore.uris'='thrift://hiveserver2:9083',
'hive_sync.db'='test',
'hive_sync.table'='hudi_test_1',
'hive_sync.username'='flinkcdc',
'hive_sync.support_timestamp'='true'
);
##启动flink作业
Flink SQL> set pipeline.name = flinkcdc_test_1;
Flink SQL> set execution.checkpointing.externalized-checkpoint-retention= RETAIN_ON_CANCELLATION;
Flink SQL> insert into hudi_test_1 select * from mysql_test_1;

Flink SQL> set execution.checkpointing.interval=30sec;

##创建flinksql mysqlcdc表
Flink SQL> create table mysql_test_1(
id bigint primary key not enforced,
data String,
create_time Timestamp(3)
) with (
'connector'='mysql-cdc',
'hostname'='192.168.2.101',    --从从库1开始同步，切换为主库或从库2时修改为对应的ip
'port'='3306',
'server-id'='5600-5604',
'username'='user_test',
'password'='user_test_password',
'server-time-zone'='Asia/Shanghai',
'debezium.snapshot.mode'='initial',
'database-name'='flink_cdc',
'table-name'='test_1'
)  

##创建flinksql hudi表
Flink SQL> create table hudi_test_1(
id bigint,
data String,
create_time Timestamp(3),
PRIMARY KEY (`id`) NOT ENFORCED
)
with(
'connector'='hudi',
'path'='hdfs:///tmp/flink/cdcata/hudi_test_1',
'hoodie.datasource.write.recordkey.field'='id',
'hoodie.parquet.max.file.size'='268435456',
'write.precombine.field'='create_time',
'write.tasks'='1',
'write.bucket_assign.tasks'='1',
'write.task.max.size'='1024',
'write.rate.limit'='30000',
'table.type'='MERGE_ON_READ',
'compaction.tasks'='1',
'compaction.async.enabled'='true',
'compaction.delta_commits'='1',
'compaction.max_memory'='500',
'changelog.enabled'='true',
'read.streaming.enabled'='true',
'read.streaming.check.interval'='3',
'hive_sync.enable'='true',
'hive_sync.mode'='hms',
'hive_sync.metastore.uris'='thrift://hiveserver2:9083',
'hive_sync.db'='test',
'hive_sync.table'='hudi_test_1',
'hive_sync.username'='flinkcdc',
'hive_sync.support_timestamp'='true'
);
##启动flink作业
Flink SQL> set pipeline.name = flinkcdc_test_1;
Flink SQL> set execution.checkpointing.externalized-checkpoint-retention= RETAIN_ON_CANCELLATION;
Flink SQL> insert into hudi_test_1 select * from mysql_test_1;
```

**3.1.2.3 成功启动作业**

![](https://pic2.zhimg.com/v2-3fd09b75008f68028114c7a379bda4f9_b.jpg)

**3.1.2.4 在 FlinkSQL 中验证 hudi 落地的数据。** 

```text
Flink SQL> select * from hudi_test_1;

```

![](https://pic3.zhimg.com/v2-df856ece2661f147eba848ca7790463a_b.jpg)

## **3.2 模拟从库数据同步异常**

目前我们搭建的环境是一主二从，这里验证从一异常，切换到从二。当然也可以切到主，切换验证的流程是一样的。但一般生产环境不会直接从主库同步数据。

### **3.2.1 模拟从库异常**

这里我们直接停掉从库 1，模拟从库宕机。在从库 1 中关掉 mysql。

这时 flink 作业会挂掉。  

![](https://pic4.zhimg.com/v2-a5156ed4883d570dddf08d5f7513385f_b.jpg)

查看 flink 作业的异常记录，可以观察到两种异常。  
一种是服务连接超时，没有收到 mysql 服务端的响应。

```text
-- The last packet sent successfully to the server was 0 milliseconds ago. The driver has not received any packets from the server.
```

另一种是 Flink 作业不断重启产生的 One or more fetchers have encountered exception。由于 flinkcdc 通过模拟 slave 的行为来同步 binlog，每个 slave 都会有一个 serverid（在 flinksql mysql 表的定义中配置，’server-id’=’5600-5604’），作业重启时会使用相同的 serverid，当重启 flink 作业的间隔小于 flink 与 mysql 连接的超时时长会，会报这个异常。

这两个异常的详细异常栈信息如下：

```text
The last packet sent successfully to the server was 0 milliseconds ago. The driver has not received any packets from the server.
	at com.mysql.cj.jdbc.exceptions.SQLError.createCommunicationsException(SQLError.java:174)
	at com.mysql.cj.jdbc.exceptions.SQLExceptionsMapping.translateException(SQLExceptionsMapping.java:64)
	at com.mysql.cj.jdbc.ConnectionImpl.createNewIO(ConnectionImpl.java:836)
	at com.mysql.cj.jdbc.ConnectionImpl.<init>(ConnectionImpl.java:456)
	at com.mysql.cj.jdbc.ConnectionImpl.getInstance(ConnectionImpl.java:246)
	at com.mysql.cj.jdbc.NonRegisteringDriver.connect(NonRegisteringDriver.java:197)
	at io.debezium.jdbc.JdbcConnection.lambda$patternBasedFactory$1(JdbcConnection.java:231)
	at io.debezium.jdbc.JdbcConnection.connection(JdbcConnection.java:872)
	at io.debezium.connector.mysql.MySqlConnection.connection(MySqlConnection.java:79)
	at io.debezium.jdbc.JdbcConnection.connection(JdbcConnection.java:867)
	at io.debezium.jdbc.JdbcConnection.query(JdbcConnection.java:550)
	at io.debezium.jdbc.JdbcConnection.query(JdbcConnection.java:498)
	at io.debezium.connector.mysql.MySqlConnection.querySystemVariables(MySqlConnection.java:125)
	... 15 more
Caused by: com.mysql.cj.exceptions.CJCommunicationsException: Communications link failure

The last packet sent successfully to the server was 0 milliseconds ago. The driver has not received any packets from the server.
	at sun.reflect.GeneratedConstructorAccessor47.newInstance(Unknown Source)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at com.mysql.cj.exceptions.ExceptionFactory.createException(ExceptionFactory.java:61)
	at com.mysql.cj.exceptions.ExceptionFactory.createException(ExceptionFactory.java:105)
	at com.mysql.cj.exceptions.ExceptionFactory.createException(ExceptionFactory.java:151)
	at com.mysql.cj.exceptions.ExceptionFactory.createCommunicationsException(ExceptionFactory.java:167)
	at com.mysql.cj.protocol.a.NativeSocketConnection.connect(NativeSocketConnection.java:91)
	at com.mysql.cj.NativeSession.connect(NativeSession.java:144)
	at com.mysql.cj.jdbc.ConnectionImpl.connectOneTryOnly(ConnectionImpl.java:956)
	at com.mysql.cj.jdbc.ConnectionImpl.createNewIO(ConnectionImpl.java:826)
	... 25 more
Caused by: java.net.ConnectException: Connection refused (Connection refused)
	at java.net.PlainSocketImpl.socketConnect(Native Method)
	at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)
	at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206)
	at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188)
	at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
	at java.net.Socket.connect(Socket.java:589)
	at com.mysql.cj.protocol.StandardSocketFactory.connect(StandardSocketFactory.java:155)
	at com.mysql.cj.protocol.a.NativeSocketConnection.connect(NativeSocketConnection.java:65)
	... 28 more

2022-02-15 19:00:54
java.lang.RuntimeException: One or more fetchers have encountered exception
	at org.apache.flink.connector.base.source.reader.fetcher.SplitFetcherManager.checkErrors(SplitFetcherManager.java:223)
	at org.apache.flink.connector.base.source.reader.SourceReaderBase.getNextFetch(SourceReaderBase.java:154)
	at org.apache.flink.connector.base.source.reader.SourceReaderBase.pollNext(SourceReaderBase.java:116)
	at org.apache.flink.streaming.api.operators.SourceOperator.emitNext(SourceOperator.java:294)
	at org.apache.flink.streaming.runtime.io.StreamTaskSourceInput.emitNext(StreamTaskSourceInput.java:69)
	at org.apache.flink.streaming.runtime.io.StreamOneInputProcessor.processInput(StreamOneInputProcessor.java:66)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.processInput(StreamTask.java:423)
	at org.apache.flink.streaming.runtime.tasks.mailbox.MailboxProcessor.runMailboxLoop(MailboxProcessor.java:204)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.runMailboxLoop(StreamTask.java:684)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.executeInvoke(StreamTask.java:639)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.runWithCleanUpOnFail(StreamTask.java:650)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.invoke(StreamTask.java:623)
	at org.apache.flink.runtime.taskmanager.Task.doRun(Task.java:779)
	at org.apache.flink.runtime.taskmanager.Task.run(Task.java:566)
	at java.lang.Thread.run(Thread.java:748)
Caused by: java.lang.RuntimeException: SplitFetcher thread 0 received unexpected exception while polling the records
	at org.apache.flink.connector.base.source.reader.fetcher.SplitFetcher.runOnce(SplitFetcher.java:148)
	at org.apache.flink.connector.base.source.reader.fetcher.SplitFetcher.run(SplitFetcher.java:103)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	... 1 more
Caused by: io.debezium.DebeziumException: Error reading MySQL variables: Communications link failure

```

### **3.2.2 人为制造从库状态同步不一致**

这时持续对主库写入多条数据，这样两个从库的状态就会出现不一致。  
a. 通过 show slave status\\G 查看从库已从主库同步的数据点位。

```text
Master_Log_File: mysql-bin.000039
Read_Master_Log_Pos: 2759
Relay_Log_File: node-142-relay-bin.000006
Relay_Log_Pos: 2885
Relay_Master_Log_File: mysql-bin.000039
```

b. 这里只能查询从库二的状态，因为从库一挂了。但我们通过两个从库生成的 binlog 文件来比较同步的状态。可以简单地查看 binlog 文件的修改时间比较同步差异。  
查询 mysql 数据保存的目录：

```text
mysql> show variables like '%dir%';
+-----------------------------------------+----------------------------+
| Variable_name                           | Value                      |
+-----------------------------------------+----------------------------+
| datadir                                 | /var/lib/mysql/            |
```

ll 数据目录看看 binlog 文件生成的状态。

查看从库 1 的 binlog 状态：

```text
-rw-r----- 1 mysql mysql      241 Feb 15 00:10 mysql-bin.000035
-rw-r----- 1 mysql mysql      723 Feb 15 17:08 mysql-bin.000036
-rw-r----- 1 mysql mysql     2786 Feb 15 18:57 mysql-bin.000037
-rw-r----- 1 mysql mysql      488 Feb 16 10:22 mysql-bin.000038
-rw-r----- 1 mysql mysql      759 Feb 16 10:37 mysql-bin.000039
-rw-r----- 1 mysql mysql      247 Feb 16 10:29 mysql-bin.index
```

查看从库 2 的 binlog 状态：

```text
-rw-r----- 1 mysql mysql      177 Feb 15 15:12 mysql-bin.000001
-rw-r----- 1 mysql mysql     2770 Feb 16 00:10 mysql-bin.000002
-rw-r----- 1 mysql mysql     2593 Feb 16 10:50 mysql-bin.000003
-rw-r----- 1 mysql mysql       96 Feb 16 00:10 mysql-bin.index
```

从库 1 的 mysql-bin.000039 在 10：37 已经停止更新，而从库二的 mysql-bin.000003 持续在更新。

通过 mysqlbinlog 来读取文件比较差异。笔者这里从库一远早于从库二的搭建，所以从库一的 binlog 文件序号更大。  
从库一：

```text
mysqlbinlog --start-datetime='2022-02-16 10:00:00' /var/lib/mysql/mysql-bin.000039

# at 705
#220216 10:37:15 server id 1  end_log_pos 736 CRC32 0x8523929e 	Xid = 183
COMMIT/*!*/;
# at 736
#220216 10:37:48 server id 2  end_log_pos 759 CRC32 0x238de33d 	Stop
```

从库二：

```text
mysqlbinlog --start-datetime='2022-02-16 10:00:00'  --stop-datetime='2020-02-16 11:00:00'  --no-defaults /var/lib/mysql/mysql-bin.000003

#220216 10:50:42 server id 1  end_log_pos 2562 CRC32 0x5e8d4964 	Write_rows: table id 156 flags: STMT_END_F

BINLOG '
gmYMYhMBAAAAOwAAAM0JAAAAAJwAAAAAAAEACWZsaW5rX2NkYwAGdGVzdF8xAAMIDxEDKAAAAhD4
QQc=
gmYMYh4BAAAANQAAAAIKAAAAAJwAAAAAAAEAAgAD//gQAAAAAAAAAARkYXRhYgxmgmRJjV4=
'/*!*/;
# at 2562
#220216 10:50:42 server id 1  end_log_pos 2593 CRC32 0xbc78edd3 	Xid = 486
COMMIT/*!*/;
# at 2593
#220216 11:39:28 server id 3  end_log_pos 2616 CRC32 0x3fe31a47 	Stop
```

## **3.3 FlinkCDC mysql 高可用第一次验证**

### **3.3.1 从作业 checkpoint 信息中获取最后一次的 checkpoint 信息**

![](https://pic1.zhimg.com/v2-a8059e78468913e8106af46dd6b158cc_b.jpg)

### **3.3.2 切换到重库 2**

把 mysql_test_1 的 hostname 切为从库 2。

```text
Flink SQL> drop table mysql_test_1;
Flink SQL> create table mysql_test_1(
> id bigint primary key not enforced,
> data String,
> create_time Timestamp(3)
> ) with (
> 'connector'='mysql-cdc',
> 'hostname'='192.168.2.102',
> 'port'='3306',
> 'server-id'='5600-5604',
> 'username'='user_flink',
> 'password'='flink@testdb',
> 'server-time-zone'='Asia/Shanghai',
> 'debezium.snapshot.mode'='initial',
> 'database-name'='flink_cdc',
> 'table-name'='test_1'
> );
```

### **3.3.3 通过 checkpoint 重启**

```text
Flink SQL> set 'execution.savepoint.path'='hdfs:///tmp/flink/checkpoints/88541fd8a08e1cee71aac55d2f39951f/chk-3'
Flink SQL> insert into hudi_test_1 select * from mysql_test_1;
```

这时候在 flink web ui 上查看新启动的作业，发现作业重启是失败的。查看异常栈，提示 binlog 的文件已经不存在，无法正常启动作业。

```text
 The connector is trying to read binlog starting at Struct{version=1.5.4.Final,connector=mysql,name=mysql_binlog_source,ts_ms=1644982889087,db=,server_id=0,file=mysql-bin.000039,pos=530,row=0}, but this is no longer available on the server. Reconfigure the connector to use a snapshot when needed.
```

异常信息里提示的 “file=mysql-bin.000039,pos=530” 实际上是从库一生成的 binlog 文件与 flinkcdc 已经同步到的点位。这是 checkpoint 时保存下来的同步状态。  
而我们在前面的 Binlog 文件分析中知道，从库二的最新 Binlog 是 mysql-bin.000003。这样我们就发现从库一和从库二的 binlog 文件不一致，是无法直接从 checkpoint 中进行主从切换的。

异常栈详细信息如下：

```text
2022-02-16 11:41:29
java.lang.RuntimeException: One or more fetchers have encountered exception
	at org.apache.flink.connector.base.source.reader.fetcher.SplitFetcherManager.checkErrors(SplitFetcherManager.java:223)
	at org.apache.flink.connector.base.source.reader.SourceReaderBase.getNextFetch(SourceReaderBase.java:154)
	at org.apache.flink.connector.base.source.reader.SourceReaderBase.pollNext(SourceReaderBase.java:116)
	at org.apache.flink.streaming.api.operators.SourceOperator.emitNext(SourceOperator.java:294)
	at org.apache.flink.streaming.runtime.io.StreamTaskSourceInput.emitNext(StreamTaskSourceInput.java:69)
	at org.apache.flink.streaming.runtime.io.StreamOneInputProcessor.processInput(StreamOneInputProcessor.java:66)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.processInput(StreamTask.java:423)
	at org.apache.flink.streaming.runtime.tasks.mailbox.MailboxProcessor.runMailboxLoop(MailboxProcessor.java:204)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.runMailboxLoop(StreamTask.java:684)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.executeInvoke(StreamTask.java:639)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.runWithCleanUpOnFail(StreamTask.java:650)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.invoke(StreamTask.java:623)
	at org.apache.flink.runtime.taskmanager.Task.doRun(Task.java:779)
	at org.apache.flink.runtime.taskmanager.Task.run(Task.java:566)
	at java.lang.Thread.run(Thread.java:748)
Caused by: java.lang.RuntimeException: SplitFetcher thread 0 received unexpected exception while polling the records
	at org.apache.flink.connector.base.source.reader.fetcher.SplitFetcher.runOnce(SplitFetcher.java:148)
	at org.apache.flink.connector.base.source.reader.fetcher.SplitFetcher.run(SplitFetcher.java:103)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	... 1 more
Caused by: java.lang.IllegalStateException: The connector is trying to read binlog starting at Struct{version=1.5.4.Final,connector=mysql,name=mysql_binlog_source,ts_ms=1644982889087,db=,server_id=0,file=mysql-bin.000039,pos=530,row=0}, but this is no longer available on the server. Reconfigure the connector to use a snapshot when needed.
	at com.ververica.cdc.connectors.mysql.debezium.task.context.StatefulTaskContext.loadStartingOffsetState(StatefulTaskContext.java:179)
	at com.ververica.cdc.connectors.mysql.debezium.task.context.StatefulTaskContext.configure(StatefulTaskContext.java:113)
	at com.ververica.cdc.connectors.mysql.debezium.reader.BinlogSplitReader.submitSplit(BinlogSplitReader.java:93)
	at com.ververica.cdc.connectors.mysql.debezium.reader.BinlogSplitReader.submitSplit(BinlogSplitReader.java:65)
	at com.ververica.cdc.connectors.mysql.source.reader.MySqlSplitReader.checkSplitOrStartNext(MySqlSplitReader.java:147)
	at com.ververica.cdc.connectors.mysql.source.reader.MySqlSplitReader.fetch(MySqlSplitReader.java:69)
	at org.apache.flink.connector.base.source.reader.fetcher.FetchTask.run(FetchTask.java:56)
	at org.apache.flink.connector.base.source.reader.fetcher.SplitFetcher.runOnce(SplitFetcher.java:140)
	... 6 more
```

## **3.4 FlinkCDC mysql 高可用第二次验证**

根据最新的 FlinkCDC mysql connector 官方文档，mysqlcdc 是支持高可用的。为什么会验证失败呢？我们深挖官方文档发现，要实现高可用，mysql 要开启 gtid。

### **3.4.1 GTID 简介**

### **3.4.1.1 GTID 是什么**

Gtid 是 mysql 在 5.7 引入的新特征，用于解决使用 binlog - potition 机制进行主从同步时缺憾：就像我们前面验证的一样，无法应对高可用！

GTID (Global Transaction ID) 是全局事务 ID, 由主库上生成的与事务绑定的唯一标识，这个标识不仅在主库上是唯一的，在 MySQL 集群内也是唯一的。  
GTID 由 server_uuid:transaction_id 组成。server_uuid 是 mysql 实例的 uuid，transaction_id 代表该实例上执行的事务数量，随事务执行的数量自增。

### **3.4.1.2 GTID 示例**

下面是一个 gtid 示例：

```text
d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61169
```

其中 d667b1cd-778a-11ec-aa60-6c92bf64e18c 是服务 uuid，  
1-61169 代码该实例执行了执行为 id 从 1 到 61169 的 61169 个事务。

### **3.4.1.3 GTID 的寻址方式**

开启 gtid 后，会在 binlog 文件头部加上前一个 binlog 文件的 gtid 范围，寻址时通过比较 gtid 的范围，确认所要同步的 gtid 所在 binlog 的位置。示意如下：

```text
mysql-binlog.00001 previous-gtids:empty
mysql-binlog.00002 previous-gtids:1-100
mysql-binlog.00003 previous-gtids:101-200
```

寻找 gtid=50，会从最新的 binlog 文件 mysql-binlog.00003 读取 previous-gtids:101-200，比较 50 与 101-200，确认 50 比 101 小，则往前两个文件查找，即从 mysql-binlog.00001 查找。

补充：  
更多的 gtid 知识可以参见：  
MySQL5.7 杀手级新特性：GTID 原理与实战

下面我们踏上 gtid 高可用验证之路。

### **3.4.2 mysql 开启 gtid**

对应 mysql 的配置文件（/etc/msyql/my.cnf）里添加以下配置：  
主库配置：

```text
gtid_mode = on
enforce_gtid_consistency = on
```

从库配置：

```text
gtid_mode = on
enforce_gtid_consistency = on
log-slave-updates = 1
```

修改完配置后，重启 mysql 服务：

通过 mysql-cli 连接从库，show slave status\\G 查看从库 slave 的同步状态。  
从库一模拟异常时与主库有同步延迟。开启 gtid 和没开启 gtid 的 binlog 解析是不一样的。这时从库一会显示 Slave_IO_Running=No，Last_IO_Error 提示解析失败。

```text
Slave_IO_Running: No
Slave_SQL_Running: Yes
Last_IO_Error: Got fatal error 1236 from master when reading data from binary log: 'Cannot replicate anonymous transaction when @@GLOBAL.GTID_MODE = ON, at file /var/lib/mysql/mysql-bin.000039, position 1049.; the first event 'mysql-bin.000039' at 1049, the last event read from '/var/lib/mysql/mysql-bin.000039' at 1114, the last byte read from '/var/lib/mysql/mysql-bin.000039' at 1114.'
```

这时可以回滚主从库的配置，重启从库一，让从库一在关闭 gtid_mode 完成同步后，再更新配置。  
如果在更新配置过程中，主从库 gtid_mode 不是同样的状态，会报以下错误。  
将状态修改为同一样状态即可。

```text
Slave_IO_Running: No
Slave_SQL_Running: Yes
Last_IO_Error: The replication receiver thread cannot start because the master has GTID_MODE = ON and this server has GTID_MODE = OFF.
```

顺利开启 gtid mode 之后，每次更新数据语句执行时都会生成一个 gtid。我们可以通过 show master/slave status 查看 gtid 的同步状态。  
master 状态：

```text
mysql> show master status\G;
*************************** 1. row ***************************
             File: mysql-bin.000042
         Position: 764
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61169
```

从库一状态：

```text
mysql> show slave status\G;
##只截取gtid相关的信息
*************************** 1. row ***************************
                  Master_UUID: d667b1cd-778a-11ec-aa60-6c92bf64e18c
           Retrieved_Gtid_Set: d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61169
            Executed_Gtid_Set: d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61169
1 row in set (0.00 sec)
```

从库二状态：

```text
mysql> show slave status\G;
*************************** 1. row ***************************
                  Master_UUID: d667b1cd-778a-11ec-aa60-6c92bf64e18c
           Retrieved_Gtid_Set: d667b1cd-778a-11ec-aa60-6c92bf64e18c:61168-61169
            Executed_Gtid_Set: d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61169
```

可见主从库数据同步都到了最新的 gtid:d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61169。

### **3.4.3 重启作业生成含 gtid 的 checkpoint**

将 mysql_test_1 的 Host 设置为从库一，让作业从从库一生成的 checkpoint 中恢复。作业成功运行后 kill 掉从库一，这时作业会报错，现象与前述过程一致，不再赘述。

注意：  
线上环境中，如果故障没有恢复时更多的是从新的从库中同步。

### **3.4.4 从含 gtid 的 checkpoint 中恢复**

我们从 web ui 中拿到含有 gtid 的 checkpoint 进行作业重启，发现依然是报 binlog 文件异常。

```text
Caused by: java.lang.IllegalStateException: The connector is trying to read binlog starting at Struct{version=1.5.4.Final,connector=mysql,name=mysql_binlog_source,ts_ms=1645074443471,db=,server_id=0,file=mysql-bin.000044,pos=194,row=0}, but this is no longer available on the server. Reconfigure the connector to use a snapshot when needed.
	at com.ververica.cdc.connectors.mysql.debezium.task.context.StatefulTaskContext.loadStartingOffsetState(StatefulTaskContext.java:179)
	at com.ververica.cdc.connectors.mysql.debezium.task.context.StatefulTaskContext.configure(StatefulTaskContext.java:113)
	at com.ververica.cdc.connectors.mysql.debezium.reader.BinlogSplitReader.submitSplit(BinlogSplitReader.java:93)
	at com.ververica.cdc.connectors.mysql.debezium.reader.BinlogSplitReader.submitSplit(BinlogSplitReader.java:65)
	at com.ververica.cdc.connectors.mysql.source.reader.MySqlSplitReader.checkSplitOrStartNext(MySqlSplitReader.java:147)
	at com.ververica.cdc.connectors.mysql.source.reader.MySqlSplitReader.fetch(MySqlSplitReader.java:69)
	at org.apache.flink.connector.base.source.reader.fetcher.FetchTask.run(FetchTask.java:56)
	at org.apache.flink.connector.base.source.reader.fetcher.SplitFetcher.runOnce(SplitFetcher.java:140)
	... 6 more
```

### **3.4.5 异常源码分析**

是 gtid 没有生效？查看 taskmanager 的日志，checkpoint 中已经包含 gtid 信息，也传递给了 SourceReaderBase。  

```text
2022-02-17 13:00:54,205 INFO  org.apache.flink.connector.base.source.reader.SourceReaderBase [] - Adding split(s) to reader: [MySqlBinlogSplit{splitId='binlog-split', offset={ts_sec=0, file=mysql-bin.000044, pos=194, gtids=d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61239, row=0, event=0}, endOffset={ts_sec=0, file=, pos=-9223372036854775808, row=0, event=0}}]
```

再查异常栈里的代码, StatefulTaskContext 里 loadStartingOffsetState 判断 isBinlogAvailable，binlog 不可用就抛异常。

```java
    private MySqlOffsetContext loadStartingOffsetState(
            OffsetContext.Loader loader, MySqlSplit mySqlSplit) {
        BinlogOffset offset =
                mySqlSplit.isSnapshotSplit()
                        ? BinlogOffset.INITIAL_OFFSET
                        : mySqlSplit.asBinlogSplit().getStartingOffset();

        MySqlOffsetContext mySqlOffsetContext =
                (MySqlOffsetContext) loader.load(offset.getOffset());

        if (!isBinlogAvailable(mySqlOffsetContext)) {
            throw new IllegalStateException(
                    "The connector is trying to read binlog starting at "
                            + mySqlOffsetContext.getSourceInfo()
                            + ", but this is no longer "
                            + "available on the server. Reconfigure the connector to use a snapshot when needed.");
        }
        return mySqlOffsetContext;
    }
```

而 isBinlogAvailable 只判断了是否找到文件名，而没有判断 gtidset。

```text
    private boolean isBinlogAvailable(MySqlOffsetContext offset) {
        String binlogFilename = offset.getSourceInfo().getString(BINLOG_FILENAME_OFFSET_KEY);
        if (binlogFilename == null) {
            return true; // start at current position
        }
        if (binlogFilename.equals("")) {
            return true; // start at beginning
        }

        // Accumulate the available binlog filenames ...
        List<String> logNames = connection.availableBinlogFiles();

        // And compare with the one we're supposed to use ...
        boolean found = logNames.stream().anyMatch(binlogFilename::equals);
        if (!found) {
            LOG.info(
                    "Connector requires binlog file '{}', but MySQL only has {}",
                    binlogFilename,
                    String.join(", ", logNames));
        } else {
            LOG.info("MySQL has the binlog file '{}' required by the connector", binlogFilename);
        }
        return found;
    }
```

### **3.4.6 gtid bugfix**

前面在 3.4.1.3 讲述了 Gtid 的寻址方式，mysql 高可以模式下，切换主库时，slave 会根据已同步到的 gtid，对比 master binlog 文件的 previous gtidset，最终确认要从新 master 的哪个 binlog 文件开始同步。而源码根本没有这样一个寻址步骤。

所以很确认的是，flinkcdc bug 了。上 github 提 bug 去。gtid 没有生效

根据 githup 上的反馈，有位兄弟 21 年年底已经发现了这个 bug，并在一月份提交了 bugfix\[mysql] update check gtid set #761。但目前的修改代码还没有合入 master 分支。

查看对方修改的代码，在 isBinlogAvailable 增加了 checkGtidSet

```text
    private boolean isBinlogAvailable(MySqlOffsetContext offset) {
        String gtidStr = offset.gtidSet();
        if (gtidStr != null) {
            return checkGtidSet(offset);
        }
        return checkBinlogFilename(offset);
    }
```

checkGtidSet 里做了以下几件事：  
1、查询了 master 的 gtidset—availableGtidStr ，  
2、和 checkpoint 的 gtidset 对比，计算出需要同步的 gtid 范围 gtidSetToReplicate。  
3、查询是否清除过 gtid—purgedGtidSet，和 gtidSetToReplicate 对比计算出没有清除的范围。确认没有清除过，就从 checkpoint 中续点同步，否则重新开始全量同步。

```text
    private boolean checkGtidSet(MySqlOffsetContext offset) {
        String gtidStr = offset.gtidSet();

        if (gtidStr.trim().isEmpty()) {
            return true; // start at beginning ...
        }

        String availableGtidStr = connection.knownGtidSet();
        if (availableGtidStr == null || availableGtidStr.trim().isEmpty()) {
            // Last offsets had GTIDs but the server does not use them ...
            LOG.warn(
                    "Connector used GTIDs previously, but MySQL does not know of any GTIDs or they are not enabled");
            return false;
        }
        // GTIDs are enabled
        GtidSet gtidSet = new GtidSet(gtidStr);
        // Get the GTID set that is available in the server ...
        GtidSet availableGtidSet = new GtidSet(availableGtidStr);
        if (gtidSet.isContainedWithin(availableGtidSet)) {
            LOG.info(
                    "MySQL current GTID set {} does contain the GTID set {} required by the connector.",
                    availableGtidSet,
                    gtidSet);
            // The replication is concept of mysql master-slave replication protocol ...
            final GtidSet gtidSetToReplicate =
                    connection.subtractGtidSet(availableGtidSet, gtidSet);
            final GtidSet purgedGtidSet = connection.purgedGtidSet();
            LOG.info("Server has already purged {} GTIDs", purgedGtidSet);
            final GtidSet nonPurgedGtidSetToReplicate =
                    connection.subtractGtidSet(gtidSetToReplicate, purgedGtidSet);
            LOG.info(
                    "GTID set {} known by the server but not processed yet, for replication are available only GTID set {}",
                    gtidSetToReplicate,
                    nonPurgedGtidSetToReplicate);
            if (!gtidSetToReplicate.equals(nonPurgedGtidSetToReplicate)) {
                LOG.warn("Some of the GTIDs needed to replicate have been already purged");
                return false;
            }
            return true;
        }
        LOG.info("Connector last known GTIDs are {}, but MySQL has {}", gtidSet, availableGtidSet);
        return false;
    }

```

### **3.4.7 合并 bugfix 代码，验证高可用**

从 gitbub 中下载代码（[https://github.com/ververica/flink-cdc-connectors.git](https://link.zhihu.com/?target=https%3A//github.com/ververica/flink-cdc-connectors.git)），将 bugfix--\[gtid 没有生效]([https://github.com/ververica/flink-cdc-connectors/issues/845](https://link.zhihu.com/?target=https%3A//github.com/ververica/flink-cdc-connectors/issues/845)) 的代码修改入 master，编译打包，更新到 flink 环境中，从走上述验证过程， 观察到：  
1、切换主从时，没有再报 binlog 文件找不到异常。  
2、作业从 checkpoint 中开始同步，而不是全量。  
现象表明，mysqlcdc 高可用切换成功。

我们查看运行日志，进一步验证：  
1、获取初始 gtids=d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61282  
2、确认 MySQL 当前 GTID set d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61290 包含初始 GTID set d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61282。  
3、计算可同步的 GTID set d667b1cd-778a-11ec-aa60-6c92bf64e18c:61283-61290。  
4、设置同步起点开始同步。

详细日志如下`:`

```text
2022-02-17 20:05:55,961 INFO  org.apache.flink.connector.base.source.reader.SourceReaderBase [] - Adding split(s) to reader: [MySqlBinlogSplit{splitId='binlog-split', offset={ts_sec=0, file=mysql-bin.000005, pos=31359, gtids=d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61282, row=0, event=0}, endOffset={ts_sec=0, file=, pos=-9223372036854775808, row=0, event=0}, isSuspended=false}]
2022-02-17 20:05:55,966 INFO  org.apache.flink.connector.base.source.reader.fetcher.SplitFetcher [] - Starting split fetcher 0
2022-02-17 20:05:55,970 INFO  org.apache.flink.runtime.taskmanager.Task                    [] - Source: TableSourceScan(table=[[default_catalog, default_database, mysql_test_1]], fields=[id, data, create_time]) -> NotNullEnforcer(fields=[id]) -> Map (1/1)#0 (7fdfa75135e212f596f80bd2716e0837) switched from INITIALIZING to RUNNING.
com.ververica.cdc.connectors.mysql.source.reader.MySqlSplitReader [] - BinlogSplitReader is created.
2022-02-17 20:05:56,237 INFO  com.ververica.cdc.connectors.mysql.debezium.task.context.StatefulTaskContext [] - MySQL current GTID set d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61290 does contain the GTID set d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61282 required by the connector.
2022-02-17 20:05:56,247 INFO  com.ververica.cdc.connectors.mysql.debezium.task.context.StatefulTaskContext [] - Server has already purged d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61103 GTIDs
2022-02-17 20:05:56,248 INFO  com.ververica.cdc.connectors.mysql.debezium.task.context.StatefulTaskContext [] - GTID set d667b1cd-778a-11ec-aa60-6c92bf64e18c:61283-61290 known by the server but not processed yet, for replication are available only GTID set d667b1cd-778a-11ec-aa60-6c92bf64e18c:61283-61290
2022-02-17 20:05:56,248 INFO  io.debezium.relational.history.DatabaseHistoryMetrics        [] - Started database history recovery
2022-02-17 20:05:56,249 INFO  io.debezium.relational.history.DatabaseHistoryMetrics        [] - Finished database history recovery of 0 change(s) in 1 ms
2022-02-17 20:05:56,277 INFO  io.debezium.util.Threads                                     [] - Requested thread factory for connector MySqlConnector, id = mysql_binlog_source named = binlog-client
2022-02-17 20:05:56,287 INFO  io.debezium.connector.mysql.MySqlStreamingChangeEventSource  [] - GTID set purged on server: d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61103
2022-02-17 20:05:56,287 INFO  io.debezium.connector.mysql.MySqlStreamingChangeEventSource  [] - Attempting to generate a filtered GTID set
2022-02-17 20:05:56,287 INFO  io.debezium.connector.mysql.MySqlStreamingChangeEventSource  [] - GTID set from previous recorded offset: d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61282
2022-02-17 20:05:56,288 INFO  io.debezium.connector.mysql.MySqlStreamingChangeEventSource  [] - GTID set available on server: d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61290
2022-02-17 20:05:56,288 INFO  io.debezium.connector.mysql.MySqlStreamingChangeEventSource  [] - Using first available positions for new GTID channels
2022-02-17 20:05:56,288 INFO  io.debezium.connector.mysql.MySqlStreamingChangeEventSource  [] - Relevant GTID set available on server: d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61290
2022-02-17 20:05:56,289 INFO  io.debezium.connector.mysql.MySqlStreamingChangeEventSource  [] - Final merged GTID set to use when connecting to MySQL: d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61282
2022-02-17 20:05:56,289 INFO  io.debezium.connector.mysql.MySqlStreamingChangeEventSource  [] - Registering binlog reader with GTID set: d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61282
2022-02-17 20:05:56,289 INFO  io.debezium.connector.mysql.MySqlStreamingChangeEventSource  [] - Skip 0 events on streaming start
2022-02-17 20:05:56,289 INFO  io.debezium.connector.mysql.MySqlStreamingChangeEventSource  [] - Skip 0 rows on streaming start
2022-02-17 20:05:56,290 INFO  io.debezium.util.Threads                                     [] - Creating thread debezium-mysqlconnector-mysql_binlog_source-binlog-client
2022-02-17 20:05:56,293 INFO  io.debezium.util.Threads                                     [] - Creating thread debezium-mysqlconnector-mysql_binlog_source-binlog-client
2022-02-17 20:05:56,302 INFO  io.debezium.connector.mysql.MySqlStreamingChangeEventSource  [] - Connected to MySQL binlog at 10.130.49.141:3306, starting at MySqlOffsetContext [sourceInfoSchema=Schema{io.debezium.connector.mysql.Source:STRUCT}, sourceInfo=SourceInfo [currentGtid=null, currentBinlogFilename=mysql-bin.000005, currentBinlogPosition=31359, currentRowNumber=0, serverId=0, sourceTime=null, threadId=-1, currentQuery=null, tableIds=[], databaseName=null], partition={server=mysql_binlog_source}, snapshotCompleted=false, transactionContext=TransactionContext [currentTransactionId=null, perTableEventCount={}, totalEventCount=0], restartGtidSet=d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61282, currentGtidSet=d667b1cd-778a-11ec-aa60-6c92bf64e18c:1-61282, restartBinlogFilename=mysql-bin.000005, restartBinlogPosition=31359, restartRowsToSkip=0, restartEventsToSkip=0, currentEventLengthInBytes=0, inTransaction=false, transactionId=null]
```

**四、总结**  

* * *

至此，能过三轮验证，确认 flinkcdc 可以支持 mysql 高可用。需要条件是:  
1、mysql 服务开启 gtid_mode  
2、目前官方发布的版本（2.2-SNAPSHOT）存在 bug，不可以直接支持高可用。需要人工合并 bugfix 的代码，自行编译后才可以正常使用。 
 [https://zhuanlan.zhihu.com/p/479832928](https://zhuanlan.zhihu.com/p/479832928)
