# Airflow 实践笔记-从入门到精通 - 知乎
[Airflow 实践笔记 - 从入门到精通 - 知乎](https://zhuanlan.zhihu.com/p/517364346) 

 数据处理逻辑**多**，脚本相互依赖**强**，运维管理监测**难**，怎么办？！为了解决这些问题，最近比较深入研究 Airflow 的使用方法，重点参考了[官方文档](https://link.zhihu.com/?target=https%3A//airflow.apache.org/docs/apache-airflow/2.2.5/concepts/overview.html%23)和[Data Pipelines with Apache Airflow](https://link.zhihu.com/?target=https%3A//www.manning.com/books/data-pipelines-with-apache-airflow)，特此笔记，跟大家分享共勉。

## Airflow 项目

2014 年在 Airbnb 的 Maxime Beauchemin 开始研发 airflow，经过 5 年的开源发展，airflow 在 2019 年被 apache 基金会列为高水平项目 Top-Level Project。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/a3b863b0-0fa0-475c-ba6f-ba63630806ea.jpeg?raw=true)

Maxime 目前是[Preset](https://link.zhihu.com/?target=https%3A//preset.io/)(Superset 的商业化版本) 的 CEO，作为 Apache Airflow 和 Apache Superset 的创建者，世界级别的数据工程师，他这样描述 “数据工程师”（[原文](https://link.zhihu.com/?target=https%3A//www.montecarlodata.com/blog-the-future-of-the-data-engineer/)）：随着大数据和云计算的普及，数据工程师的角色和责任也更加多样化，包括**ETL 开发**、维护数据平台、搭建基于**云的数据基础设施**、**数据治理**，同时也是负责良好数据习惯的**守护者、守门人**，负责在数据团队中推广和普及最佳实践，尤其是在效率（处理增量负载）、数据建模和编码标准方面，依靠数据可观察性和 DataOps 来确保每个人都以相同的方式处理数据。

源自创建者深刻的理解和设计理念，加上开源社区在世界范围聚集人才的组织力，Airflow 取得当下卓越的成绩。作为一款优秀的数据工作流的管理工具，已被广泛的应用在包括 Adobe, Airbnb, Etsy, Google, ING, Lyft, PayPal, Reddit, Square, Twitter, and United Airlines 等世界知名的公司。Airflow 完全是 python 语言编写的，加上其开源的属性，具有非常强的扩展和二次开发的功能，能够最大限度的跟其他大数据产品进行融合使用，包括 AWS S3, Docker, Apache Hadoop HDFS, Apache Hive, Kubernetes, MySQL, Postgres, Apache Zeppelin 等。

## Airflow 可实现的功能

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/12ab8fc4-ba7d-4122-8058-aec66f50d718.jpeg?raw=true)

Apache Airflow 提供基于 DAG 有向无环图来编排工作流的、可视化的分布式任务调度，与 Oozie、Azkaban 等任务流调度平台类似。采用 Python 语言编写，提供可编程方式定义 DAG 工作流，可以定义一组有依赖的任务，按照依赖依次执行， 实现任务管理、调度、监控功能。此外提供 WebUI 可视化界面，提供了工作流节点的运行监控，查看每个节点的运行状态、运行耗时、执行日志等。

## 主要概念

-   Data Pipeline：数据管道或者数据流水线，可以理解为贯穿数据处理分析过程中不同工作环节的流程，例如加载不同的数据源，数据加工以及可视化。
-   DAGs：是有向非循环图（directed acyclic graphs），可以理解为有先后顺序任务的多个 Tasks 的组合。图的概念是由节点组成的，有向的意思就是说节点之间是有方向的，转成工业术语我们可以说节点之间有依赖关系；非循环的意思就是说节点直接的依赖关系只能是单向的，不能出现 A 依赖于 B，B 依赖于 C，然后 C 又反过来依赖于 A 这样的循环依赖关系。每个 Dag 都有唯一的 DagId，当一个 DAG 启动的时候，Airflow 都将在数据库中创建一个 DagRun 记录，相当于一个日志。
-   Task：是包含一个具体 Operator 的对象，operator 实例化的时候称为 task。DAG 图中的每个节点都是一个任务，可以是一条命令行（BashOperator），也可以是一段 Python 脚本（PythonOperator）等，然后这些节点根据依赖关系构成了一个图，称为一个 DAG。当一个任务执行的时候，实际上是创建了一个 Task 实例运行，它运行在 DagRun 的上下文中。
-   Connections：是管理外部系统的连接对象，如外部 MySQL、HTTP 服务等，连接信息包括 conn_id／hostname／login／password／schema 等，可以通过界面查看和管理，编排 workflow 时，使用 conn_id 进行使用。
-   Pools: 用来控制 tasks 执行的并行数。将一个 task 赋给一个指定的 pool，并且指明 priority_weight 权重，从而干涉 tasks 的执行顺序。
-   XComs：在 airflow 中，operator 一般是原子的，也就是它们一般是独立执行，不需要和其他 operator 共享信息。但是如果两个 operators 需要共享信息，例如 filename 之类的，则推荐将这两个 operators 组合成一个 operator；如果一定要在不同的 operator 实现，则使用 XComs (cross-communication) 来实现在不同 tasks 之间交换信息。在 airflow 2.0 以后，因为 task 的函数跟 python 常规函数的写法一样，operator 之间可以传递参数，但本质上还是使用 XComs，只是不需要在语法上具体写 XCom 的相关代码。
-   Trigger Rules：指 task 的触发条件。默认情况下是 task 的直接上游执行成功后开始执行，airflow 允许更复杂的依赖设置，包括 all_success(所有的父节点执行成功)，all_failed(所有父节点处于 failed 或 upstream_failed 状态)，all_done(所有父节点执行完成)，one_failed(一旦有一个父节点执行失败就触发，不必等所有父节点执行完成)，one_success(一旦有一个父节点执行成功就触发，不必等所有父节点执行完成)，dummy(依赖关系只是用来查看的，可以任意触发)。另外，airflow 提供了 depends_on_past，设置为 True 时，只有上一次调度成功了，才可以触发。
-   Backfill: 可以支持重跑历史任务，例如当 ETL 代码修改后，把上周或者上个月的数据处理任务重新跑一遍。
-   Airflow 2.0 API，是一种通过修饰函数，方便对图和任务进行定义的编码方式，主要差别是 2.0 以后前一个任务函数作为后一个任务函数的参数，通过这种方式来定义不同任务之间的依赖关系。
-   AIRFLOW_HOME 是 Airflow 寻找 DAG 和插件的基准目录。当数据工程师开发完 python 脚本后，需要以 DAG 模板的方式来定义任务流，然后把 dag 文件放到 AIRFLOW_HOME 下的 DAG 目录，就可以加载到 airflow 里开始运行该任务。

## 安装 Airflow

Airflow 适合安装在 linux 或者 mac 上，官方推荐使用 linux 系统作为生产系统。如果要在 windows 安装，就需要通过 WSL2 (Windows Subsystem for Linux 2) 一种 windows 版本但是能运行 linux 命令的子系统，或者通过 Linux Containers 容器来安装。

这里我们选择在 windows 环境下（日常个人的开发环境是 windows）通过容器来安装，首先要安装 docker。如果在安装 docker 时有报错信息 “Access denied. You are not allowed to use docker. You must be in the “docker-users” group”，看上去是权限问题，但实际上很有可能是因为 windows 版本的问题。具体查看 windows 安装容器前提条件:[https://docs.docker.com/desktop/windows/install/](https://link.zhihu.com/?target=https%3A//docs.docker.com/desktop/windows/install/)，这是安装 WSL 2 backend 的指南。重要是其中两个步骤，一个是要开启 WSL 2 功能，一个是安装 Linux 内核更新包。

### 制作 Dockerfile 文件

使用 freeze 命令先把需要在 python 环境下安装的包依赖整理出来，看看哪些包是需要依赖的。使用命令 pip freeze > requirements.txt

准备镜像的时候，可以继承 (extend)airflow 已经做好的官方镜像，也可以自己重新 customize 自定义镜像。这里我们使用 extend 的方法，会更加快速便捷。

该镜像默认的 airflow_home 在容器内的地址是 / opt/airflow/，dag 文件的放置位置是 /opt/airflow/dags。这个镜像同时定义了 “airflow” 用户，所以如果要安装一些工具的时候（例如 build-essential 这种 linux 下的开发必要工具），需要切换到 root 用户，用 pip 的时候要切换回 airflow 用户。更多参考 [https://airflow.apache.org/docs/docker-stack/build.html](https://link.zhihu.com/?target=https%3A//airflow.apache.org/docs/docker-stack/build.html)。

在官方镜像中，用户 airflow 的用户组 ID 默认设置为 0（也就是 root），所以为了让新建的文件夹可以有写权限，都需要把该文件夹授予权限给这个用户组。

以下是具体 dockerfile 的内容

```text
#使用官方发布的镜像
FROM apache/airflow:2.3.0   
# 安装软件的时候要用root权限
USER root  
RUN apt-get update \
  && apt-get install -y --no-install-recommends \
         vim \
  && apt-get autoremove -yqq --purge \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*

# pip安装用airflow用户
USER airflow  
COPY requirements.txt /tmp/requirements.txt

#使用requirements安装指定包的例子
RUN pip install -r /tmp/requirements.txt  
# 一个用pip安装指定包的例子
#RUN pip install --no-cache-dir apache-airflow-providers-docker==2.5.1
# 拷贝DAG文件，并且设置权限给airflow
COPY --chown=airflow:root BY02_AirflowTutorial.py /opt/airflow/dags  
COPY src/data.sqlite /opt/airflow/data.sqlite
#建立一个可以写的文件夹，这里的~指的是主目录
RUN umask 0002; \
    mkdir -p ~/writeable_directory
```

### 容器部署

准备好 dockerfile 以及相关的文件 (例如脚本 dag.py 和数据库 sqlite)，具体部署有两种方法：

一种方法是采用 docker 命令。

运行命令来生成镜像

```text
docker build -t airflow:latest 
```

镜像做好以后，需要使用 docker run 来启动镜像，不要用 docker desktop 的启动按钮（会默认使用 airflow 的命令，会报如下错误 airflow command error: the following arguments are required: GROUP_OR_COMMAND, see help above）。

运行下面的命令：其中 -it 意思是进入容器的 bash 输入, --env 是设置管理者密码

```text
docker run -it --name  test -p 8080:8080   --env "_AIRFLOW_DB_UPGRADE=true"  --env "_AIRFLOW_WWW_USER_CREATE=true" --env "_AIRFLOW_WWW_USER_PASSWORD=admin"  airflow:latest airflow standalone      
```

第二种方法是：按照官方教程使用 docker compose（将繁琐多个的 Docker 操作整合成一个命令）来创建镜像并完成部署。

在 windows 环境下，安装 docker desktop 后默认就安装了 docker-compose 工具。Docker Compose 使用的模板文件是 docker-compose.yml，其中定义的每个服务都必须通过 image 指令指定镜像或使用 Dockerfile 的 build 指令进行自动构建，其它大部分指令跟 docker run 的选项类似。Compose 使用的三个步骤：

1）使用 Dockerfile 定义应用程序的环境。

2）使用 docker-compose.yaml 定义构成应用程序的服务，这样它们可以在隔离环境中一起运行。

3）执行 docker-compose up 命令来启动并运行整个应用程序。

Docker descktop 的配置要把内存调整到 4G 以上，否则后续可能会报内存不足的错误。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/a16d0c6d-7f0b-4eca-82ca-a743adda1aef.jpeg?raw=true)

同时需要把本地 yaml 所在文件夹加入到允许 file sharing 的权限，否则后续创建容器时可能会有报错信息 “Cannot create container for service airflow-init: user declined directory sharing”

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/57bbf4cf-8b86-4423-a3a0-807e94bc1799.jpeg?raw=true)

Airflow 官方教程中使用 CeleryExecutor 来进行容器部署，会使用 compose 命令建立多个容器，不同的容器承担不同的服务。直接使用官方提供的 yaml 文件（[https://airflow.apache.org/docs/apache-airflow/2.2.5/docker-compose.yaml](https://link.zhihu.com/?target=https%3A//airflow.apache.org/docs/apache-airflow/2.2.5/docker-compose.yaml)）

这个 yaml 文件包含的操作主要是

1) 安装 airflow，使用官方镜像（也可以自定义镜像），定义环境变量（例如数据库的地址）

2）安装 postgres 服务，指定其对应的镜像

3）安装 Redis，作为 celery 的 broker

4）启动 airflow 的 webserver 服务

5）启动 airflow 的 schedule 服务

6）启动 worker node

7）启动 trigger 服务，这是一个新的组件，目的是检查任务正确性

8）数据库初始化

同样的目录下，新建一个名字为. env 文件，跟 yaml 文件在一个文件夹。里面内容为 AIRFLOW_UID=50000，主要是为了 compose 的时候赋予运行容器的 userID, 50000 是默认值。

在 cmd 界面进入 yaml 所在文件夹，运行以下命令就可以自动完成容器部署并且启动服务。运行 docker ps 应该可以看到 6 个在运行的容器

## 运行 airflow

安装万 airflow 后，运行以下命令会将相关的服务启动起来

上面的命令等同于下面的命令，逐个启动相关服务

```text
airflow db init
airflow users create \
    --username admin \
    --firstname Peter \
    --lastname Parker \
    --role Admin \
    --email spiderman@superhero.org
airflow webserver --port 8080
airflow scheduler
```

在 terminal 初始化数据库，会在 / Users/XXXX/airflow / 下生成 airflow.db 的 SQLiteDB（默认的数据库），可以进一步查看其底层设计的表结构。 这个数据库被称为 metastore 元数据存储。

默认前台 web 管理界面会加载 airflow 自带的 dag 案例，如果不希望加载，可以在配置文件中修改 AIRFLOW\_\_CORE\_\_LOAD_EXAMPLES=False，然后重新 db init

## 参数配置

/Users/XXXX/airflow/airflow.cfg 是配置表，里面可以配置连接数据库的字符串，配置变量是 sql_alchemy_conn。

Airflow 默认使用 SQLite，但是如果生产环境需要考虑采用其他的数据库例如 Mysql,PostgreSQL(因为 SQLite 只支持 Sequential Executor，就是非集群的运行)。

当设置完这个配置变量，就可以 airflow db init，自动生成后台数据表。配置文件中的 secrets backend 指的是一种管理密码的方法或者对象，数据库的连接方式是存储在这个对象里，无法直接从配置文件中看到，起到安全保密的作用。

AIRFLOW\_\_CORE\_\_DAGS_FOLDER 是放置 DAG 文件的地方，airflow 会定期扫描这个文件夹下的 dag 文件，加载到系统里。当然这会消耗系统资源，所以可以通过设置其他的参数来减少压力。例如 AIRFLOW\_\_SCHEDULER\_\_PROCESSOR_POLL_INTERVAL

AIRFLOW\_\_CORE\_\_EXECUTOR 配置使用哪种 executor

如果不想加载 airflow 自带的案例，可以配置

n Set AIRFLOW\_\_CORE\_\_LOAD_EXAMPLES=False

n Set AIRFLOW\_\_CORE\_\_LOAD_DEFAULT_CONNECTIONS=False

如果需要对 web 管理界面自定义，例如 颜色、title 等，参考

[https://airflow.apache.org/docs/apache-airflow/2.2.5/howto/customize-ui.html](https://link.zhihu.com/?target=https%3A//airflow.apache.org/docs/apache-airflow/2.2.5/howto/customize-ui.html)

如果需要配置邮件，参考

[https://airflow.apache.org/docs/apache-airflow/2.2.5/howto/email-config.html](https://link.zhihu.com/?target=https%3A//airflow.apache.org/docs/apache-airflow/2.2.5/howto/email-config.html)

## web 管理界面

在界面中，先要把最左边的 switch 开关打开，然后再按最右边的开始箭头，就可以启动一个 DAG 任务流。启动任务流的方式还有两种：CLI 命令行方式和 HTTP API 的方式

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/5f0bd4ac-89e8-40dc-a304-28c37d899287.jpeg?raw=true)

点击 link->graph，可以进一步看到网状的任务图，点击每一个任务，可以看到一个菜单，里面点击 log，可以看到具体的执行日志。如果某个任务失败了，可以点击图中的 clear 来清除状态，airflow 会自动重跑该任务。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/648e271a-1d5b-467c-b4c4-32b8badbd294.jpeg?raw=true)

菜单点击 link->tree，可以看到每个任务随着时间轴的执行状态。

菜单 admin 下的 connections 可以管理数据库连接 conn 变量，后续 operator 在调用外部数据库的时候，就可以直接调用 conn 变量。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/19d1c8cf-89ee-4c2a-9663-0bb7fcfb191d.png?raw=true)

