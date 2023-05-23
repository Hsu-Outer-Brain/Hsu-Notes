# Python多线程捕获子线程的异常，并退出主进程。 - 就是想学习 - 博客园
[Python 多线程捕获子线程的异常，并退出主进程。 - 就是想学习 - 博客园](https://www.cnblogs.com/sidianok/p/15726845.html) 

 自己在项目的开发中，一般能避免在单个进程中使用多线程就尽量把每个线程包装成独立的进程执行，通过 socket 或者一些中间件比如 redis 进行通讯，工作，协调。

但有时候必须涉及到多线程操作，而且碰到的情况中，多个线程必须协调全部正常工作才能执行逻辑，但子线程有着自己的栈区，报错了并不影响其它的线程，导致整个进程无法退出。

我当时想到的有两种思路，一种是多个线程间进行通讯或者一个全局变量的标记，当报错的时候，就修改这个标记，所有的子线程定时去查询这个标记，但感觉这个思路的拓展性太差，而且每个子线程需要主动定期查询或者通讯，太麻烦了。

后面一种就是我准备上代码的思路， 将所有的子线程设计成守护线程，主线程循环查询子线程的状态值，当发现任意的子线程状态异常，获取该子线程的异常对象，并上浮，退出主线程，导致所有的子线程退出。

```

```

2022 年 2 月 22 日更新。

由于项目的改版，重新需要用到多线程，个人定制版的 Thread

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-23%2016-31-02/bd40efe7-6567-487b-be47-3b905a2dc6b9.gif?raw=true)

class ExcThread(threading.Thread): def \_\_init\_\_(self, target=None, \*args, name=None, \*\*kwargs): if name is None:
            name = target.\_\_name\_\_ super(ExcThread, self).\_\_init\_\_(None, target, name, args, kwargs)
        self.exit_code = None
        self.exception = None
        self.exc_traceback = ''

        # 默认为守护线程

 self.setDaemon(True) def run(self): try:
            self.\_run() except Exception as e:
            self.exception = e
            self.exit_code = 1 self.exc_traceback = traceback.format_exc() else:
            self.exit_code = 0 def \_run(self): try:
            self.\_target(\*self.\_args, \*\*self.\_kwargs) except Exception as e: raise e

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-23%2016-31-02/9371a35e-f066-4ea4-9b46-379e80d07f31.gif?raw=true)

主线程巡查各子线状态

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-23%2016-31-02/aa4fba80-f53f-4d25-a4ef-d0109d9e07e0.gif?raw=true)

    # 主线程检测子线程运行，接受到子线程死亡信号，上浮子线程错误信息
    def \_check\_child\_thread\_status(self): while True: for task in self.\_thread\_task_list.copy(): # 已经完成的任务删除
                if not task.is_alive():
                    self.\_thread\_task_list.remove(task) if task.exit_code: print(f'{datetime.datetime.now()} task: {task.name} raise exception.', file=sys.stderr) print(f'{datetime.datetime.now()} task: {task.name} raise exception.') try: raise task.exception except:
                        l = logger\_sd(stream\_display=False, file\_out=False, mail\_send=True)
                        l.critical(f'*-smt-*\\n {traceback.format_exc()} \\n *-smt-*') print(traceback.format_exc()) raise time.sleep(.1)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-23%2016-31-02/d936234d-073c-4a3f-a141-2aa9f7c26586.gif?raw=true)
