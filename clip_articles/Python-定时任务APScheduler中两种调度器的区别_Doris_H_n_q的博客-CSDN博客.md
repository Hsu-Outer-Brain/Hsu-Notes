# Python-定时任务APScheduler中两种调度器的区别_Doris_H_n_q的博客-CSDN博客
[Python - 定时任务 APScheduler 中两种调度器的区别\_Doris_H_n_q 的博客 - CSDN 博客](https://blog.csdn.net/Dorisi_H_n_q/article/details/107658508) 

### **概述**

两种调度器`BackgroundScheduler`和`BlockingScheduler`的区别

### **举例说明**

`APScheduler`是 python 的一个定时任务调度框架，能实现类似 linux 下`crontab`类型的任务，使用起来比较方便。它提供基于固定时间间隔、日期以及`crontab`配置类似的任务调度，并可以持久化任务，或将任务以 daemon 方式运行。

```null
from apscheduler.schedulers.blocking import BlockingScheduler    sched = BlockingScheduler(timezone='MST')    sched.add_job(job, 'interval', id='3_second_job', seconds=3)
```

它能实现每隔 3s 就调度`job()`运行一次，所以程序每隔 3s 就输出’job 3s’。通过修改`add_job()`的参数`seconds`，就可以改变任务调度的间隔时间。

### **`BlockingScheduler`与`BackgroundScheduler`区别**

`APScheduler`中有很多种不同类型的调度器，`BlockingScheduler`与`BackgroundScheduler`是其中最常用的两种调度器。那他们之间有什么区别呢？ 简单来说，区别主要在于`BlockingScheduler`会阻塞主线程的运行，而`BackgroundScheduler`不会阻塞。所以，我们在不同的情况下，选择不同的调度器：

-   `BlockingScheduler`: 调用 start 函数后会阻塞当前线程。当调度器是你应用中唯一要运行的东西时（如上例）使用。
-   `BackgroundScheduler`: 调用 start 后主线程不会阻塞。当你不运行任何其他框架时使用，并希望调度器在你应用的后台执行。

下面用两个例子来更直观的说明两者的区别。

-   `BlockingScheduler`的真实例子

```null
from apscheduler.schedulers.blocking import BlockingScheduler    sched = BlockingScheduler(timezone='MST')    sched.add_job(job, 'interval', id='3_second_job', seconds=3)
```

可见，`BlockingScheduler`调用 start 函数后会阻塞当前线程，导致主程序中 while 循环不会被执行到。

-   `BackgroundScheduler`的真实例子

```null
from apscheduler.schedulers.background import BackgroundScheduler    sched = BackgroundScheduler(timezone='MST')    sched.add_job(job, 'interval', id='3_second_job', seconds=3)
```

可见，`BackgroundScheduler`调用 start 函数后并不会阻塞当前线程，所以可以继续执行主程序中 while 循环的逻辑。

### **job 执行时间大于定时**

```null
from apscheduler.schedulers.background import BackgroundScheduler    sched = BackgroundScheduler(timezone='MST')    sched.add_job(job, 'interval', id='3_second_job', seconds=3)Execution of job "job (trigger: interval[0:00:03], next run at: 2018-05-07 02:44:29 MST)" skipped: maximum number of running instances reached (1)
```

可见，3s 时间到达后，并不会 “重新启动一个 job 线程”，而是会跳过该次调度，等到下一个周期（再等待 3s），又重新调度 job()。

为了能让多个 job() 同时运行，我们也可以配置调度器的参数`max_instances`，如下例，我们允许 2 个 job() 同时运行：

```null
from apscheduler.schedulers.background import BackgroundScheduler    job_defaults = { 'max_instances': 2 }    sched = BackgroundScheduler(timezone='MST', job_defaults=job_defaults)    sched.add_job(job, 'interval', id='3_second_job', seconds=3)
```

### **每个 job 是被线程还是进程调度的？**

```null
from apscheduler.schedulers.background import BackgroundSchedulerprint('job thread_id-{0}, process_id-{1}'.format(threading.get_ident(), os.getpid()))    job_defaults = { 'max_instances': 20 }    sched = BackgroundScheduler(timezone='MST', job_defaults=job_defaults)    sched.add_job(job, 'interval', id='3_second_job', seconds=3)job thread_id-10644, process_id-8872job thread_id-3024, process_id-8872job thread_id-6728, process_id-8872job thread_id-11716, process_id-8872
```

可见，每个 job() 的进程 ID 都相同，但线程 ID 不同。所以，job() 最终是以线程的方式被调度执行。

参考：[https://blog.csdn.net/ybdesire/article/details/82228840](https://blog.csdn.net/ybdesire/article/details/82228840)

详细：[https://www.cnblogs.com/brithToSpring/p/13374911.html](https://www.cnblogs.com/brithToSpring/p/13374911.html)