在 airflow 的前端页面、日志、以及数据库中的元数据，默认使用 UTC 时间。如果我们在 DAG 定义的时候，指定 start_date 为中国时区，界面里面的时间元素会转为本地的中国时间，但并不是全部，所以有的时候看起来比较 confusing。 在 UI 前端，点击列 run 下面的数字，进入 “dag run 列表”，里面有如下时间相关的字段：

 LogicalDate：按照逻辑计划的时间，如果该任务是 schedule 触发的方式，就是计划任务预定的执行时间，例如每天的凌晨 1 点。如果该任务是手动触发的，则是触发发生的时间。如果在 dag 设置了时区，会显示本地时区的时间格式。

 RunId: 如果是 schedule 启动的，值是以 schedule 开头；如果是手动触发，值是以 manual 开头，后面跟着是 UTC 格式的系统时间。

 Queued at: 任务排队的时间，基本上都等于任务的启动时间（如果计算资源充分的情况）

 Start Date: 实际执行的开始时间。如果在 dag 设置了时区，会显示本地时区的时间格式。

 End Date: 实际执行的结束时间。如果在 dag 设置了时区，会显示本地时区的时间格式。

在手动触发任务的时候，是无法执行还未到执行时间的定时任务的。例如 每天下午 3 点的任务，如果是今天下午 2 点 30 手动触发，实际上跑的是昨天下午 3 点的脚本。手动触发时，有一个选项 trigger DAG with config，点击进去可以选择 logical date，例如前天或者大前天的任务；但是如果选择明天的时间，会发现这个任务会处于排队 queue 的状态。

