# hive与hbase集成 - 超人学院 - OSCHINA - 中文开源技术交流社区
详细步骤

一、简介

Hive

是基于 Hadoop 的一个数据仓库工具，可以将结构化的数据文件映射为一张数据库表，并提供完整的 sql 查询功能，可以将 sql 语句转换为 MapReduce 任务进行运行。其优点是学习成本低，可以通过类 SQL 语句快速实现简单的 MapReduce 统计，不必开发专门的 MapReduce 应用，十分适合数据仓库的统计分析。  
Hive 与 HBase 的整合功能的实现是利用两者本身对外的 API 接口互相进行通信，相互通信主要是依靠 hive_hbase-handler.jar 工具类。二、安装步骤：

1 .Hadoop

和 Hbase 都已经成功安装了

Hadoop

集群配置：[http://www.linuxidc.com/Linux/2012-02/53632.htm](http://www.linuxidc.com/Linux/2012-02/53632.htm)  
Hbase 安装配置：[http://www.linuxidc.com/Linux/2012-02/54225.htm](http://www.linuxidc.com/Linux/2012-02/54225.htm)

2 . 

拷贝 hbase-0.90.3-cdh3u1.jar 和 zookeeper-3.3.3-cdh3u1.jar 到 hive/lib 下。注意：如何 hive/lib 下已经存在这两个文件的其他版本（例如 zookeeper-3.3.2.jar），建议删除后使用 hbase 下的相关版本。

2. 

修改 hive/conf 下 hive-site.xml 文件，在底部添加如下内容：
<property>  
<name>hive.exec.scratchdir</name>  
<value>/tmp</value>  
<description>Scratch space for Hive jobs</description>  
</property>

<property>  
<name>hive.querylog.location</name>  
<value>/usr/local/hive/logs</value>  
</property>

<property>  
<name>hive.aux.jars.path</name>  
<value>file:///usr/local/hive/lib/hive-hbase-handler-0.7.1-cdh3u1.jar,file:///usr/local/hive/lib/hbase-0.90.3-cdh3u1.jar,fi  
le:///usr/local/hive/lib/zookeeper-3.3.1.jar</value>  
</property>

注意：如果 hive-site.xml 不存在则自行创建，或者把 hive-default.xml.template 文件改名后使用。

3. 

拷贝 hbase-0.90.3-cdh3u1.jar 到所有 hadoop 节点 (包括 master) 的 hadoop/lib 下。

4. 

拷贝 hbase/conf 下的 hbase-site.xml 文件到所有 hadoop 节点 (包括 master) 的 hadoop/conf 下。注意，如果 3,4 两步跳过的话，运行 hive 时很可能出现如下错误：

view plaincopy   
org.apache.hadoop.hbase.ZooKeeperConnectionException: HBase is able to connectto ZooKeeper but the connection closes immediately.   
This could be a sign that the server has too many connections (30 is thedefault). Consider inspecting your ZK server logs for that error and   
then make sure you are reusing HBaseConfiguration as often as you can. SeeHTable's javadoc for more information. at org.apache.hadoop.   
hbase.zookeeper.ZooKeeperWatcher. 

三、启动 Hive

1.

单节点启动

\#bin/hive -hiveconf hbase.dwn01=master:490001

2 

集群启动：

\#bin/hive -hiveconf hbase.zookeeper.quorum=dwn01,dwd01,dwd02,dwd03

如何 hive-site.xml 文件中没有配置 hive.aux.jars.path，则可以按照如下方式启动。

bin/hive --auxpath /usr/local/hive/lib/hive-hbase-handler-0.8.0.jar,/usr/local/hive/lib/hbase-0.90.5.jar, /usr/local/hive/lib/zookeeper-3.3.2.jar-hiveconf hbase.zookeeper.quorum=dwn01,dwd01,dwd02,dwd03

四、测试:

1.

创建 hbase 识别的数据库：

CREATE TABLE hbase_table_1(key int, value string)   
STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'   
WITH SERDEPROPERTIES ("hbase.columns.mapping" =":key,cf1:val")   
TBLPROPERTIES ("hbase.table.name" = "xyz");   
hbase.table.name 

定义在 hbase 的 table 名称

hbase.columns.mapping 

定义在 hbase 的列族   
2. 使用 sql 导入数据

1) 

新建 hive 的数据表:

CREATE TABLE hb_test (id INT, url STRING);  
2)

批量插入数据:

hive> LOAD DATA LOCAL INPATH '/tmp/id.txt' OVERWRITE INTO TABLE hb_test

3)

使用 sql 导入 hbase_table_1:

hive> INSERT OVERWRITE TABLE hbase_table_1 SELECT \* FROM hb_test;

3. 

查看数据

hive> select \* from hbase_table_1; 

这时可以登录 Hbase 去查看数据了  
#bin/hbase shell  
hbase(main):001:0> describe 'xyz'   
hbase(main):002:0> scan 'xyz'   
hbase(main):003:0> put 'xyz','100','cf1:val','www.51.com'这时在 Hive 中可以看到刚才在 Hbase 中插入的数据了。

4 hive

访问已经存在的 hbase  

使用 CREATE EXTERNAL TABLE:

CREATE EXTERNAL TABLE hbase_table_2(key int, value string)   
STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'   
WITH SERDEPROPERTIES ("hbase.columns.mapping" = "cf1:val")   
TBLPROPERTIES("hbase.table.name" = "some_existing_table");   
内容参考：[http://wiki.apache.org/hadoop/Hive/HBaseIntegration](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fwiki.apache.org%2Fhadoop%2FHive%2FHBaseIntegration)

 [https://my.oschina.net/crxy/blog/423967](https://my.oschina.net/crxy/blog/423967)

 [https://my.oschina.net/crxy/blog/423967](https://my.oschina.net/crxy/blog/423967)
