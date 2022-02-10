# 通过PyFlink作业处理Kafka数据
本文介绍如何使用阿里云 E-MapReduce 创建的 Hadoop 和 Kafka 集群，运行 PyFlink 作业以消费 Kafka 数据。

## 前提条件

-   已注册阿里云账号，详情请参见[阿里云账号注册流程](https://help.aliyun.com/document_detail/37195.htm#concept-gpr-axx-wdb)。
-   已开通 E-MapReduce 服务。
-   已完成云账号的授权，详情请参见[角色授权](https://help.aliyun.com/document_detail/28072.htm#concept-ykn-bd3-y2b "阿里云 E-MapReduce 服务（例如 Hadoop 和 Spark），在运行时需要有访问其他阿里云资源和执行操作的权限。每个 E-MapReduce 集群必须有服务角色以及 ECS 应用角色。本文为您介绍 EMR 角色授权的流程及其关联的角色。")。
-   已创建项目，详情请参见[项目管理](https://help.aliyun.com/document_detail/85392.htm#concept-rqw-qz2-z2b "创建 E-MapReduce 集群后，您可以在数据开发中创建项目，并在项目中进行作业的编辑和工作流的调度。新建项目之后，您可以对项目进行管理，为项目关联集群资源、添加项目成员以及设置全局变量。")。
-   本地安装了 PuTTY 和文件传输工具（SSH Secure File Transfer Client）。

## 步骤一：创建 Hadoop 集群和 Kafka 集群

创建同一个安全组下的 Hadoop 和 Kafka 集群。创建详情请参见[创建集群](https://help.aliyun.com/document_detail/35223.htm#concept-nrp-154-y2b "本文介绍如何在 E-MapReduce 控制台，通过一键购买方式，快速创建一个按量付费的 Hadoop 集群。")。

1.  登录[阿里云 E-MapReduce 控制台](https://emr.console.aliyun.com/)。
2.  创建 Hadoop 集群，并选择 Flink 服务。

    ![](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/0538030261/p271184.png)
3.  创建 Kafka 集群。

    ![](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/0538030261/p271183.png)

## 步骤二：在 Kafka 集群上创建 Topic

本示例将创建两个分区数为 10、副本数为 2、名称为 payment_msg 和 results 的 Topic。

1.  登录 Kafka 集群的 Master 节点，详情请参见[使用 SSH 连接主节点](https://help.aliyun.com/document_detail/169150.htm#task-2508490 "本文为您介绍如何使用 SSH 方式（SSH 密钥对或 SSH 密码方式）在 Windows 和 Linux 环境中连接主节点。")。
2.  执行如下命令，创建名称为 payment_msg 的 Topic。

    ````null
    /usr/lib/kafka-current/bin/kafka-topics.sh ```

    ````
3.  执行如下命令，创建名称为 results 的 Topic。

    ````null
    /usr/lib/kafka-current/bin/kafka-topics.sh ```

    **说明** 创建Topic后，请保留该登录窗口，后续步骤仍将使用。

    ````

## 步骤三：准备测试数据

在[步骤二](#section-25f-krc-a95)中 Kafka 集群的命令行窗口，执行如下命令，不断生成测试数据。

````null
python3 -m pip install kafka
rm -rf produce_data.py
cat>produce_data.py<<EOF
import random
import time, calendar
from random import randint
from kafka import KafkaProducer
from json import dumps
from time import sleep


def write_data():
    data_cnt = 20000
    order_id = calendar.timegm(time.gmtime())
    max_price = 100000

    topic = "payment_msg"
    producer = KafkaProducer(bootstrap_servers=['emr-worker-1:9092'],
                             value_serializer=lambda x: dumps(x).encode('utf-8'))

    for i in range(data_cnt):
        ts = time.strftime("%Y-%m-%d %H:%M:%S", time.localtime())
        rd = random.random()
        order_id += 1
        pay_amount = max_price * rd
        pay_platform = 0 if random.random() < 0.9 else 1
        province_id = randint(0, 6)
        cur_data = {"createTime": ts, "orderId": order_id, "payAmount": pay_amount, "payPlatform": pay_platform, "provinceId": province_id}
        producer.send(topic, value=cur_data)
        sleep(0.5)


if __name__ == '__main__':
    write_data()

EOF
python3 produce_data.py```

步骤四：创建并运行PyFlink作业
------------------

1.  登录Hadoop集群的Master节点，详情请参见[使用SSH连接主节点](https://help.aliyun.com/document_detail/169150.htm#task-2508490 "本文为您介绍如何使用SSH方式（SSH密钥对或SSH密码方式）在Windows和Linux环境中连接主节点。")。
2.  执行如下命令，生成lib.jar和job.py。
    
    ```null
    rm -rf job.py
    cat>job.py<<EOF
    import os
    from pyflink.datastream import StreamExecutionEnvironment, TimeCharacteristic
    from pyflink.table import StreamTableEnvironment, DataTypes, EnvironmentSettings
    from pyflink.table.udf import udf
    
    
    provinces = ("beijing", "shanghai", "hangzhou", "shenzhen", "jiangxi", "chongqing", "xizang")
    
    
    @udf(input_types=[DataTypes.INT()], result_type=DataTypes.STRING())
    def province_id_to_name(id):
        return provinces[id]
    
    
    def log_processing():
        kafka_servers = "xx.xx.xx.xx:9092,xx.xx.xx.xx:9092,xx.xx.xx.xx:9092"
        kafka_zookeeper_servers = "xx.xx.xx.xx:2181,xx.xx.xx.xx:2181,xx.xx.xx.xx:2181"
        source_topic = "payment_msg"
        sink_topic = "results"
        kafka_consumer_group_id = "test_3"
    
        env = StreamExecutionEnvironment.get_execution_environment()
        env.set_stream_time_characteristic(TimeCharacteristic.EventTime)
        env_settings = EnvironmentSettings.Builder().use_blink_planner().build()
        t_env = StreamTableEnvironment.create(stream_execution_environment=env, environment_settings=env_settings)
        t_env.get_config().get_configuration().set_boolean("python.fn-execution.memory.managed", True)
    
        source_ddl = f"""
                CREATE TABLE payment_msg(
                    createTime VARCHAR,
                    rt as TO_TIMESTAMP(createTime),
                    orderId BIGINT,
                    payAmount DOUBLE,
                    payPlatform INT,
                    provinceId INT,
                    WATERMARK FOR rt as rt - INTERVAL '2' SECOND
                ) WITH (
                  'connector.type' = 'kafka',
                  'connector.version' = 'universal',
                  'connector.topic' = '{source_topic}',
                  'connector.properties.bootstrap.servers' = '{kafka_servers}',
                  'connector.properties.zookeeper.connect' = '{kafka_zookeeper_servers}',
                  'connector.properties.group.id' = '{kafka_consumer_group_id}',
                  'connector.startup-mode' = 'latest-offset',
                  'format.type' = 'json'
                )
                """
    
        es_sink_ddl = f"""
                CREATE TABLE es_sink (
                province VARCHAR,
                pay_amount DOUBLE,
                rowtime TIMESTAMP(3)
                ) with (
                  'connector.type' = 'kafka',
                  'connector.version' = 'universal',
                  'connector.topic' = '{sink_topic}',
                  'connector.properties.bootstrap.servers' = '{kafka_servers}',
                  'connector.properties.zookeeper.connect' = '{kafka_zookeeper_servers}',
                  'connector.properties.group.id' = '{kafka_consumer_group_id}',
                  'connector.startup-mode' = 'latest-offset',
                  'format.type' = 'json'
                )
        """
    
        t_env.sql_update(source_ddl)
        t_env.sql_update(es_sink_ddl)
    
        t_env.register_function('province_id_to_name', province_id_to_name)
    
        query = """
        select province_id_to_name(provinceId) as province, sum(payAmount) as pay_amount, tumble_start(rt, interval '5' second) as rowtime
        from payment_msg
        group by tumble(rt, interval '5' second), provinceId
        """
    
        t_env.sql_query(query).insert_into("es_sink")
    
        t_env.execute("payment_demo")
    
    
    if __name__ == '__main__':
        log_processing()
    EOF
    
    rm -rf lib
    mkdir lib
    cd lib
    wget https://maven.aliyun.com/nexus/content/groups/public/org/apache/flink/flink-sql-connector-kafka_2.11/1.10.1/flink-sql-connector-kafka_2.11-1.10.1.jar
    wget https://maven.aliyun.com/nexus/content/groups/public/org/apache/flink/flink-json/1.10.1/flink-json-1.10.1-sql-jar.jar
    cd ../
    zip -r lib.jar lib/*```
    
    请您根据集群的实际情况，修改job.py中如下参数。
    
      
    | 参数 | 描述 |
    | --- | --- |
    | `kafka_servers` | Kafka集群中Kafka Broker组件的地址列表。IP地址为Kafka集群的内网IP地址，端口号默认为9092。IP地址如[Kafka集群组件](#fig-1)所示。 |
    | `kafka_zookeeper_servers` | Kafka集群中Zookeeper服务的地址列表。IP地址为Kafka集群的内网IP地址，端口号默认为2181。IP地址如[Kafka集群组件](#fig-1)所示。 |
    | `source_topic` | 源表的Kafka Topic，本文示例为payment\_msg。 |
    | `sink_topic` | 结果表的Kafka Topic，本文示例为results。 |
    
    图 1. Kafka集群组件
    
    ![](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/8042598951/p52814.png)
    
    本地以Windows为例，生成lib.jar和job.py示例如下。![](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/8042598951/p148435.png)
    
3.  使用文件传输工具链接Hadoop集群的Master节点，下载生成的lib.jar和job.py至本地。
4.  上传生成的lib.jar和job.py至OSS管理控制台。
    1.  登录[OSS管理控制台](https://oss.console.aliyun.com/)。
    2.  创建存储空间和上传文件，详情请参见[创建存储空间](https://help.aliyun.com/document_detail/31885.htm#task-u3p-3n4-tdb "存储空间（Bucket）是用于存储对象（Object）的容器。在上传任意类型的Object前，您需要先创建Bucket。本文主要介绍如何通过控制台方式创建Bucket。")和[上传文件](https://help.aliyun.com/document_detail/31886.htm#concept-zx1-4p4-tdb "您可以通过OSS控制台上传不超过5 GB大小的文件。")。
        
        本示例的上传路径分别为oss://emr-logs2/test/lib.jar和oss://emr-logs2/test/job.py。
        
5.  创建PyFlink作业。
    1.  登录[阿里云E-MapReduce控制台](https://emr.console.aliyun.com/)。
    2.  在顶部菜单栏处，根据实际情况选择地域和资源组。
    3.  单击上方的数据开发页签。
    4.  在项目列表页面，单击待编辑项目所在行的作业编辑。
    5.  在作业编辑区域，在需要操作的文件夹上，右键选择新建作业。
    6.  输入作业名称、作业描述，在作业类型下拉列表中选择Flink。
    7.  配置作业内容，示例如下。
        
        ```null
        run -m yarn-cluster -py ossref://emr-logs2/test/job.py -j ossref://emr-logs2/test/lib.jar```
        
6.  运行PyFlink作业。
    1.  单击右上角的保存。
    2.  单击右上角的运行。
    3.  在运行作业对话框中，从执行集群列表，选择新建的Hadoop集群。
    4.  单击确定。

步骤五：查看作业信息
----------

1.  通过Yarn UI查看Flink作业的信息。
2.  在Hadoop控制台，单击作业的ID。
    
    您可以查看作业运行详情。![](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/8042598951/p148135.png)
    
    详细信息如下。![](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/8042598951/p148503.png)
    
3.  **可选：** 单击Tracking URL后面的链接，进入Apache Flink Dashboard页面。
    
    您可以查看详情的作业信息。
    

步骤六：查看输出数据
----------

1.  登录Kafka集群的Master节点。详情请参见[使用SSH连接主节点](https://help.aliyun.com/document_detail/169150.htm#task-2508490 "本文为您介绍如何使用SSH方式（SSH密钥对或SSH密码方式）在Windows和Linux环境中连接主节点。")。
2.  执行如下命令，查看results的数据。
    
    ```null
    kafka-console-consumer.sh --bootstrap-server emr-header-1:9092 --topic results```
    
    返回信息如下。![](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/8042598951/p148491.png)
    
    查看完信息后，您可以在数据开发的Flink作业页面，单击右上角的停止，停掉正在运行的Flink作业。 
 [https://help.aliyun.com/document_detail/181568.html](https://help.aliyun.com/document_detail/181568.html)
````