注意，虽然在 DAG 中设置了时区，但是在传递给 task 函数的模板变量，例如 execution_date、ts 等，还是维持为 UTC 时间。因为本质上，airflow 内部还是统一使用 UTC 来传递参数，本地时间的转换只是方便展示。

这些模板变量，可以很好的记录定时脚本每次执行的计划时间或者执行时间，因此方便任务的回填，例如把过去 10 天的脚本都按照原来计划的定时任务全部一批次跑一遍。同时，这些模板变量可以很好的作为全局共有变量，在不同的 task 函数中传递共享。

## 用户权限管理

在 airflow 中，有不同的角色，admin 拥有最大的权限，Viewer 可以查看前端 UI 的内容，

User 除了查看还有具体操作（对 DAG 的增删改查）的权限，Op 在 User 的基础上增加了参数配置的权限。

如果需要对某一个具体的用户只赋予查看局部 DAG 以及操作 DAG 的权限，就赋予其以下权限：

-   can read on Website: 可以登录 UI 系统
-   can read on DAG（Dag 名称）：可以访问具体的 DAG
-   can eidt on DAG（Dag 名称）：可以停止或者开启具体的 DAG
-   can read on DAG Runs: 可以查看每一次 Run 的日志
-   can read on Task Instances/can read on Task Logs: 可以点击 DAG 进入查看其日志
-   can delete on Task Instances: 可以将某一个任务 clear，重启任务

## DAG

配置表中的变量 DAG_FOLDER 是 DAG 文件存储的地址，DAG 文件是定义任务流的 python 代码，airflow 会定期去查看这些代码，自动加载到系统里面。

DAG 是多个脚本处理任务组成的工作流 pipeline，概念上包含以下元素

1) 各个脚本任务内容是什么

2) 什么时候开始执行工作流

3) 脚本执行的前后顺序是什么

针对 1），通过 operator 来实现对任务的定义。Operator，翻译成 “操作单元”，有很多种形式，可以是一个 bash 命令，也可以是一个 python 函数，或者是一个数据库连接任务。Airflow 封装了很多 operator，开发者基于需要来做二次开发。实际上各种形式的 operator 都是 python 语言写的对象。

针对 2），在 DAG 的配置函数中有一个参数 schedule_interval，约定被调度的频次，是按照每天、每周或者固定的时间来执行。这个参数，跟 start_date 开始时间和 end_date 结束时间（需要某个时间段后不需要执行该任务）配合着用，来约定什么时候跑这个 DAG。logical date 指的是这个 DAG 后续预计执行发生的时间。

