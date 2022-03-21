# (11条消息) Flink学习2-安装、启动和配置_迷路剑客的博客-CSDN博客_flink启动
-   更多 Flink 系列文章请点击[Flink 系列文章](https://blog.csdn.net/baichoufei90/article/details/105674793)
-   更多大数据文章请点击[大数据好文推荐](https://blog.csdn.net/baichoufei90/article/details/90264272)

本篇文章主要讲解 Flink 下载、安装和启动的步骤。

关于下载的更多信息可参考[Flink 官网](https://flink.apache.org/downloads.html)

如果是用的 MacOS X，可以直接用 Homebrew 安装：

```shell
brew install apache-flink

```

当前最新稳定的版本是 v1.6.1。Flink 可以不依赖 Hadoop，但我们环境中要把 Flink 跑在 Yarn 上，所以需要下载 Flink With Hadoop 的版本的 tgz 包：

-   [Flink with Hadoop® 2.7-binary](https://www.apache.org/dyn/closer.lua/flink/flink-1.6.1/flink-1.6.1-bin-hadoop27-scala_2.11.tgz)
-   [Flink with Hadoop® 2.6-binary](https://www.apache.org/dyn/closer.lua/flink/flink-1.6.1/flink-1.6.1-bin-hadoop26-scala_2.11.tgz)
-   [Flink 1.6.1-source](https://www.apache.org/dyn/closer.lua/flink/flink-1.6.1/flink-1.6.1-src.tgz)

## 2.1 安装

只需直接解压即可

```shell
 $ tar -zxvf flink-1.6.1-bin-hadoop27-scala_2.11.tgz
 $ cd flink-1.6.1
 $ bin/flink --version
 Version: 1.6.1, Commit ID: 23e2636

```

懒人可以设置一个 PATH，以便以后在任意路径可以直接使用 flink 命令:

```shell
$ vim ~/.bash_profile

PATH="/Users/chengc/cc/apps/flink-1.6.1/bin:${PATH}"
export PATH

```

保存后可以试试看:

```shell
$ flink -v
Version: 1.6.1, Commit ID: 23e2636

```

## 2.2 配置

可参考[Flink-Configuration](https://ci.apache.org/projects/flink/flink-docs-master/ops/config.html)

配置项应写入`$FLINK_HOME/conf/flink-conf.yaml`文件或启动命令中使用类似`-yD env.java.opts.jobmanager="-XX:+PrintGCDetails"`方式配置

### 2.2.1 启动配置

#### 2.2.2.1 Job 并行度

-   taskmanager.numberOfTaskSlots  
    指定一个 TaskManager 的 Slot 个数，默认为 1。该值推荐设为 TM CPU 核心数或一半。

    每个 slot 可以运行一个 task 或 pipeline。该选项不同设置分析：

    -   该选项设置大于 1 时，可使得 TM 上的 slot 们共享 JVM、应用库、网络等资源，利用多 CPU 核心，提高资源利用率，但内存会被分隔为 slot 数量份。
    -   而如果设置为 1，则提供了最高的 task-slot 资源隔离级别。
-   parallelism.default  
    当没有指定`parallelism`时的默认并行度，默认值为 1。

Flink on YARN 时，TaskManagers 个数 = `Job 中的最大并行度 / 每个 TaskManager 分配的 Slot 数`。

也就是说 TM 数量由 Flink 根据你的 Job 情况自动推算，`-yn`启动参数失效了。

一个 Flink Job 生成 T 执行计划时关于分为多个 Task 或 Pipeline，Task 可以由多线程并发执行，每个线程处理 Task 输入数据的一个子集，并发线程数就是`Parallelism`（并行度）。

我们可以有多种方法设置并行度，优先级从高到低为：

1.  算子级别 \`\`\`java
    val text: DataStream[String] = env.socketTextStream(hostname, port, '\\n')
    text.flatMap { w => w.split("\\s") }.setParallelism(2)

    ```

    ```
2.  ExecutionEnvironment 级别 \`\`\`java
    val bsEnv = StreamExecutionEnvironment.getExecutionEnvironment
        bsEnv.setParallelism(1)

    ```

    ```
3.  提交时命令行指定  
    将 job 并行度设为 40\`\`\`shell
    bin/flink run -m yarn-cluster ... -p 40 ...

    ```

    ```
4.  配置文件 (`flink-conf.yaml`) 级别  
    parallelism.default: 4

### 2.2.2 内存配置

参考：

-   [Flink 学习 1 - 基础概念 - 内存](https://blog.csdn.net/baichoufei90/article/details/82875496#memory-config)

可参考[Standalone Cluster](https://ci.apache.org/projects/flink/flink-docs-master/ops/deployment/cluster_setup.html)

## 3.1 Flink 集群的启动

通过简单命令就能在本地启动一个 Flink 集群

```shell
$ ./bin/start-cluster.sh 
Starting cluster.
Starting standalonesession daemon on host chengcdeMacBook-Pro.local.
Starting taskexecutor daemon on host chengcdeMacBook-Pro.local.

```

看到以上信息代表 Flink 启动成功，我们可以通过 jps 来看看启动了哪些进程：

```shell
$ jps
70673 TaskManagerRunner
70261 StandaloneSessionClusterEntrypoint
70678 Jps
69647 Launcher
69646 NailgunRunner

```

可以看到分别启动了好几个 Flink 的重要组件，如果你看了第一章应该了解他们的作用。

## 3.2 Flink 监控页面

我们可以通过访问[http://localhost:8081](http://localhost:8081/)看看效果:  
![](https://img-blog.csdn.net/20180928094410824?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

可以从 flink 的 web 界面上看到现在运行了一个 Task Manager 实例。

## 3.3 Flink 集群日志

还可以通过查看日志看到 flink 服务器正常启动：

```shell
tail -100f log/flink-*-standalonesession-*.log

```

## 3.4 Flink 集群的停止

通过简单命令就能停止 Flink 集群：

```shell
$ ./bin/stop-cluster.sh

```

## 4.1 Maven

### 4.1.1 骨架构建

Scala 版本官网构建初始 Flink 工程例子：[Project Template for Scala](https://ci.apache.org/projects/flink/flink-docs-release-1.9/dev/projectsetup/scala_api_quickstart.html)

[DataStream API Tutorial](https://ci.apache.org/projects/flink/flink-docs-release-1.9/getting-started/tutorials/datastream_api.html#setting-up-a-maven-project)

构建过程如遇问题可参考[Flink Maven archetype:generate](https://blog.csdn.net/baichoufei90/article/details/103600544)

用 Maven 骨架构建 Flink 初始工程需要使用`Maven 3.0.4+`以及`Java 8.x`，然后有以下两种方式构建初始工程：

-   mvn archetype:generate 方式 \`\`\`java
    mvn archetype:generate                               \\
          \-DarchetypeGroupId=org.apache.flink              \\
          \-DarchetypeArtifactId=flink-quickstart-scala     \\
          \-DarchetypeVersion=1.9.0

    ```

    此方式或让你输入工程的groupId、artifactId、version等信息，最后得到构建好的初始工程，名字就是输入的artifactId。
    ```
-   调用 quickstart 远程脚本 \`\`\`bash
    $ curl [https://flink.apache.org/q/quickstart-scala.sh](https://flink.apache.org/q/quickstart-scala.sh) | bash -s 1.9.0

    ```

    此方法得到的工程名为`quickstart`。
    ```

以上两个方法得到的工程初始文件情况类似如下：

```bash
$ tree chengc-flink-demo/
chengc-flink-demo/
├── pom.xml
└── src
    └── main
        ├── resources
        │   └── log4j.properties
        └── scala
            └── com
                └── chengc
                    └── flink
                        ├── BatchJob.scala
                        └── StreamingJob.scala

```

BatchJob 和 StreamingJob 分别表示 DataSet 批处理、DataStream 流处理项目，他们各自的 main 方法里有一些基本方法和注释。

以上步骤执行完后，可以使用`mvn clean package`打包。如果要改变打出的包的 main-class，请修改 pom 中 maven-shade-plugin 的`<mainClass>`。

### 4.1.2 Maven

以下的依赖分为 Java 版和 Scala 版。这些依赖包括 Flink 本地运行环境所以可以在本地运行调试我们的 Flink 代码。

#### 4.1.2.1 For Java

```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-java</artifactId>
  <version>1.6.1</version>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-java_2.11</artifactId>
  <version>1.6.1</version>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-clients_2.11</artifactId>
  <version>1.6.1</version>
</dependency>

```

#### 4.1.2.2 For Scala

```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-scala_2.11</artifactId>
  <version>1.6.1</version>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-scala_2.11</artifactId>
  <version>1.6.1</version>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-clients_2.11</artifactId>
  <version>1.6.1</version>
</dependency>

```

## 4.2 Code

### 4.2.1 Java

```java
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.common.functions.ReduceFunction;
import org.apache.flink.api.java.utils.ParameterTool;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.util.Collector;


@SuppressWarnings("serial")
public class SocketWindowWordCount {

	public static void main(String[] args) throws Exception {

		
		final String hostname;
		final int port;
		try {
			final ParameterTool params = ParameterTool.fromArgs(args);
			hostname = params.has("hostname") ? params.get("hostname") : "localhost";
			port = params.getInt("port");
		} catch (Exception e) {
			System.err.println("No port specified. Please run 'SocketWindowWordCount " +
				"--hostname <hostname> --port <port>', where hostname (localhost by default) " +
				"and port is the address of the text server");
			System.err.println("To start a simple text server, run 'netcat -l <port>' and " +
				"type the input text into the command line");
			return;
		}

		
		final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

		
		DataStream<String> text = env.socketTextStream(hostname, port, "\n");

		
		DataStream<WordWithCount> windowCounts = text

				.flatMap(new FlatMapFunction<String, WordWithCount>() {
					@Override
					public void flatMap(String value, Collector<WordWithCount> out) {
						for (String word : value.split("\\s")) {
							out.collect(new WordWithCount(word, 1L));
						}
					}
				})

				.keyBy("word")
				.timeWindow(Time.seconds(5))

				.reduce(new ReduceFunction<WordWithCount>() {
					@Override
					public WordWithCount reduce(WordWithCount a, WordWithCount b) {
						return new WordWithCount(a.word, a.count + b.count);
					}
				});

		
		windowCounts.print().setParallelism(1);

		env.execute("Socket Window WordCount");
	}

	

	
	public static class WordWithCount {

		public String word;
		public long count;

		public WordWithCount() {}

		public WordWithCount(String word, long count) {
			this.word = word;
			this.count = count;
		}

		@Override
		public String toString() {
			return word + " : " + count;
		}
	}
}

```

### 4.2.2 scala

```java
import org.apache.flink.api.java.utils.ParameterTool
import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.api.windowing.time.Time


object SocketWindowWordCount {

  
  def main(args: Array[String]) : Unit = {

    
    var hostname: String = "localhost"
    var port: Int = 0

    try {
      val params = ParameterTool.fromArgs(args)
      hostname = if (params.has("hostname")) params.get("hostname") else "localhost"
      port = params.getInt("port")
    } catch {
      case e: Exception => {
        System.err.println("No port specified. Please run 'SocketWindowWordCount " +
          "--hostname <hostname> --port <port>', where hostname (localhost by default) and port " +
          "is the address of the text server")
        System.err.println("To start a simple text server, run 'netcat -l <port>' " +
          "and type the input text into the command line")
        return
      }
    }
    
    
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    
    
    val text: DataStream[String] = env.socketTextStream(hostname, port, '\n')

    
    val windowCounts = text
          .flatMap { w => w.split("\\s") }
          .map { w => WordWithCount(w, 1) }
          .keyBy("word")
          .timeWindow(Time.seconds(5))
          .sum("count")

    
    windowCounts.print().setParallelism(1)

    env.execute("Socket Window WordCount")
  }

  
  case class WordWithCount(word: String, count: Long)
}

```

## 4.3 打包

将文件打包为 jar

```shell
flink-demo-1.0-SNAPSHOT-jar-with-dependencies.jar

```

## 4.4 启动示例程序

以上代码所写的程序功能是从`socket`中读取文本，然后每隔 5 秒打印出每个单词在当前时间往前推 5 秒的时间窗口内的出现次数。

### 4.4.1 启动 netcat

在`9999`端口启动本地 netcat 服务：

```shell
$ nc -l 9999

```

### 4.4.2 提交 Flink 应用

```shell
$ flink run /Users/chengc/cc/work/projects/flink-demo/target/SocketWindowWordCount-jar-with-dependencies.jar --port 9999

Starting execution of program

```

现在我们看看前面提到过的 flink web 界面：  
![](https://img-blog.csdn.net/20180928143010881?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

点击这行 job 信息能看到 job 详情页：  
![](https://img-blog.csdn.net/20180928152232860?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 4.4.3 测试 Flink 应用

通过以上步骤我们建立了 Flink 应用和 9999 端口的关系，现在我们试试再`nc`界面输入一些字符串:

```shell
$ nc -lk 9999
i am a chinese
who are you
how do you do
how do you do

```

与此同时，我们使用`tailf` 查看 flink 应用的输出：

```shell
$  tail -f log/flink-*-taskexecutor-*.out
i : 1
chine : 1
a : 1
am : 1
who : 1
you : 1
are : 1
how : 2
you : 2
do : 4

```

可以看到 ，示例程序以翻滚窗口（tumbling window）的形式每隔 5 秒将前 5 秒的数据进行了字符统计。

先决条件：

-   Hadoop 2.4+

可参考:

-   [YARN Setup](https://ci.apache.org/projects/flink/flink-docs-master/ops/deployment/yarn_setup.html)
-   [一文精通 Flink on YARN](http://itindex.net/detail/59299-flink-on-yarn)
-   [Flink on yarn 部署实例](https://www.jianshu.com/p/1b05202c4fb6)
-   [Flink Maven archetype:generate](https://blog.csdn.net/baichoufei90/article/details/103600544)

## 5.1 配置 HADOOP/YARN 环境变量

提交 Flink 任务时，所在机器必须要至少设置环境变量`YARN_CONF_DIR`、`HADOOP_CONF_DIR`、`HADOOP_CONF_PATH`中的一个，才能读取 YARN 和 HDFS 的配置信息（会按三者从左到右的顺序读取，只要发现一个就开始读取。如果没有正确设置，会尝试使用`HADOOP_HOME/etc/hadoop`），否则提交任务会失败。

## 5.2 关于执行脚本参数

### 5.2.1 $FLINK_HOME/bin/yarn-session.sh

`yarn-session.sh`主要是用来在 Yarn 上启动常驻 Flink 集群。

参数说明如下：

```shell
必填  
     -n,--container <arg>            分配多少个yarn container (=TaskManager的数量)  
可选  
     -D <property=value>             设置属性值  
     -d,--detached                   Start detached，类似于spark的yarn-cluster模式，既不依赖于本地Yarn Session启动进程。
     -h,--help                       yarn session帮助
     -id,--applicationId <arg>       依附到某个指定yarn session来提交job
     -j,--jar <arg>                  flink jar路径
     -jm,--jobManagerMemory <arg>    JobManager的Container内存 [默认是MB]  
     -m,--jobmanager <arg>           当需要连接不同于configuration中的那个JobManager(master)时使用该参数指定。
     -nl,--nodeLabel <arg>           当需要flink job仅运行在基于标签的Yarn NM node上时使用该参数
     -nm,--name                      设置该job在yarn上运行时的application name
     -q,--query                      显示yarn中可用的资源 (内存, cpu)  
     -qu,--queue <arg>               指定要提交到的Yarn队列.
     -s,--slots <arg>                每个TaskManager使用的slot数。推荐设为每个机器的CPU数。
     -t,--ship <arg>                 上传指定的相对路径的本地文件到Flink集群，程序中读取文件也使用相对路径
     -tm,--taskManagerMemory <arg>   每个TaskManager的Container内存 [默认是MB]  
     -yd,--yarndetached              If present, runs the job in detached mode (deprecated; use non-YARN specific option instead)
     -z,--zookeeperNamespace <arg>   针对HA模式时，在zookeeper上用来创建子路径的NameSpace 

```

### 5.2.2 $FLINK_HOME/bin/flink

#### 5.2.2.1 概述

可参考:

-   [Command-Line Interface](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/cli.html)

该方法可加很多 Action 参数：

-   run  
    用来编译、运行程序
-   info  
    展示优化后 JSON 格式的程序执行计划
-   list  
    展示正在运行的和调度的程序
-   stop  
    停止一个正在运行的流式程序
-   cancel  
    撤销一个正在运行的程序
-   savepoint  
    触发正在运行的作业的 savepoint 或处置现有的 savepoint。
-   modify  
    修改一个运行中的任务参数，如改变并行度等

#### 5.2.2.2 run

```shell
-c,--class <classname>               当jar包内没有用manifest指定运行的class时，需要使用该选项来指定主类（既包括`main`或`getPlan`方法的类）
-C,--classpath <url>                 Adds a URL to each user code
                                     classloader  on all nodes in the
                                     cluster. The paths must specify a
                                     protocol (e.g. file://) and be
                                     accessible on all nodes (e.g. by means
                                     of a NFS share). You can use this
                                     option multiple times for specifying
                                     more than one URL. The protocol must
                                     be supported by the {@link
                                     java.net.URLClassLoader}.
-d,--detached                        使用detached mode运行job，此时client会将flink job体较到固定的Flink集群运行。这种模式不能使用flink命令停止yarn-session，需要使用yarn application -kill <appId>
-n,--allowNonRestoredState           允许跳过无法还原的保存点状态。 
                                     如果在触发保存点时从程序中删除了一部分运算符，则需要允许此操作。
-p,--parallelism <parallelism>       运行job的并行度，配置了就会覆盖configuration内的默认值(parallelism.default: 1)。
-q,--sysoutLogging                   If present, suppress logging output to
                                     standard out.
-s,--fromSavepoint <savepointPath>   保存点恢复job来源路径
                                     (如 hdfs:///flink/savepoint-1537)

yarn-cluster mode可选项:
-d,--detached                        使用detached mode运行job，此时client会将flink job体较到固定的Flink集群运行。
									 这种模式不能使用flink命令停止yarn-session，需要使用yarn application -kill <appId>
-m,--jobmanager <arg>                当需要连接不同于configuration中的那个JobManager(master)时使用该参数指定。
-yD <property=value>                 设置属性值  
-yd,--yarndetached                   (deprecated; use non-YARN
                                     specific option instead)If present, runs the job in detached
                                     mode 
-yh,--yarnhelp                       Help for the Yarn session CLI.
-yid,--yarnapplicationId <arg>       Attach to running YARN session
-yj,--yarnjar <arg>                  Path to Flink jar file
-yjm,--yarnjobManagerMemory <arg>    Memory for JobManager Container with
                                     optional unit (default: MB)
-yn,--yarncontainer <arg>            分配多少个yarn container (=TaskManager的数量)  
-ynl,--yarnnodeLabel <arg>           Specify YARN node label for the YARN
                                     application
-ynm,--yarnname <arg>                Set a custom name for the application
                                     on YARN
-yq,--yarnquery                      Display available YARN resources
                                     (memory, cores)
-yqu,--yarnqueue <arg>               Specify YARN queue.
-ys,--yarnslots <arg>                Number of slots per TaskManager
-yst,--yarnstreaming                 Start Flink in streaming mode
-yt,--yarnship <arg>                 Ship files in the specified directory
                                     (t for transfer)
-ytm,--yarntaskManagerMemory <arg>   Memory per TaskManager Container with
                                     optional unit (default: MB)
-yz,--yarnzookeeperNamespace <arg>   Namespace to create the Zookeeper
                                     sub-paths for high availability mode
-z,--zookeeperNamespace <arg>        Namespace to create the Zookeeper
                                     sub-paths for high availability mode

default mode可选项:
-m,--jobmanager <arg>           Address of the JobManager (master) to which
                                to connect. Use this flag to connect to a
                                different JobManager than the one specified
                                in the configuration.
-z,--zookeeperNamespace <arg>   Namespace to create the Zookeeper sub-paths
                                for high availability mode

```

## 5.2 提交 Flink Job 的三种方式

![](https://img-blog.csdnimg.cn/20191101105057244.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

### 5.2.1 default mode-Yarn Session

#### 5.2.1.1 概述

![](https://img-blog.csdnimg.cn/20200914102311807.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

对应上面图例的右上。

此方式需要首先在 Yarn 中初始化一个持续运行的 Flink Session 集群（就是一个 Yarn Application），并得到一个 application_id，此时总的 Container 资源就确定了，也就是说所有后续提交到该 Session 集群的 Job 共享 JM 和 TM。Client Develoyer 就可通过命令行、RestAPI、WebUI 等方式使用该 id 进行后续所有 job 提交运行（上传所需依赖的 Dependencies jar 包、生成 JobGraph，然后提交到 JM），不再需要每次都与 Yarn 交互了。

而且此模式下 JM 的生命周期不受提交的 Job 影响，会长期运行。

FlinkSession 模式支持 Native 模式部署，即 TM 可动态申请。

Yarn-Session 方式又包括两种方式：

-   普通模式，类似 Spark 中的 Yarn-Client 模式  
    如果使用 Yarn Session Cluster 模式，则启动后集群的运行依赖于本地 Yarn Session 本地启动进程，一旦该进程关闭则整个 Session 集群也会终止。

    改模式启动后，可在 Yarn 上看到该 Flink Session 生成的 Application，并可点击对应 URL 进入 Flink Web UI。
-   Detached - 分离模式，类似 Spark 中的 Yarn-Cluster 模式  
    指定参数`-d`或`--detached`改为依附模式，即将 Flink Session 集群运行交给 Yarn 管理，不再依赖本地启动进程。

    这种模式不能使用 flink 命令停止 yarn-session，需要使用`yarn application -kill <appId>`来停止该 Flink Session 集群。

#### 5.2.1.2 Detached 模式例子

1.  在 Yarn 上启动 detached 模式的常驻 Flink 集群，记录 application_id。  
    启动一个 2 个 TaskManager 的 Flink 集群，为每个 TaskManager 的 Container 分配 1GB 内存，为 JobManager 分配 1GB 内存。\`\`\`shell
    $FLINK_HOME/bin/yarn-session.sh -n 2 -tm 1024 -jm 1024 -d

    ```

    ```
2.  detach 到该 Flink Session 集群 \`\`\`shell
    $FLINK_HOME/bin/yarn-session.sh -id application_id

    ```

    ```
3.  提交 Job 到该 Flink Session 集群执行 \`\`\`shell
    $FLINK_HOME/bin/flink run ./examples/streaming/WordCount.jar --input hdfs:///..../LICENSE-2.0.txt --output hdfs:///.../wordcount-result.txt

    ```

    ```
4.  通过 Flink 集群 WebUI 查看 Job
5.  可通过 WebUI 或 stop/cancel 停止 Job

#### 5.2.1.3 小结

-   优点
    -   资源固定
    -   资源共享，利用率高
-   缺点
    -   资源隔离较差  
        集群资源固定，如果前面的 Job 将资源占满，那么后续的 Job 可能会因为无足够资源只能等待。线上环境一般不用该模式。
    -   不易扩展，计算资源伸缩性较差

### 5.2.2 yarn-cluster Per-Job mode

#### 5.2.2.1 概述

![](https://img-blog.csdnimg.cn/2020091410275797.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70#pic_center)

对应上面图例的右下。

每个 Job 一个 Yarn-Session Cluster，每次都单独申请资源，资源数量根据客户端提交 Job 时指定。客户端依然要负责生成和提交 JobGraph 和上传依赖 jar。

此模式下 JM 的声明周期和 Job 相同。

#### 5.2.2.2 例子

该模式需要使用`$FLINK_HOME/bin/flink`，可以使用`$FLINK_HOME/bin/yarn-session.sh`的参数，只不过参数名前加了`y`或`yarn`前缀。

此模式 Job 资源受 Yarn 分配队列的总资源限制。我们在线上生成环境使用的该模式。

1.  提交任务  
    yarn-cluster 模式，设置 job 并行为 4，JobManager 内存 1GB，每个 TaskManager 内存 4GB\`\`\`shell
    $FLINK_HOME/bin/flink run -m yarn-cluster -p 4 -yjm 1024m -ytm 4096m ./examples/batch/WordCount.jar

    ```

    ```
2.  该模式每提交一个 Job 就有一个 Yarn-Session 集群，也对应了一个 application_id

#### 5.2.2.2 小结

优点：

-   资源隔离程度最高
-   资源申请数由客户端决定  
    缺点：
-   资源利用率不高，需要用户仔细评估

### 5.2.3 Application Mode

#### 5.2.3.1 Session 和 Per-Job 问题

-   每次都需要上传用户 Jar
-   每次都需要在用户客户端生成 JobGraph
-   任务多时客户端压力大，因为每个任务是阻塞时提交，后续 Job 需要排队

#### 5.2.3.2 概述

Flink 1.11 中加入的新模式，公共 Jar 包提前放置到 HDFS，然后 Client 将运行作业需要的依赖都通过`Yarn Local Resource`传递到 JM。

最重要的一点是 `JobGraph` 的生成以及 Job 的提交不再会在 Client 执行，而是转移到 JM 执行（也就是说 Application 的`main()`方法从 Client 转到 JM 执行），客户端不需要上传 Jar 了而只需要负责 Job 提交和管理，这样网络下载上传的负载也会分散到集群中，不再有 Client 单点瓶颈的问题。

注意，此模式下每个 Application 对应一个 JM 但，可以包含多个 Job。

#### 5.2.3.3 例子

可通过`-Dyarn.provided.lib.dirs`指定依赖的包路径，就不用上传了。如果不指定会每次都上传依赖包。当然，这样提交后还是需要上传用户 Jar，如下例的`WordCount.jar`

```bash
$FLINK_HOME/bin/flink run-application -t yarn-application \
-Djobmanager.memory.process.size=1024m \
-Dtaskmanager.memory.process.size=2048m \
-Dyarn.provided.lib.dirs="hdfs://xxx/flink" \
./WordCount.jar

```

当然，也可以把用户 Jar 也指定为 HDFS 上的 Jar：

```bash
$FLINK_HOME/bin/flink run-application -t yarn-application \
-Djobmanager.memory.process.size=1024m \
-Dtaskmanager.memory.process.size=2048m \
-Dyarn.provided.lib.dirs="hdfs://xxx/flink" \
hdfs://xxx/flink-user/WordCount.jar

```

#### 5.2.3.4 小结

优点：

-   降低带宽和客户端负载
-   Application 之间资源隔离
-   Application 内资源共享  
    缺点：
-   生产案例较少
-   仅支持 Yarn、K8S

## 5.3 Yarn 容错配置

详情请参考[Flink 学习 1 - 基础概念 - HA](https://blog.csdn.net/baichoufei90/article/details/82875496#ha)

Flink On Yarn 时需要依赖 Yarn 来实现高可用，Yarn 会对失败挂掉的 JobManager(AM) 重启，最大重试次数的配置是`yarn-site.xml`的`yarn.resourcemanager.am.max-attempts`。

Flink 的 yarn-client 有一些配置可以控制在 container 失败的情况下的行为，也可通过`$FLINK_HOME/conf/flink-conf.yaml`或启动`yarn-session.sh`时以`-D`参数指定：

-   yarn.application-attempts  
    ApplicationMaster（运行着 JobManager）及其 TaskManager Container 的最大失败重试次数。

    默认值为 1，此时 AM 挂掉就直接导致整个 flink yarn session 失败了。所以一般需要设为较高的值，使得可在失败时可多次尝试重启 AM。如果超过该阈值，不会再重启 AM，该 yarn session 上提交的任务也会全部停止。

    比如设为 5 时，代表初始化一次、最大重试 4 次。

    注意该值不能超过 yarn 中`yarn.resourcemanager.am.max-attempts`配置的最大重启次数。

## 5.4 Job 优先级

可通过`$FLINK_HOME/conf/flink-conf.yaml`或启动`yarn-session.sh`时以`-D`参数指定：

-   yarn.application.priority  
    设为非负数时，指定 Flink 提交到 Yarn 上的 Application 的优先级，仅当 Yarn 开启了优先级调度时生效。更大的整数代表更高的优先级。

    如果设为负数或默认值 - 1，则不会设置 yarn 优先级，而是使用默认的集群优先级。

## 5.5 yarn session 失败调试

### 5.5.1 日志文件

基于 Yarn 的日志聚合功能，在`yarn-site.xml`配置`yarn.log-aggregation-enable`为 true，然后就能使用以下命令查看日志：

```shell
bin/yarn logs -applicationId <application_id>

```

### 5.5.2 yarn client 控制台或 WebUI

-   yarn client 会在错误发生在运行期（比如某个 TaskManager 在工作一段时间后挂掉）时也会打印错误信息到控制台。
-   Yarn WebUI 也可看到错误日志

## 5.6 yarn-job 停止

### 5.6.1 界面方式

进入 flink job 监控界面，点击进入要停止的 Job  
![](https://img-blog.csdnimg.cn/20191224165434727.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

然后点击右上角的`Cancel Job`  
![](https://img-blog.csdnimg.cn/20191224165521554.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

### 5.6.2 yarn 命令

-   查看任务状态

```shell
yarn application -status application_id

```

-   kill 任务

```shell
yarn application -kill application_id

```

## 5.7 从 Checkpoint/Savepoint 恢复

### 5.7.1 从 Checkpoint 恢复

可参考：

-   [Checkpoints](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/state/checkpoints.html)

Checkpoint 与 Savepoint 相比，有一些不同：

-   Checkpoint 使用 StateBackend 特定的数据格式，可以增量方式存储。
-   Checkpoint 不支持 Flink 的特定功能，比如扩缩容。

命令格式如下：

```shell
$ bin/flink run -s :checkpointMetaDataPath [:runArgs]

```

从 Checkpoint 恢复步骤实例：

1.  先在 flink 监控界面看最近成功的那次 checkpoint 路径：  
    ![](https://img-blog.csdnimg.cn/20200310123822783.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
2.  Cancel 要恢复的 Flink 任务  
    注意：必须先在代码或`flink-conf.yaml`中配置 checkpoint 在 Cancel 后依然保留，否则 checkpoint 会在 Cancel 后删除！

```java
val env = StreamExecutionEnvironment.getExecutionEnvironment
  
  .enableCheckpointing(5 * 60 * 1000)

val checkpointConfig = env.getCheckpointConfig
checkpointConfig.setMinPauseBetweenCheckpoints(2 * 60 * 1000)
checkpointConfig.setCheckpointTimeout(3 * 60 * 1000)

checkpointConfig.enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION)

```

3.  启动恢复任务  
    注意要找到`chk-xxx`目录的下一级`_metadata`目录来恢复任务：\`\`\`shell
    $ bin/flink run -m yarn-cluster -ynm kafka-to-hdfs -yjm 2048m -ytm 8192m -p 40 -s hdfs:/tmp/flink/flink-checkpoints/v78e8c6f8b885cab29c11976bd5c8hj8/chk-111/\_metadata ./user_jars/kafka2hdfs-1.0-SNAPSHOT.jar --groupid Kafka2Hive &

    ```

    ```

### 5.7.2 从 Savepoint 恢复

可参考：

-   [Savepoints-Cli](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/cli.html#savepoints)
-   [Savepoints - 概念](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/state/savepoints.html)

#### 5.7.2.1 Trigger a Savepoint

命令格式：

```shell
./bin/flink savepoint <jobId> [savepointDirectory]

```

-   savepointDirectory 可选 (该路径必须是 JobManager 可访问)，如果 flink-conf.yaml 中配置了`state.savepoints.dir`则可以不填，如果没配也没填否则会导致 savepoint 失败。
-   命令执行后会返回 savepoint 的路径信息，该信息被用来恢复 Flink 任务或杀死 savepoint。

#### 5.7.2.2 Trigger a Savepoint with YARN

该命令是针对 Yarn 上的 Flink 程序生成 Savepoint。

命令格式：

```shell
./bin/flink savepoint <jobId> [savepointDirectory] -yid <yarnAppId>

```

-   savepointDirectory 可选 (该路径必须是 JobManager 可访问)，如果 flink-conf.yaml 中配置了`state.savepoints.dir`则可以不填，如果没配也没填否则会导致 savepoint 失败。
-   yid 为 Flink 程序在 Yarn 上运行的 application id
-   命令执行后会返回 savepoint 的路径信息，该信息被用来恢复 Flink 任务或杀死 savepoint。

#### 5.7.2.3 Stop

该命令用来优雅停止一个 Flink Streaming Job，并同时生成 savepoint。

命令格式：

```shell
./bin/flink stop [-p targetDirectory] [-d] <jobID>

```

`stop`命令是停止正在运行的 Flink Streaming Job 的一种更为优雅的方法，因为`stop`信号从 source 流向 sink。

1.  具体来说，当我们发出`stop`命令时，所有`source`会收到请求，随后发送最后一次`checkpoint barrier`给下游触发一次 savepoint。
2.  savepoint 成功执行后，触发`SourceFunction.cancel()`结束 Source。
3.  如果加入了`-d`选项，则`new Watermark(Long.MAX_VALUE)`会在最后一个 checkpoint barrier 之前被发送到。这会导致所有已注册的 event-time timer 被触发，随后 flush 所有等待特殊 watermark 的 state，比如 Window 计算。
4.  该 job 会继续运行一段时间，直到所有 Source 正确关闭，处理完所有传输中数据。所有称为`优雅关闭`.

#### 5.7.2.4 Cancel with a savepoint (deprecated)

原子性的触发 Savepoint 并 Cancel 一个 job：

```shell
./bin/flink cancel -s [savepointDirectory] <jobID>

```

-   savepointDirectory 可选 (该路径必须是 JobManager 可访问)，如果 flink-conf.yaml 中配置了`state.savepoints.dir`则可以不填，如果没配也没填否则会导致 savepoint 失败。
-   Job 只会在 savepoint 成功后 Cancel
-   注意：本方法已经被标记为`deprecated`，使用`stop`替代

#### 5.7.2.5 Restore a savepoint

从 Savepoint 中恢复 Flink job:

```shell
./bin/flink run -s <savepointPath> ...

```

-   `run`提交 job 时带有`-s <savepointPath>`即可从该路径对应的 savepoint 中恢复，这里的路径就是我们之前用`savepoint`或`stop`命令停止后得到的。
-   默认会恢复所有 savepoint state，如果想跳过一些不适用于新 job 的 savepoint state 可设置`allowNonRestoredState`。

    比如新 Job 从代码中移除了一个算子，就可以这么做。

    ```shell
    ./bin/flink run -s <savepointPath> -n ...

    ```

#### 5.7.2.6 Dispose a savepoint

丢弃指定路径的 savepoint：

```shell
./bin/flink savepoint -d <savepointPath>

```

如果使用了自定义 state，比如自定义 reduce、RocksDB 等，需要同时指定该 job 的 jar 包路径，才能通过用户代码的 ClassLoader 来丢弃该 savepoint，否则会报错`ClassNotFoundException`：

```shell
./bin/flink savepoint -d <savepointPath> -j <jarFile>

```

也可以手动从文件系统删除 savepoint 目录，不会影响别的 savepoint/checkpoint。

## 5.8 Flink 启动配置

可参考

-   [Flink-Config](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/config.html)

### 5.8.1 flink-conf.yaml

可以在该文件中设置所有支持的 Flink Option，格式如：

```yaml
env.java.opts: -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:+AlwaysPreTouch -server -XX:+HeapDumpOnOutOfMemoryError

jobmanager.rpc.address: localhost
jobmanager.rpc.port: 6123

```

### 5.8.2 -yD

可以在启动 flink on yarn 时通过`-yD`参数指定，比如：

```shell
./flink-1.2.0/bin/flink run -m yarn-cluster -yn 4 -yjm 2048 -ytm 8086 -c beam.count.OpsCount -yqu data-default \
-yD taskmanager.heap.size=4096 -yD yarn.heap-cutoff-ratio=0.6 -yD taskmanager.debug.memory.startLogThread=true -yD taskmanager.debug.memory.logIntervalMs=600000 \
-yz toratest -yst -yd ./xxx.jar --parallelism=4
-yD env.java.opts="-XX:NewRatio=2"

```

此时用`jmap -heap`看一个 container 内存情况如下：

```shell
Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 2359296000 (2250.0MB)
   NewSize                  = 786432000 (750.0MB)
   MaxNewSize               = 786432000 (750.0MB)
   OldSize                  = 1572864000 (1500.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 476053504 (454.0MB)
   used     = 224818576 (214.40370178222656MB)
   free     = 251234928 (239.59629821777344MB)
   47.225484974058716% used
From Space:
   capacity = 101187584 (96.5MB)
   used     = 85147536 (81.20301818847656MB)
   free     = 16040048 (15.296981811523438MB)
   84.14820537665965% used
To Space:
   capacity = 155189248 (148.0MB)
   used     = 0 (0.0MB)
   free     = 155189248 (148.0MB)
   0.0% used
PS Old Generation
   capacity = 1572864000 (1500.0MB)
   used     = 922110224 (879.3928375244141MB)
   free     = 650753776 (620.6071624755859MB)
   58.62618916829427% used

22206 interned Strings occupying 2255248 bytes.

```

### 5.8.3 JVM 参数设置

可参考:

-   [JVM and Logging Options](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/config.html#jvm-and-logging-options)

#### 5.8.3.1 Flink Java 配置项

| Key                         | Default | Type    | Description                                                                                                                                                                                                               |
| --------------------------- | ------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| env.hadoop.conf.dir         | (none)  | String  | Path to hadoop configuration directory. It is required to read HDFS and/or YARN configuration. You can also set it via environment variable.                                                                              |
| env.java.opts               | (none)  | String  | 用于启动所有 Flink 进程的 JVM 的 Java 配置                                                                                                                                                                                            |
| env.java.opts.historyserver | (none)  | String  | JVM HistoryServer java 配置项                                                                                                                                                                                                |
| env.java.opts.jobmanager    | (none)  | String  | JVM JobManager java 配置项                                                                                                                                                                                                   |
| env.java.opts.taskmanager   | (none)  | String  | JVM TaskManager java 配置项                                                                                                                                                                                                  |
| env.log.dir                 | (none)  | String  | Defines the directory where the Flink logs are saved. It has to be an absolute path. (Defaults to the log directory under Flink’s home)                                                                                   |
| env.log.max                 | 5       | Integer | The maximum number of old log files to keep.                                                                                                                                                                              |
| env.ssh.opts                | (none)  | String  | Additional command line options passed to SSH clients when starting or stopping JobManager, TaskManager, and Zookeeper services (start-cluster.sh, stop-cluster.sh, start-zookeeper-quorum.sh, stop-zookeeper-quorum.sh). |
| env.yarn.conf.dir           | (none)  | String  | Path to yarn configuration directory. It is required to run flink on YARN. You can also set it via environment variable.                                                                                                  |

#### 5.8.3.2 JVM 常用配置

详细配置可参考

-   [Java-JVM-GC - 配置](https://blog.csdn.net/baichoufei90/article/details/88074531#gc-config)

```json
堆设置
-Xms :初始堆大小
-Xmx :最大堆大小
-Xmn:设置年轻代大小
-XX:NewSize=n :设置年轻代大小
-XX:NewRatio=n: 设置年轻代和年老代的比值。如:为3，表示年轻代与年老代比值为1：3，年轻代占整个年轻代年老代和的1/4
-XX:SurvivorRatio=n :年轻代中Eden区与两个Survivor区的比值。注意Survivor区有两个。如：3，表示Eden：Survivor=3：2，一个Survivor区占整个年轻代的1/5
-XX:MaxPermSize=n :设置持久代大小
收集器设置
-XX:+UseSerialGC :设置串行收集器
-XX:+UseParallelGC :设置并行收集器
-XX:+UseParalledlOldGC :设置并行年老代收集器
-XX:+UseConcMarkSweepGC :设置并发收集器
垃圾回收统计信息
-XX:+PrintHeapAtGC GC的heap详情
-XX:+PrintGCDetails  GC详情
-XX:+PrintGCTimeStamps  打印GC时间信息
-XX:+PrintTenuringDistribution    打印年龄信息等
-XX:+HandlePromotionFailure   老年代分配担保（true  or false）
并行收集器设置
-XX:ParallelGCThreads=n :设置并行收集器收集时使用的CPU数。并行收集线程数。
-XX:MaxGCPauseMillis=n :设置并行收集最大暂停时间
-XX:GCTimeRatio=n :设置垃圾回收时间占程序运行时间的百分比。公式为1/(1+n)
并发收集器设置
-XX:+CMSIncrementalMode :设置为增量模式。适用于单CPU情况。
-XX:ParallelGCThreads=n :设置并发收集器年轻代收集方式为并行收集时，使用的CPU数。并行收集线程数

```

## 5.9 Flink On Yarn 小结

优点

-   直接对接主流 Hadoop 2.4 + 版本

-   Yarn 是主流调度技术，可支持大多大数据栈，如 MR、Spark 等，统一资源管理

-   由 Yarn 负责资源管理，用户只需专注 Flink 代码开发

-   支持 Native 模式部署，动态申请 TM 资源，提高资源利用率

-   可利用 Yarn Failover 机制实现 JM、TM 容错  
    缺点

-   Yarn 黑盒，出现问题不好排查

-   Yarn 对内存隔离做的较好，但对其他资源如 CPU、网络隔离做的较差

-   [Apache Flink on Kubernetes：四种运行模式详解](http://www.dockone.io/article/10372)

-   [极客时间 - Flink On K8S 详解](https://time.geekbang.org/course/detail/100058801-281856)

可参考[flink-metrics-monitor](https://ci.apache.org/projects/flink/flink-docs-release-1.9/monitoring/metrics.html)

## 7.1 概述

在 flink 系统中，可以通过 REST API、Flink WebUI 等方式查看 Flink 系统指标，还可以通过代码方式自定义指标注册到 Flink。

此外还能配置 Reporter 来将 Flink 指标暴露给第三方监控工具，比如 Prometheus 等。

## 7.2 Prometheus 配置

1.  需要先将`flink-metrics-prometheus_2.11-1.9.0.jar`拷贝到`$FLINK_HOME/lib`
2.  修改 \`flink-conf.yaml\`\`\`\`shell
    vim $FLINK_HOME/conf/flink-conf.yaml

    metrics.reporter.prom.class: org.apache.flink.metrics.prometheus.PrometheusReporter

    metrics.reporter.prom.port: 9250-9350

    ```

    ```

## 6.3 背压 Back Pressure

可参考:

-   [浅析背压（Back Pressure）机制及其在 Spark & Flink 中的实现](https://blog.csdn.net/baichoufei90/article/details/103396623)
-   [Monitoring Back Pressure](https://ci.apache.org/projects/flink/flink-docs-release-1.9/monitoring/back_pressure.html)

## 81 内存调优

### 8.1.1 内存调优指南

-   [Memory tuning guide](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_tuning.html)

### 8.1.2 Flink JobManager 内存调优

参考:

-   [Flink JobManager Memory Options](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/config.html#jobmanager-heap-size)

本篇文章主要讲了下 Flink 的安装和示例程序的提交，希望大家有所收获。

下一章我们学习下 Flink 的 API，看看 Flink 作者是怎么抽象 API 的：  
[Flink 系列 3-API 介绍](https://blog.csdn.net/baichoufei90/article/details/82891909)

-   [Flink-Quickstart](https://ci.apache.org/projects/flink/flink-docs-release-1.10/quickstart/setup_quickstart.html)
-   [Flink 总结 - 设置 Jvm 参数](https://www.jianshu.com/p/6f2f53358b9c) 
    [https://blog.csdn.net/baichoufei90/article/details/82884554](https://blog.csdn.net/baichoufei90/article/details/82884554)
