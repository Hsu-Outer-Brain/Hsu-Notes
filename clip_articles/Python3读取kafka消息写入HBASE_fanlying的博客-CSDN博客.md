# Python3读取kafka消息写入HBASE_fanlying的博客-CSDN博客
kafka 消息格式为 (None,\[json 串])

利用 Python3 有以下 3 种方式将 kafka 消息的往 HBASE 写入

1、直接消费 kafka 消息写入 HBASE：

````null
from kafka import KafkaConsumerhbase_ip='192.168.xxx.xxx'pool = happybase.ConnectionPool(size=3, host=ip)def hbase_load(tableName, lists):with pool.connection() as connection:if tableName not in str(connection.tables()):        create_table(connection, tableName)    table = connection.table(tableName)    b = table.batch(batch_size=1024)                b.put(row=rowkey, data=data_dicts)                print("rowkey:" + rowkey + " data append success")            print(str(ex) + " 插入数据失败")def create_table(conn, table):"ss": dict(max_versions=10)        print(str(ex) + " table exists ！！！")    t = time.strftime(r"%Y-%m-%d_%H-%M-%S", time.localtime())    print("[%s]%s" % (t, str))consumer = KafkaConsumer('logfile', group_id='test-consumer-group', bootstrap_servers=['192.168.xxx.xxx:9092'])    recv = "%s:%d:%d: key=%s value=%s" % (msg.topic, msg.partition, msg.offset, msg.key, msg.value)    dict_data = json.loads(msg.value)    dict_data['info'] = str(dict_data['time'])+'-'+dict_data['pool']    hbase_load('logfile_zf', lst)```

2、使用sparkstreaming的方法直接将RDD往HBASE写：

写入HBASE配置参考：http://dblab.xmu.edu.cn/blog/1715-2/

需要注意：在Spark 2.0版本上缺少相关把hbase的数据转换python可读取的jar包，需要我们另行下载。

打开[spark-examples\_2.11-1.6.0-typesafe-001.jar](https://mvnrepository.com/artifact/org.apache.spark/spark-examples_2.11/1.6.0-typesafe-001)下载jar包

```null
from pyspark import SparkConf, SparkContextfrom pyspark.streaming import StreamingContextfrom pyspark.streaming.kafka import KafkaUtilsconf = SparkConf().setAppName("logSparkStreaming")sc = SparkContext(conf=conf)ssc = StreamingContext(sc, 5)table = 'logfile_stream2'broker = "192.168.xxx.xxx:9092"hbaseZK = "192.168.xxx.xxx"keyConv = "org.apache.spark.examples.pythonconverters.StringToImmutableBytesWritableConverter"valueConv = "org.apache.spark.examples.pythonconverters.StringListToPutConverter"hbaseConf = {"hbase.zookeeper.quorum": hbaseZK, "hbase.mapred.outputtable": table,"mapreduce.outputformat.class": "org.apache.hadoop.hbase.mapreduce.TableOutputFormat","mapreduce.job.output.key.class": "org.apache.hadoop.hbase.io.ImmutableBytesWritable","mapreduce.job.output.value.class": "org.apache.hadoop.io.Writable"}    t = time.strftime(r"%Y-%m-%d %H:%M:%S", time.localtime())    print("[%s]%s" % (t, str))        msg_dict['info'] = str(msg_dict['time'])+'-'+msg_dict['pool']        rowkey = msg_dict['info']for d, x in msg_dict.items():            msg_tuple = (rowkey, [rowkey, col_family, col_name, col_value])            print("rowkey:" + rowkey + "\ndata " + str(msg_tuple) + " append success")def connectAndWrite(data):        msg_list = data.map(lambda x: json.loads(x[1]))            msg_row = msg_list.map(lambda x: fmt_data(x))            msg_row.flatMap(lambda x: x).map(lambda x: x).saveAsNewAPIHadoopDataset(conf=hbaseConf, keyConverter=keyConv,valueConverter=valueConv)            print(str(ex) + " 插入数据失败")kafkaStreams = KafkaUtils.createDirectStream(ssc, [topic], kafkaParams={"metadata.broker.list": broker})kafkaStreams.foreachRDD(connectAndWrite)```

```groovy
$SPARK_HOME/bin/spark-submit --master local --packages org.apache.spark:spark-streaming-kafka_2.11:1.6.0  --jars spark-examples_2.11-1.6.0-typesafe-001.jar /home/user/spark/sparkstreaming_kafka2.py > /home/user/spark/sparkstreaming_kafka.log
````

注：spark-examples_2.11-1.6.0-typesafe-001.jar 为把 hbase 的数据转换 python 可读取的 jar 包

3、读出 sparkstreaming 的 RDD 数据往 HBASE 写：

````null
from pyspark import SparkConf, SparkContextfrom pyspark.streaming import StreamingContextfrom pyspark.streaming.kafka import KafkaUtilsfrom pyspark.sql import SQLContexthbase_ip='192.168.xxx.xxx'pool = happybase.ConnectionPool(size=3, host=ip)def create_table(conn, table):"ss": dict(max_versions=10)        print(str(ex) + " table exists ！！！")    t = time.strftime(r"%Y-%m-%d_%H-%M-%S", time.localtime())    print("[%s]%s" % (t, str))with pool.connection() as connection:if table not in str(connection.tables()):         create_table(connection, table)    hbaseTable = connection.table(table)    b = hbaseTable.batch(batch_size=1024)        msg_rdd = msg.map(lambda x: json.loads(x[1]))        msg_list = msg_rdd.collect()for msg_dict in msg_list:            msg_dict['info'] = str(msg_dict['time'])+'-'+msg_dict['pool']                rowkey = msg_dict['info']for d, x in msg_dict.items():                b.put(row=rowkey, data=data_dict)                print("rowkey:" + rowkey + "\ndata " + str(data_dict) + " append success")                print(str(ex) + " 插入数据失败")conf = SparkConf().setAppName("logSparkStreaming")sc = SparkContext(conf=conf)ssc = StreamingContext(sc, 2)broker = "192.168.xxx.xxx:9092"kafkaStreams = KafkaUtils.createDirectStream(ssc, [topic], kafkaParams={"metadata.broker.list": broker})kafkaStreams.foreachRDD(writeHbase)```

  

```groovy
$SPARK_HOME/bin/spark-submit --master local[3] --packages org.apache.spark:spark-streaming-kafka_2.11:1.6.0  /home/user/spark/sparkstreaming_kafka.py > /home/user/spark/sparkstreaming_kafka.log
````

 [https://blog.csdn.net/fanlying/article/details/80284915](https://blog.csdn.net/fanlying/article/details/80284915)