下图是参数设置为 @daily 的执行节奏

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/13dc46e9-beb1-4a42-a266-aa1016f1bdae.jpeg?raw=true)

airflow 有事先定义好的参数，例如 @daily,@hourly,@weekly 等，一般场景下足够使用，如果需要更精细化的定义，可以使用 cron-based 配置方法、基于次数的方法。

Schedule 本质上是一个 while true 循环，不断检查每个任务的状态，如果其上游任务都跑完，并且当前系统资源足够 task slots，就会把该任务变成 queued 状态，等待 executor 去具体执行

针对 3），使用 >> 或者 &lt;&lt;来定义任务之间的依赖关系，例如 start>> \[fetch_weather, fetch_sales]意思是，start 执行完以后，同时执行 fetch_weather 和 fetch_sales。进一步定义\[clean_weather, clean_sales] >> join_datasets，就会形成下面的 DAG 图。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/ac3be06e-5b6f-4110-943e-6290e32e75b9.jpeg?raw=true)

注意：在图里面的分支，有的时候是都需要执行，有的时候可能两个分支会根据条件选择一个分支执行。这种分支判断 (branch) 的逻辑，可以在函数里面写，也可以通过 brach operator 实现。用后者的好处是，可以在 DAG 里面直观的看到具体执行的是哪个分支。

一般来讲，只有当上游任务 “执行成功” 时，才会开始执行下游任务。但是除了“执行成功 all_success” 这个条件以外，还有其他的 trigger rule，例如 one_success, one_failed（至少一个上游失败），none_failed ，none_skipped

DAG 在配置的时候，可以配置同时运行的任务数 concurrency，默认是 16 个。这个 16，就是 task slot，可以理解为资源，如果资源满了，具备运行条件的 task 就需要等待。

定义 DAG 的方式有两种：可以使用 with 语法，也可以使用修饰函数 @dag。

```text
with DAG(
    dag_id='example_bash_operator',
    schedule_interval='0 0 * * *',
    start_date=pendulum.datetime(2021, 1, 1, tz="UTC"),
    catchup=False,
    dagrun_timeout=datetime.timedelta(minutes=60),
    tags=['example', 'example2'],
    params={"example_key": "example_value"},
) as dag:
```

配置 DAG 的参数：

| 'depends_on_past': False, | 前置任务成功后或者 skip，才能运行 |
| 'email': \['airflow@example.com'], | 警告邮件发件地址 |
| 'email_on_failure': False, | 失败的时候发邮件 |
| 'email_on_retry': False, | 任务重新尝试的时候发邮件 |
| 'retries': 1, | 尝试次数 |
| 'retry_delay': timedelta(minutes=5), | 尝试之间的间隔 |
| 'queue': 'bash_queue', | 指定一个 队列 运行该任务，CeleryExecutor 用到 |
| 'pool': 'backfill', |  |
| 'priority_weight': 10, | 任务优先级 |
| 'end_date': datetime(2016, 1, 1), | 任务计划的截止时间 |
| 'wait_for_downstream': False, | 如果前一个任务实例的下游任务没有跑完，该任务是否可以跑 |
| 'sla': timedelta(hours=2), | 如果在规定的时间间隔内任务没有跑完，会发警告 |
| 'execution_timeout': timedelta(seconds=300), | 如果执行超出所设置的时间，任务被当做失败 |
| 'on_failure_callback': some_function, | 当任务失败时，调用的函数 |
| 'on_success_callback': some_other_function, | 当任务成功时，调用的函数 |
| 'on_retry_callback': another_function, | 当任务重新尝试的时候，调用的函数 |
| 'sla_miss_callback': yet_another_function, |  |
| 'trigger_rule':'all_success' | 前置任务的执行状态符合什么条件时，该任务会被启动 |
| tags:\[‘example’] | 相当于是对 DAG 的一个分类，方便在前台 UI 根据 tag 来进行查询 |

DAG Run 是 DAG 运行一次的对象（记录），记录所包含任务的状态信息。如果所有的任务状态是 success 或者 skipped，就是 success；如果任务有 failed 或者 upstream_failed，就是 falied。

其中的 run_id 的前缀会有如下几个

-   scheduled\_\_ 表明是不是定时的
-   backfill\_\_ 表明是不是回填的
-   manual\_\_ 表明是不是手动或者 trigger 的

启动 DAG，除了根据定时方法，也可以通过 CLI 命令或者 Rest api 的方式。在调用的时候可以通过指定 dag_run.conf，作为参数让 DAG 根据不同的参数处理不同的数据。

在定义 DAG 的时候，有时会使用 Edge Labels，可以理解成是虚拟的节点，目的是为了在前端 UI 更方便看到任务之间的依赖关系（类似注释的方法）。为了提高相同 DAG 操作的复用性，可以使用 subDAG 或者 Taskgroup。

## 本地时间

前端页面、日志、以及数据库中的任务元数据，默认都是 UTC 时间。如果要改成本地时间，可以在 dag 设置时，定义 start_date = pendulum.datetime(2020,1,1,tz ="Asia/Shanghai")

在利用模板元素在不同的任务之间传递本地时间参数时，可以用

{{data_interval_end.in_timezone("Aisa/Shanghai") | ds_nodash}}

