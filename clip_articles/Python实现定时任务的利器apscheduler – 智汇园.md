# Python实现定时任务的利器apscheduler – 智汇园
[Python 实现定时任务的利器 apscheduler – 智汇园](https://www.dengqihua.net/index.php/2022/10/12/python%E5%AE%9E%E7%8E%B0%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1%E7%9A%84%E5%88%A9%E5%99%A8apscheduler/) 

## 前言

之前有介绍了用 Linux crontab 的方式来实现定时任务，这是使用 Linux 内置模块来实现的。而在 Python 中，还可以用第三方包来管理定时任务，比如 celery、apscheduler。相对来说 apscheduler 使用起来更简单一些，这里来介绍一下 apscheduler 的使用方法。

首先安装起来很简单，运行`pip install apscheduler`即可。

## 初识 apscheduler

来个简单的例子看看 apscheduler 是如何使用的。

    #encoding:utf-8

    from apscheduler.schedulers.blocking import BlockingScheduler
    import datetime

    def sch_test():
        now = datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        print('时间：{}, 测试apscheduler'.format(now))

    task = BlockingScheduler()
    task.add_job(func=sch_test, trigger='cron', second='*/10')
    task.start()
    复制代码

上述例子很简单，我们首先要定义一个 apscheduler 的对象，然后 add_job 添加任务，最后 start 开启任务就行了。

例子是每隔 10 秒运行一次 sch_test 任务，运行结果如下：

    时间：2022-10-08 15:16:30, 测试apscheduler
    时间：2022-10-08 15:16:40, 测试apscheduler
    时间：2022-10-08 15:16:50, 测试apscheduler
    时间：2022-10-08 15:17:00, 测试apscheduler
    复制代码

如果我们要在执行任务函数时携带参数，只要在 add_job 函数中添加 args 就行，比如`task.add_job(func=sch_test, args=('a'), trigger='cron', second='*/10')`。

## apscheduler 有哪些模块

上面例子中我们初步了解到如何使用 apschedulerl 了，接下来需要知道 apscheduler 的设计框架。apscheduler 有四个主要模块，分别是：**触发器 triggers**、**任务存储器 job_stores**、**执行器 executors**、**调度器 schedulers**。

### 1. 触发器 triggers:

触发器指的是任务指定的触发方式，例子中我们用的是 “cron” 方式。我们可以选择 cron、date、interval 中的一个。

-   cron 表示的是定时任务，类似 linux crontab，在指定的时间触发。

可用参数如下：

| 参数          | 释义              |
| ----------- | --------------- |
| year        | 年份（4 位数，如 2022） |
| month       | 月份（1-12）        |
| day         | 一个月的第几天（1-31）   |
| week        | 一年的第几周（1-53）    |
| day_of_week | 一星期的第几天（0-6）    |
| hour        | 小时              |
| minute      | 分钟              |
| second      | 秒               |
| start_date  | 开始时间            |
| end_date    | 结束时间            |
| timezone    | 时区              |
| jitter      | 触发的误差时间         |

除此之外，我们还可用表达式类型去设置 cron。比如常用的有：

| 表达式   | 释义             |
| ----- | -------------- |
| \*    | 每个值都触发         |
| \*/n  | 每隔 n 触发一次      |
| a-b   | 在 a-b 内任何时间都触发 |
| a,b,c | 分别在 a,b,c 时间触发 |

使用方法示例，在每天 7 点 20 分执行一次：

`task.add_job(func=sch_test, args=('定时任务',), trigger='cron',`

hour`='7', minute='20')`

-   date 表示具体到某个时间的一次性任务；

使用方法示例：

    # 使用run_date指定运行时间
    task.add_job(func='sch_test', trigger='date', run_date=datetime.datetime(2022 ,10 , 8, 16, 1, 30))
    # 或者用next_run_time
    task.add_job(func=sch_test,trigger='date', next_run_time=datetime.datetime.now() + datetime.timedelta(seconds=3))
    复制代码

-   interval 表示的是循环任务，指定一个间隔时间，每过间隔时间执行一次。

interval 可设置如下的参数：

| 参数         | 释义        |
| ---------- | --------- |
| weeks      | 周         |
| days       | 一个月的第几天   |
| hours      | 小时        |
| minutes    | 分钟        |
| seconds    | 秒         |
| start_date | 间隔触发的开始时间 |
| end_date   | 间隔触发的结束时间 |
| jitter     | 触发的时间误差   |

使用方法示例，每隔 3 秒执行一次 sch_test 任务：

`task.add_job(func=sch_test, args=('循环任务',), trigger='interval', seconds=3)`。

来个例子把 3 种触发器都使用一遍：

    # encoding:utf-8
    from apscheduler.schedulers.blocking import BlockingScheduler
    import datetime

    def sch_test(job_type):
        now = datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        print('时间：{}, {}测试apscheduler'.format(now, job_type))

    task = BlockingScheduler()
    task.add_job(func=sch_test, args=('一次性任务',),trigger='date', next_run_time=datetime.datetime.now() + datetime.timedelta(seconds=3))
    task.add_job(func=sch_test, args=('定时任务',), trigger='cron', second='*/5')
    task.add_job(func=sch_test, args=('循环任务',), trigger='interval', seconds=3)
    task.start()
    复制代码

打印部分结果：

    时间：2022-10-08 15:45:49, 一次性任务测试apscheduler
    时间：2022-10-08 15:45:49, 循环任务测试apscheduler
    时间：2022-10-08 15:45:50, 定时任务测试apscheduler
    时间：2022-10-08 15:45:52, 循环任务测试apscheduler
    时间：2022-10-08 15:45:55, 定时任务测试apscheduler
    时间：2022-10-08 15:45:55, 循环任务测试apscheduler
    时间：2022-10-08 15:45:58, 循环任务测试apscheduler
    复制代码

通过代码示例和结果展示，我们可清晰的知道不同触发器的使用区别。

### 2. 任务存储器 job_stores

顾名思义，任务存储器是存储任务的地方，默认都是存储在内存中。我们也可自定义存储方式，比如将任务存到 mysql 中。这里有以下几种选择：

| 存储器类型              | 释义                          |
| ------------------ | --------------------------- |
| MemoryJobStore     | 任务存储在内存中                    |
| SQLAlchemyJobStore | 使用 sqlalchemy 作为存储方式，存储在数据库 |
| MongoDBJobStore    | 存储在 mongodb 中               |
| RedisJobStore      | 存储在 redis 中                 |

通常默认存储在内存即可，但若程序故障重启的话，会重新拉取任务运行了，如果你对任务的执行要求高，那么可以选择其他的存储器。

使用 SQLAlchemyJobStore 存储器示例：

    from apscheduler.schedulers.blocking import BlockingScheduler

    def sch_test(job_type):
        now = datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        print('时间：{}, {}测试apscheduler'.format(now, job_type))

    sched = BlockingScheduler()
    # 使用mysql存储任务
    sql_url = 'mysql+pymysql://root:root@localhost:3306/db_name?charset=utf8'
    sched.add_jobstore('sqlalchemy',url=sql_url)
    # 添加任务
    sched.add_job(func=sch_test, args=('定时任务',), trigger='cron', second='*/5')
    sched.start()
    复制代码

### 3. 执行器 executors

执行器的功能就是将任务放到线程池或进程池中运行。有以下几种选择：

| 执行器类型               | 释义            |
| ------------------- | ------------- |
| ThreadPoolExecutor  | 线程池执行器        |
| ProcessPoolExecutor | 进程池执行器        |
| GeventExecutor      | Gevent 程序执行器  |
| TornadoExecutor     | Tornado 程序执行器 |
| TwistedExecutor     | Twisted 程序执行器 |
| AsyncIOExecutor     | asyncio 程序执行器 |

默认是 ThreadPoolExecutor， 常用的也就是第线程和进程池执行器。如果应用是 CPU 密集型操作，可用 ProcessPoolExecutor 来执行。

### 4. **调度器 schedulers**

调度器属于 apscheduler 的核心，它扮演着统筹整个 apscheduler 系统的角色，存储器、执行器、触发器在它的调度下正常运行。调度器有以下几个：

| 调度器                 | 使用场景                            |
| ------------------- | ------------------------------- |
| BlockingScheduler   | 当调度器是你应用中唯一要运行的，start 开启后会阻塞    |
| BackgroundScheduler | 适用于调度程序在应用程序的后台运行，start 开启后不会阻塞 |
| AsyncIOScheduler    | 当程序使用了 asyncio 的异步框架时使用。        |
| GeventScheduler     | 当程序用了 Tornado 的时候用              |
| TwistedScheduler    | 当程序用了 Twisted 的时候用              |
| QtScheduler         | 当应用是 QT 应用的时候用                  |

不是特定场景下，我们最常用的是 BlockingScheduler 调度器。

### 异常监听

定时任务在运行时，若出现错误，需要设置监听机制，我们通常结合 logging 模块记录错误信息。

使用示例：

    from apscheduler.schedulers.blocking import BlockingScheduler
    import datetime
    from apscheduler.events import EVENT_JOB_EXECUTED , EVENT_JOB_ERROR
    import logging

    # logging日志配置打印格式及保存位置
    logging.basicConfig(level=logging.INFO,
                        format='%(asctime)s %(filename)s[line:%(lineno)d] %(levelname)s %(message)s',
                        datefmt='%Y-%m-%d %H:%M:%S',
                        filename='sche.log',
                        filemode='a')

    def log_listen(event):
    	if event.exception :
    		print ( '任务出错，报错信息：{}'.format(event.exception))
    	else:
    		print ( '任务正常运行...' )

    def sch_test(job_type):
        now = datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        print('时间：{}, {}测试apscheduler'.format(now, job_type))
        print(1/0)

    sched = BlockingScheduler()
    # 使用mysql存储任务
    sql_url = 'mysql+pymysql://root:root@localhost:3306/db?charset=utf8'
    sched.add_jobstore('sqlalchemy',url=sql_url)
    # 添加任务
    sched.add_job(func=sch_test, args=('定时任务',), trigger='cron', second='*/5')

    # 配置任务执行完成及错误时的监听
    sched.add_listener(log_listen, EVENT_JOB_EXECUTED | EVENT_JOB_ERROR)
    # 配置日志监听
    sched._logger = logging

    sched.start()
    复制代码

## apscheduler 的封装使用

上面介绍了 apscheduler 框架的主要模块，我们基本能掌握怎样使用 apscheduler 了。下面就来封装一下 apscheduler 吧，以后要用直接在这份代码上修改就行了。

    from apscheduler.schedulers.blocking import BlockingScheduler
    from apscheduler.executors.pool import ThreadPoolExecutor, ProcessPoolExecutor
    from apscheduler.events import EVENT_JOB_EXECUTED , EVENT_JOB_ERROR
    import logging
    import logging.handlers
    import os
    import datetime


    class LoggerUtils():
        def init_logger(self, logger_name):
            # 日志格式
            formatter = logging.Formatter('%(asctime)s %(filename)s[line:%(lineno)d] %(levelname)s %(message)s')
            log_obj = logging.getLogger(logger_name)
            log_obj.setLevel(logging.INFO)
            # 设置log存储位置
            path = '/data/logs/'
            filename = '{}{}.log'.format(path, logger_name)
            if not os.path.exists(path):
                os.makedirs(path)
            # 设置日志按照时间分割
            timeHandler = logging.handlers.TimedRotatingFileHandler(
               filename,
               when='D',  # 按照什么维度切割， S:秒，M：分，H:小时，D:天，W:周
               interval=1, # 多少天切割一次
               backupCount=10  # 保留几天
            )
            timeHandler.setLevel(logging.INFO)
            timeHandler.setFormatter(formatter)
            log_obj.addHandler(timeHandler)
            return log_obj


    class Scheduler(LoggerUtils):
        def __init__(self):
            # 执行器设置
            executors = {
                'default': ThreadPoolExecutor(10),  # 设置一个名为“default”的ThreadPoolExecutor，其worker值为10
                'processpool': ProcessPoolExecutor(5)  # 设置一个名为“processpool”的ProcessPoolExecutor，其worker值为5
            }
            self.scheduler = BlockingScheduler(timezone="Asia/Shanghai", executors=executors)
            # 存储器设置
            # 这里使用sqlalchemy存储器,将任务存储在mysql
            sql_url = 'mysql+pymysql://root:root@localhost:3306/db?charset=utf8'
            self.scheduler.add_jobstore('sqlalchemy',url=sql_url)

            def log_listen(event):
                if event.exception:
                    # 日志记录
                    self.scheduler._logger.error(event.traceback)
        
            # 配置任务执行完成及错误时的监听
            self.scheduler.add_listener(log_listen, EVENT_JOB_EXECUTED | EVENT_JOB_ERROR)
            # 配置日志监听
            self.scheduler._logger = self.init_logger('sche_test')

        def add_job(self, *args, **kwargs):
            """添加任务"""
            self.scheduler.add_job(*args, **kwargs)

        def start(self):
            """开启任务"""
            self.scheduler.start()

    # 测试任务
    def sch_test(job_type):
        now = datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        print('时间：{}, {}测试apscheduler'.format(now, job_type))
        print(1/0)


    # 添加任务,开启任务
    sched = Scheduler()
    # 添加任务
    sched.add_job(func=sch_test, args=('定时任务',), trigger='cron', second='*/5')
    # 开启任务
    sched.start()
    复制代码

## 小结

这篇文章介绍了 Python 实现定时任务的又一利器 apscheduler，通过简单例子及 apscheduler 框架的主要模块分解，我们可以根据实际需求配置好模块信息，再结合 logging 模块，我们可以实时监控到定时任务的运行情况。话说大家是如何来实现定时任务的，欢迎评论区讨论。