更多模板变量参考：[Templates reference](https://link.zhihu.com/?target=https%3A//airflow.apache.org/docs/apache-airflow/2.2.5/templates-ref.html)

## Operator

在任务流中的具体任务执行中，需要依据一些外部条件，例如之前任务的执行时间、开始时间等。这些 “公有变量参数”，我们称为模板参数。airflow 利用 Jinja templates，实现“公有变量” 调用的机制。在 bashoprator 中引用，例如 {{ execution_date}}就代表一个参数。在前端 UI 中，点击 graph 中的具体任务，在点击弹出菜单中 rendered tempalate 可以看到该参数在具体任务中代表的值。

除了公有变量，如果 operator 之间要互相传递参数或者中间过程数据，例如一个 operator 要依赖另一个 operator 的输出结果进行执行，有以下几个方式

-   使用 XCom，有点像 dict 对象，存储在 airflow 的 database 里，但是不适合数据量大的场景。另外，XCom 如果设置过多后，也无形中也增加了 operator 的约束条件且不容易直观发现。在前端 UI 的 adimin-》Xcoms 里可以看到各个 DAG 用到的值。Airflow2 中允许自定义 XCom，以数据库的形式存储，从而支持较大的数据。

```text
# 从该实例中的xcom里面取 前面任务train_model设置的键值为model_id的值。
model_id = context["task_instance"].xcom_pull(
 task_ids="train_model", key="model_id")
```

-   在 operator 中使用 op_kwargs，里面配置模板参数
-   存储在数据库，例如一个 operator 存储数据在外部数据库中，另一个 operator 查询该数据库获得数据
-   使用 Taskflow API，其实就是 @task 这样的修饰函数，被称为 TaskFlow function。在 python 函数上使用修饰函数 @task，就是 pythonOperator，也可以用 PythonOperator 来定义任务逻辑。不同的 task 之间直接用函数嵌套的方式来交换信息，例如 email_info = compose_email(get_ip())。这种方式跟传统的函数编程方式比较接近，同时也完成了依赖关系的定义，不需要使用 >> 来定义任务之间的依赖关系。这种 @修饰函数的方式，目前只限于 python 类型的 operator。task 可以用原来 1.0 的方式来定义，也可以用 @task 的方式来定义，相互之间如果需要传递参数，可以使用. output 的方法。 task 可以通过在函数参数中定义 \*\*kwargs，或者使用 get_current_context，获得该任务执行期间的上下文信息。

Operator 的类型有以下几种：

1） DummyOperator 

作为一个虚拟的任务节点，使得 DAG 有一个起点，但实际不执行任务；或者是在上游几个分支任务的合并节点，为了清楚的现实数据逻辑。

2）BashOperator

当一个任务是执行一个 shell 命令，就可以用 BashOperator。 可以是一个命令，也可以指向一个具体的脚本文件。

3） 条件分支判断

BranchDateTimeOperator

在一个时间段内执行一种任务，否则执行另一个任务。Target_lower 可以设置为 None

```text
cond1 = BranchDateTimeOperator(
 task_id='datetime_branch',
 follow_task_ids_if_true=['date_in_range'],
 follow_task_ids_if_false=['date_outside_range'],
 target_upper=pendulum.datetime(2020, 10, 10, 15, 0, 0),
 target_lower=pendulum.datetime(2020, 10, 10, 14, 0, 0),
 dag=dag,)
```

BranchDayOfWeekOperator

根据是哪一天来选择跑哪个任务

BranchPythonOperator

根据业务逻辑条件，选择下游的一个 task 运行

```text
dummy_task_1 = DummyOperator(task_id='branch_true', dag=dag)
dummy_task_2 = DummyOperator(task_id='branch_false', dag=dag)

branch = BranchDayOfWeekOperator(
 task_id="make_choice",
 follow_task_ids_if_true="branch_true",
 follow_task_ids_if_false="branch_false",
 week_day="Monday",
)

# Run dummy_task_1 if branch executes on Monday
branch >> [dummy_task_1, dummy_task_2]

```

4）PythonOperator

用的最广泛的 Operator，在 airflow1.0 的时候，定义 pythonOperator 会有两部分，一个是 operator 的申明，一个是 python 函数。这时候函数传参是需要用到 op_args 或者 op_kwargs 或者 templates_dict

```text
def _get_data(output_path, **context): year, month, day, hour, *_ = context["execution_date"].timetuple() url = ( 
 "https://dumps.wikimedia.org/other/pageviews/"
f"{year}/{year}-{month:0>2}/pageviews-{year}{month:0>2}{day:0>2}-{hour:0>2}0000.gz" ) 
 request.urlretrieve(url, output_path)
 
get_data = PythonOperator( 
task_id="get_data", 
python_callable=_get_data, 
op_args=["/tmp/wikipageviews.gz"], dag=dag, 
) 
 
 
def _calculate_stats(input_path, output_path):
 """Calculates event statistics."""
Path(output_path).parent.mkdir(exist_ok=True) events = pd.read_json(input_path) stats = events.groupby(["date", "user"]).size().reset_index() stats.to_csv(output_path, index=False) 
 
calculate_stats = PythonOperator(
 task_id="calculate_stats",
 python_callable=_calculate_stats,
 op_kwargs={
 "input_path": "/data/events.json",
 "output_path": "/data/stats.csv",
 },
dag=dag, ) 
```

在 airflow2.0 以后，用 TaskFlow API 以后，传参简单很多，就是当函数参数用即可。但是需要注意的是，这种传参本质上还是通过 xcom 来实现传递的, 必须是可序列号的对象，所以参数必须是 python 最基本的数据类型，像 dataframe 就不能作为参数来传递。

```text
@task(task_id="print_the_context")
def print_context(ds=None, **kwargs):
 """Print the Airflow context and ds variable from the context."""
 pprint(kwargs)
 print(ds)
 return 'Whatever you return gets printed in the logs'
```

5）图之间依赖关系的 operator

如果两个任务流之间，存在一些依赖关系。

使用 ExternalTaskSensor，根据另一个 DAG 中的某一个任务的执行情况，例如当负责下载数据的 DAG 完成以后，这个负责计算指标的 DAG 才能启动。

```text
child_task1 = ExternalTaskSensor(
 task_id="child_task1",
 external_dag_id=parent_dag.dag_id,
 external_task_id=parent_task.task_id,
 timeout=600,
 allowed_states=['success'],
 failed_states=['failed', 'skipped'],
 mode="reschedule",
)
```

当 任务图的某一个任务的执行状态被清理 (clear)，其相应影响的另一个图的任务状态也要随着连带被清理，就要用上 ExternalTaskMarker。使用 TriggerDagRunOperator ，可以让 DAG 的某一个任务 启动另一个 DAG

6）LatestOnlyOperator

LatestOnlyOperator，是为了标识该 DAG 是不是最新的执行时间，只有在最新的时候才有必要执行下游任务，例如部署模型的任务，只需要在最近一次的时间进行部署即可。

```text
from airflow.operators.latest_only import LatestOnlyOperator 
latest_only = LatestOnlyOperator(
 task_id="latest_only",
 dag=dag,
)
train_model >> latest_only >> deploy_model
```

7）Sensor

Sensor 是用来判断外部条件是否成熟的感应器，例如判断输入文件是否到位（可以设置一个时间窗口内，例如到某个时间点之前检查文件是否到位），但是 sensor 很耗费计算资源（设置 mode 为 reschedule 可以减少开销，默认是 poke），DAG 会设置 concurrency 约定同时最多有多少个任务可以运行，称为 task slot。所以一种办法是使用 Deferrable Operators。FileSensor，判断是否文件存在了；自定义 sensor，继承 BaseSensorOperator，通过实现 poke 函数来实现检查逻辑

Deferrable Operators 是 sensor operator 的替代品，其本质是应用了 python 的异步机制，有一个 trigger process，当其符合特定条件后，就会将任务调度起来。如果用原生的 Sensor Operator，并行 150 个任务的时候，会占用 13G 左右的内存，并且消耗 CPU 在无意义的轮询中；而用新的 Deferrable Operators 之后，额外使用的内存非常少只有几百兆。

Trigger 触发器，要继承自 BaseTrigger，重点要实现三个方法：

-   \_\_init\_\_函数，接受来自 Operator 的参数。因为 Trigger 是在 Deferrable Operator 的 execute 函数中完成初始化
-   Run 函数，是具体发出触发行为的逻辑实现，例如判断时间是否到了，数据是否到位。其中会用到 asyncio.sleep 的函数，其实质是把控制权让出去留给其他的任务。在 run 函数中会通过一个 while 循环来控制逻辑，也就是如果满足条件会一直在 while 中循环，但是通过 sleep 函数，可以作为异步操作，而不阻碍线程的工作。当满足条件后，就会跳出 while 循环了。从而实现类似触发器的机制。
-   Serialize 函数，序列化函数，目的是为了能够将该实例类存储到 airflow 的数据库中，然后还可以重新实例化，因为序列化后便于存储，就可以在多个子节点之间通过实例化继续使用，例如 celery 中的不同 worker 节点。为了让 airflow 能找到这个类地址，就需要用 sys.path.append 方法，把类代码的路径加进去。

注意：如果 Trigger 这个类写的有问题，在 airflow 运行时并不会在日志里面体现，会正常返回 被当作正常触发，怀疑是 报错时相当于发出了一个 TriggerEvent。所以事先要用 python 本地测试一下。

具体在实现 Deferrable Operator 的时候，有以下几个组成部分：

-   trigger 触发器
-   method_name: 当触发器激活时，需要 airflow 调用的函数，例如完成什么操作，也可以什么操作都不做正常返回，把需要做的动作放在下游的 operator 中。 按照官网的说明，会返回触发产生的 event 给这个方法，并且可以访问参数 payload，但实践过程中发现返回的 success 字符串。
-   kwargs: 传递给函数的参数
-   timeout: 需要延迟的时间

8）自定义 Operator

-   Hook 是一种自定义的 operator，可以理解为与外部系统的接口函数，类似数据库连接对象，负责权限认证、连接和关闭的动作。根据需要我们也可以自己开发 hook，继承自 Baseoperator 或者 Basehook。例如 PostgresHook 会自动加载 conn 的连接字符串，连接目的数据库。具体连接数据库的字符串，可以在前台界面的 Admin > Connections 进行管理，然后在自己定义的 hook 里面有 get_connection 获得具体的连接字符串
-   数据库 operator，可以直接执行包含 sql 语句的文件。不同的数据库，需要安装对应的 provider 包，主要的作用是 hook 连接外部的数据库，管理连接池。
-   自定义的 operator，继承自 Baseoperator，在方法 execute 里定义主要的操作逻辑。自定义 Operator 的初始函数中，如果参数的赋值会需要用到模板变量，可以在类定义中通过 template_fields 来指定是哪个参数会需要用到模板变量。
-   在 UI 界面中展示自定义 Operatior 的样式，也可以在类中通过 ui_color 等属性进行定义。
-   其他 provider 包提供的 operator，例如连接 AWS 云服务器的 operator，亚马逊云提供的模型训练的接口等，当然也可以自己来开发这些 operator，继承 baseoperator。
-   SparkSubmitOperator 可以调用另外一个 spark 实例，从而把复杂的处理工作交给 spark 处理
-   自定义的 operator, 可以通过设置 setup.py，形成 package，方便其他人安装使用[https://github.com/audreyr/cookiecutter-pypackage](https://link.zhihu.com/?target=https%3A//github.com/audreyr/cookiecutter-pypackage)

```text
#自定义一个从PostgreSQL取数，转移数据到S3的operator
def execute(self, context): postgres_hook = PostgresHook(postgres_conn_id=self._postgres_conn_id) s3_hook = S3Hook(aws_conn_id=self._s3_conn_id) 
results = postgres_hook.get_records(self._query) s3_hook.load_string( 
Fetch records from the PostgreSQL database. 
 string_data=str(results),
 bucket_name=self._s3_bucket,
 key=self._s3_key,
) 
```

## 集群部署

airflow 具体运行的时候，有多种 executor 模式。这个 executor 可以是 SequentialExecutor（默认的），单机 LocalExecutor（最多并行 32 个任务），也可以是集群模式，例如 CeleryExecutor and the KubernetesExecutor 可以充分应用集群的计算能力。SequentialExecutor 和 LocalExecutor 差别不大，只是前者是单进程，后者是多进程。

### Celery

通过 pip install apache-airflow\[celery] 来安装包含 celery 依赖的 airflow。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/6655d1a7-6707-4f02-a17e-a5abb642a689.jpeg?raw=true)

Celery 是通过 Distributed Task Queue 分布式任务队列，来实现异步任务调度的工具。分布式决定了可以有多个 worker 的存在，队列表示其是异步操作，客户端提交一系列的任务，这些任务按照顺序分配给后端的任务执行单元。异步任务可以极大的提升系统的响应速度，无需等一个用户请求执行完毕，就可以接受第二个用户请求，

1. Producer（celery client）： 负责生产相关的 task，并发送到一个 broker 中。

2. broker： 作为一个 task 的路由器，将 task 分发到不同的接收者。例如 RabbitMQ 和 Redis 中间人的功能比较齐全。

3. Consumer（celery worker）：接收到 task 以后，负责根据 task 的信息结构，执行相关的功能。

4. "消息队列"（message Queue）可以有多个，不同的消息指定发送给不同的队列，通过 Exchange 通过指定 routiing_key 实现。每个 queue 指定对应的 worker 来完成任务。

Flower, 是一个监测 celery 运行情况的包，观察每个 celery work 执行任务的情况，参考

[https://airflow.apache.org/docs/apache-airflow/2.2.5/executor/celery.html](https://link.zhihu.com/?target=https%3A//airflow.apache.org/docs/apache-airflow/2.2.5/executor/celery.html)

在每台机器上都要安装 airflow, 并且其 DAGS_FOLDER 的内容是实时同步的（使用 Chef, Puppet, Ansible），参考

[https://blog.csdn.net/weixin_41347419/article/details/112423531](https://link.zhihu.com/?target=https%3A//blog.csdn.net/weixin_41347419/article/details/112423531)

### Kurbenetes

1998 年谷歌开始研发 Kubernetes(K8S)，并在 2014 年开源发展至今。K8S 是一个自动化的容器编排平台，用于自动部署，扩展和管理容器化应用程序的开源系统。K8S 的产生，主要是基于两个因素：庞大的软件系统架构逐步转向灵活解耦的微服务结构，软件的部署逐渐从本地向云上迁移。因为容器是运行微服务的最佳载体，同时云可以提供无线扩展的计算机资源，因此 K8S 的目标是管理云平台中多个主机上的容器化的应用，让部署容器化的应用简单并且高效, Kubernetes 提供了资源调度、部署管理、服务发现、扩容缩容、监控，维护等一整套功能。Kubernetes 有自动修复功能，当一个节点健康检查的功能，监测这个集群中所有的宿主机，当宿主机本身出现故障，或者软件出现故障的时候，这个节点健康检查会立即发现。Kubernetes 有水平伸缩的功能，监测业务上所承担的负载，如果这个业务本身的 CPU 利用率过高，或者响应时间过长，可以对这个业务进行扩容。

K8S 架构中机器分为 Master Node 和 Worker node。Control Plane 控制类型的组件安装在 master node 上，具体执行任务的组件安装在 Worker node 上。组件如下：

-   API Server：属于 control plane, 管理 K8S 的命令都是以 HTTP 请求的形式发送给 APISever，由它负责跟其他组件交互。
-   etcd：属于 control plane, 是一个分布式的一个存储系统，APIServer 通过读写 etcd 来记录和管理整个集群的运行状态。
-   Kublet: 属于 work node，它时刻跟 API sever 保持信息通讯，接受指令传递给 Container engine。
-   Kube-proxy: 属于 worker node，负责节点中 pods 之间的网络通讯功能，例如 K8S 中的 Service 就是基于 proxy 组件实现的
-   Scheduler：属于 control plane, 起到资源调度的功能。观察正在被调度的这个容器的情况，比如说任务所需要的 CPU 以及它所需要的 memory，然后在集群中找一台相对比较空闲的机器来进行一次 placement 放置，就是一次调度。
-   Controller-manager：属于 control plane，是控制器，完成对集群状态的一些管理，确保每个节点的实际工作状态和 etcd 里面记录的是一致的。
-   Container engine(docker): 属于 worker node，根据 kublet 的指令来完成容器部署。
-   Kubectl: 客户端，用户通过客户端来发送指令给集群。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/16ee38de-5295-4a40-a192-09b8209fbfd6.jpeg?raw=true)

重要的概念：

l **Pod** **是** **Kubernetes** **的一个最小调度以及资源单元。用户可以通过** **Kubernetes** **的** **Pod API** **生产一个** **Pod，让 Kubernetes** **对这个** **Pod** **进行调度，也就是把它放在某一个** **Kubernetes** **管理的节点上运行起来。一个** **Pod** **简单来说是对一组容器的抽象，它里面会包含一个或多个容器。** 

l **Volume** **就是卷的概念，它是用来管理** **Kubernetes** **存储的，是用来声明在** **Pod** **中的容器可以访问文件目录的，一个卷可以被挂载在** **Pod** **中一个或者多个容器的指定路径下面。** 

l **Deployment** **是在** **Pod** **这个抽象上更为上层的一个抽象，它可以定义一组** **Pod** **的副本数目、以及这个** **Pod** **的版本。一般大家用** **Deployment** **这个抽象来做应用的真正的管理，而** **Pod** **是组成** **Deployment** **最小的单元。** 

l **Service** **提供了一个或者多个** **Pod** **实例的稳定访问地址。** 

l **Namespace** **是用来做一个集群内部的逻辑隔离的，它包括鉴权、资源管理等。Kubernetes** **的每个资源，比如刚才讲的** **Pod、Deployment、Service** **都属于一个** **Namespace，同一个** **Namespace** **中的资源需要命名的唯一性，不同的 Namespace** **中的资源可以重名。** 

官方教程推荐使用 Helm chart 来在 K8S 集群上安装 airflow。Helm（[https://cloud.tencent.com/developer/article/1694640](https://link.zhihu.com/?target=https%3A//cloud.tencent.com/developer/article/1694640)）本质上是 K8S 配置文件的管理工具，配置文件的格式为 yaml，内容为应用部署到 K8S 集群时各个组件和对象的配置细节。因为配置文件的内容较为复杂，通过 helm 有助于借鉴其他团队的经验，基于此进一步补充自己所需的特有配置。A Helm Chart 指的是包含一组 K8s 资源集合的描述文件。Helm 的使用步骤如下

l 从 chart 仓库中获取 chart

l 使用者配置自己的 values 文件，根据自己的运行环境对 values 进行修改；

l 默认 values 文件和使用者 values 文件会进行一个 merge，形成最终的配置文件

l 使用最终的配置文件，渲染 chart 的 template，形成可以被 kubernetes 执行的 yaml

l 调用 kube apply 提交 yaml 到 kubernetes

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/66975d04-102d-4738-9562-9c9fbd4675e1.jpeg?raw=true)

在 Docker Desktop 默认是有单机版的 Kubernetes(minikube)，在 linux 上，需要安装 virtualbox 来使用 minikube，在 Windows 系统需要安装 Hyper-V hypervisor.

安装 docker desktop 以后需要开启 K8S

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/359a37a7-ee87-419a-8777-ffaaf2bdb9fc.jpeg?raw=true)

K8S 启动成功后，在 docker desktop 界面左下角显示绿色的图案，相当于一个集群已经建立起来了。生成环境，就要使用 kind 在真实的物理机群上建立 K8S 集群。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/d947fd04-d0c5-4e62-8c06-304f7939f52f.png?raw=true)

接下来在 windows 下安装 helm。首先用管理员身份运行 cmd，输入以下命令安装 Chocolatey，这是用来安装 helm 的工具。

```text
et-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

然后使用 chocolatey 安装 helm，如果速度很慢，考虑其他方法见[https://www.lsbin.com/9448.html](https://link.zhihu.com/?target=https%3A//www.lsbin.com/9448.html)

```text
choco install kubernetes-helm
```

通过系统管理员权限运行 CMD，运行以下命令。命令的作用是：把官方的配置加载到本地仓库，根据配置文件的信息，在集群上新建一个 namespace(资源空间)，在该空间下安装 airflow。命令也可以加上—debug 参考以便观察部署的日志信息。

```text
helm repo add apache-airflow https://airflow.apache.org
helm upgrade --install airflow apache-airflow/airflow --namespace airflow --create-namespace
```

安装成功后，会有如下的提示：

```text
our release is named airflow.
You can now access your dashboard(s) by executing the following command(s) and visiting the corresponding port at localhost in your browser:

Airflow Webserver:     kubectl port-forward svc/airflow-webserver 8080:8080 --namespace airflow
Default Webserver (Airflow UI) Login credentials:
    username: admin
    password: admin
Default Postgres connection credentials:
    username: postgres
    password: postgres
    port: 5432

```

按照提示，运行以下命令就可以在本地机访问 web 管理界面

```text
kubectl port-forward svc/airflow-webserver 8080:8080 --namespace airflow
```

运行以下命令，检查集群各个组件和服务的运行情况

```text
kubectl get pods -n airflow
kubectl get services -n airflow
```

如果要卸载 airflow 在集群上的部署，运行如下命令

```text
helm delete airflow --namespace airflow
```

在 K8s 环境里，DAG 文件同步的方法有如下，我们采用其中第一种方法

1）容器初始化的时候使用 git 来同步文件

2）在 pods 之间通过 PersistentVolume 来共享

3）每次把新更新的代码做成镜像，在 dockerfile 里把开发者目录最新的代码拷贝到 DAGs 文件夹下

基于官方的 helm chart，我们可以进一步自定义集群的配置。下载官方的 chart，注意下面第二个命令的 version 指的是 char 的 version

```text
helm search repo apache-airflow/airflow -l
helm show values apache-airflow/airflow --version 1.6.0 > airflow_values.yaml
helm show values apache-airflow/airflow > values.yaml
```

在下载的 yaml 文件中，行数很多但当前只关注以下的配置

\- defaultAirflowRepository、defaultAirflowTag、airflowVersion 约定了默认的镜像，如果需要集群加载我们自己的镜像，需要在这里设置

\- 符号~，指的是采用默认值; IfNotPresent 指尽量使用本地镜像

\- executor: "CeleryExecutor" 改成 KubernetesExecutor

\- gitSync 的 enabled 改为 True，我们将使用 git 来同步 DAG 文件

\- subPath 的值，要根据放在 github 里面的具体文件路径来设置，例如默认是 tests/dags

配置 gitSync 的部分

```text
gitSync:
    enabled: true
    # git repo clone url
    # ssh examples ssh://git@github.com/apache/airflow.git
    # git@github.com:apache/airflow.git
    # https example: https://github.com/apache/airflow.git
    repo: ssh://git@github.com/yzengnash/learngit.git
    branch: main
    rev: HEAD
depth: 1
    # the number of consecutive failures allowed before aborting
    maxFailures: 0
    # subpath within the repo where dags are located
    # should be "" if dags are at repo root
    subPath: "tests/dags"
--------------省略很多行------------------
    # If you are using an ssh clone url, you can load
    # the ssh private key to a k8s secret like the one below
    #   ---
    #   apiVersion: v1
    #   kind: Secret
    #   metadata:
    #     name: airflow-ssh-secret
    #   data:
    #     # key needs to be gitSshKey
    #     gitSshKey: <base64_encoded_data>
    # and specify the name of the secret below
    sshKeySecret: airflow-ssh-secret
```

在集群上建立一个 secret，即把 github 上的个人秘钥转成 secret 的形式存储。（具体 git 秘钥生成过程见后面章节）

```text
kubectl create secret generic airflow-ssh-git-secret --from-file=gitSshKey=$Home/.ssh/id_rsa -n airflow
```

运行以下命令看看是否 secret 新建成功

```text
kubectl get secrets -n airflow
```

运行以下命令更新 cluster 的安装

```text
helm upgrade --install airflow apache-airflow/airflow -n airflow -f values.yaml --debug
```

查看运行的 pod

```text
kubectl describe pod airflow-scheduler-756d8c548c-67d99 -n airflow
```

通过查看日志，发现错误信息 “Failed to pull image "[http://k8s.gcr.io/git-sync/git-sync:v3.4.0](https://link.zhihu.com/?target=http%3A//k8s.gcr.io/git-sync/git-sync%3Av3.4.0)": rpc”，主要原因是由于国内网络防火墙问题导致无法正常拉取。我们使用阿里的国内镜像，参考[https://zhuanlan.zhihu.com/p/429685829](https://zhuanlan.zhihu.com/p/429685829)

在 github 里面新建一个 repository，然后增加一个 dockerfile，内容是需要拉取的 image，例如 FROM [http://k8s.gcr.io/git-sync/git-sync:v3.4.0](https://link.zhihu.com/?target=http%3A//k8s.gcr.io/git-sync/git-sync%3Av3.4.0)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/5469b1ee-b02b-487b-8576-10e54706c3c2.jpeg?raw=true)

登录阿里云，进入容器镜像服务控制台，创建个人实例，设置过 Registry 登录密码。创建命名空间

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/312f5560-00a6-4b1a-be04-8637c697298c.jpeg?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/a4bd082b-bc3c-4733-83b1-1b126ca4c14f.jpeg?raw=true)

选择上面创建的 github 镜像仓库，勾选海外机器构建

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/94778428-291e-4f3c-8f73-e627f0e9586f.jpeg?raw=true)

进入镜像仓库，添加规则，然后点击 立即构建

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/057e78dd-b08e-47dc-b8e3-27c4f35944ce.jpeg?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/3666f0cc-ecd4-4d38-a178-b3cb729445e8.png?raw=true)

然后进行登录（注意：容器登录镜像服务实例的密码和阿里平台的密码不是同一个）

```text
docker login registry.cn-hangzhou.aliyuncs.com
```

把刚才的镜像拉下来

```text
docker pull registry.cn-hangzhou.aliyuncs.com/nash_image/image_repo:v1
```

重新命名

```text
docker tag registry.cn-hangzhou.aliyuncs.com/nash_image/image_repo:v1 k8s.gcr.io/git-sync/git-sync:v3.4.0
```

再次根据最新的配置文件安装，就成功了

```text
helm upgrade --install airflow apache-airflow/airflow -n airflow -f values.yaml --debug
```

## Git 安装配置

CVS 及 SVN 都是集中式的版本控制系统, Git 是分布式版本控制系统。分布式版本控制系统根本没有 “中央服务器”，每个人的电脑上都是一个完整的版本库，只是在联网的时候互相同步一下最新的代码即可，别的机器可以“克隆” 代码仓库，而且每台机器的版本库其实都是一样的，并没有主次之分。因此分布式的安全性更高。实际上常用的方法是，找一台电脑充当服务器的角色，每天 24 小时开机，其他每个人都从这个 “服务器” 仓库克隆一份到自己的电脑上，并且各自把各自的提交推送到服务器仓库里，也从服务器仓库中拉取别人的提交。更多操作教程见[https://www.liaoxuefeng.com/wiki/896043488029600](https://link.zhihu.com/?target=https%3A//www.liaoxuefeng.com/wiki/896043488029600)

在 windows 安装 Git, 使用[https://git-scm.com/downloads](https://link.zhihu.com/?target=https%3A//git-scm.com/downloads)

新建一个文件夹，然后在该文件夹右键 Git bash here，输入一下命令来建立用户名和密码，global 参数会让本机的所有仓库都使用这个配置

```text
git config --global “user.name” 
git config --global ”user.email" 
```

可以运行以下命令看所有配置

把该目录变成一个仓库

用命令 git add 告诉 Git，把文件添加到仓库

用命令 git commit 告诉 Git，把文件提交到仓库：

```text
git commit -m "wrote a readme file"
```

查看当前仓库的状态

恢复到上一个版本

注册一个 GitHub 账号，就可以免费获得 Git 远程仓库。也可以自建自己的 Git 服务器

创建 SSH Key，后续配置远程仓库的时候需要，作为身份认证

```text
$ ssh-keygen -t rsa -C "youremail@example.com"
```

在用户主目录里找到. ssh 目录，里面有 id_rsa 和 id_rsa.pub 两个文件，这两个就是 SSH Key 的秘钥对，id_rsa 是私钥，不能泄露出去，id_rsa.pub 是公钥，可以放心地告诉任何人。打开 id_rsa.pub，复制里面的 key（所有文字）。进入到 github 页面，菜单 account->setting->SSH and GPG keys，点 “Add SSH Key”，填上任意 Title，在 Key 文本框里粘贴 id_rsa.pub 文件的内容

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-4-3%2019-04-19/a7f07ab0-bd9f-4794-945e-a8307e94e700.jpeg?raw=true)

登陆 GitHub，然后，在右上角找到 “Create a new repo” 按钮，创建一个新的仓库，例如 learngit。然后在本地仓库运行以下命令创建一个叫做 origin 的 remote，并且完成对远程仓库的推送。其中：https 以后的内容是有 Github 新建仓库后生成的地址。

```text
git remote add origin https://github.com/yzengnash/learngit.git
git branch -M main
git push -u origin main
```

如果报错 Connection was reset, errno 10054，就运行以下命令设置解除 SSL 验证。

```text
git config --global http.sslVerify "false"
```

然后再进行 push 的时候，会在浏览器中弹出登录 github 的窗口，正常登录后选择 authorize gitCredentialManager, 后续提交就会成功了。
