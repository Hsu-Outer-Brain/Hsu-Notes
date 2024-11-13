# python中父子进程_父子进程os.getpgid不一样-CSDN博客
[python 中父子进程\_父子进程 os.getpgid 不一样 - CSDN 博客](https://blog.csdn.net/wenzhou1219/article/details/81320622) 

 最近在使用 python 中的 multiprocessing 模块时遇到一些问题，很多人应该遇到相同问题，简单研究下，供有需要的参考。

首先，要明白 multiprocessing 的出现很大程度是为了解决 python GIL 锁带来的[多线程](https://so.csdn.net/so/search?q=%E5%A4%9A%E7%BA%BF%E7%A8%8B&spm=1001.2101.3001.7020)低效问题，其次，注意 Windows 上和 Linux 上的进程、线程行为不一致。

那么我们常遇到的问题如下：

1\. 父进程开新的子进程完成任务，父进程关闭时，必须关闭子进程

2\. 父进程被强制关闭时，子进程也必须关闭

3\. 子进程被强制关闭时，父进程也必须关闭

4\. 父子进程没必然联系，关闭互不影响

下面就从这四个问题出发，讨论每种问题的处理方式和在不同系统下的差别。

## 1. 父进程关闭时，必须关闭子进程

### a. 常规情况

python 10.py 执行如下代码

```null
from multiprocessing import Processprint('Run child process %s (%s)...' % (name, os.getpid()))print('Exit child process %s (%s)...' % (name, os.getpid()))if __name__ == '__main__':print('Parent process %s start.' % os.getpid())        p = Process(target=run_proc, args=('test %s' %i,))print('Parent process end.')
```

在 Windows 上和 Linux 上均输出

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-11-13%2011-14-53/55e90a68-6302-4bd2-afba-8585d566ac0b.png?raw=true)

可以看到，显示 Parent process end. 后还需等待一段时间子进程关闭后父进程才能退出，此时 ps -ux 或任务管理器查看无子进程，默认父进程关闭时会等待子进程运行完毕才退出。

### b. 子进程设为守护进程

我们知道 multiprocessing.Process 有一个参数设置进程为守护进程 p.daemon = True，此时子进程可以独立运行。

python 11.py 执行如下代码:

```null
from multiprocessing import Processprint('Run child process %s (%s)...' % (name, os.getpid()))print('Exit child process %s (%s)...' % (name, os.getpid()))if __name__ == '__main__':print('Parent process %s start.' % os.getpid())        p = Process(target=run_proc, args=('test %s' %i,))print('Parent process end.')
```

在 Windows 和 Linux 上执行均输出：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-11-13%2011-14-53/2cbfe756-36b0-4695-b1ea-ee3c928d8d64.png?raw=true)

显示 Parent process end. 后主进程直接退出，此时 ps -ux 或任务管理器查看无子进程，即父进程关闭时强制杀死了子进程。

为了取得和之前一样的效果，可以显示指定等待子进程执行完成后再退出，即指定 p.join() 等待子进程执行完成，代码如下，此时和之前运行效果一样。

```null
from multiprocessing import Processprint('Run child process %s (%s)...' % (name, os.getpid()))print('Exit child process %s (%s)...' % (name, os.getpid()))if __name__ == '__main__':print('Parent process %s start.' % os.getpid())        p = Process(target=run_proc, args=('test %s' %i,))print('Parent process end.')
```

### c. 子进程响应关闭消息

既然父进程关闭时会强制杀死守护子进程，那么不妨加上进程关闭消息响应，实现优雅关闭，执行 python 12.py 如下：

```null
from multiprocessing import Processdef term(sig_num, addtion):print '%d to term child process' % os.getpid()print('Run child process %s (%s)...' % (name, os.getpid()))print('Process %s' % (os.getpid()))print('Exit child process %s (%s)...' % (name, os.getpid()))if __name__ == '__main__':    signal.signal(signal.SIGTERM, term)print('Parent process %s start.' % os.getpid())        p = Process(target=run_proc, args=('test %s' %i,))print('Parent process end.')
```

此时 Windows 上显示如下：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-11-13%2011-14-53/7c844d19-175d-482c-aaf2-147e5a956383.png?raw=true)

即 windows 还是强制关闭，父进程和子进程都关闭。

Linux 显示如下：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-11-13%2011-14-53/c98ae177-1a9d-41fe-b385-6da18d17118f.png?raw=true)

Linux 上显示 Parent process end. 后主进程通知子进程关闭，子进程响应了了这一消息，但是主循环仍然在持续输出，此时子进程和父进程都还存在。

 这就涉及到 Windows 和 Linux 上进程结束的特点：Windows 上没有一个很好的机制来通知进程关闭，这里通过 TerminateProcess 来强杀进程；Linux 上可以给进程发送 signal.SIGTERM 来通知进程关闭，如果进程 signal.signal 注册了这个消息的响应，那么接到通知后进程如何处理退出是自己的事了，甚至可以像本示例不退出，如果不注册这个消息的响应则像上个示例一样进程被强杀。

如上逻辑，可以查看对应的源码如下：

multiprocessing.Process 类中**\_bootstrap 函数**初始化运行，可以看到这里结束时都调用了 util.\_exit_function() 函数

查看 multiprocessing.util 中**util.\_exit_function() 函数**如下

```null
if current_process() is not None:for p in active_children():            info('calling terminate() for daemon %s', p.name)for p in active_children():        info('calling join() for process %s', p.name)
```

可以看到，结束时会遍历当前进程的子进程，先对每个子进程发送 terminate 通知，然后等待剩余存活子进程执行完毕。多说一点，注意这里的逻辑，也就是如果子进程（包括守护进程）不全部退出，那么父进程是不退出的。

在 multiprocessing.forking 中查看**teminate 代码**如下，可以看到 Windows 下是强制结束进程，Linux 下是发送通知信号。

```null
if sys.platform != 'win32':if self.returncode is None:                os.kill(self.pid, signal.SIGTERM)if self.wait(timeout=0.1) is None:if self.returncode is None:                _subprocess.TerminateProcess(int(self._handle), TERMINATE)if self.wait(timeout=0.1) is None:
```

## 2. 父进程被强制关闭时，关闭子进程

Windows 中，执行 python 10.py 或 python 11.py，任务管理器中结束父进程，查看子进程是否关闭；linux 中执行 python 10.py & 或 python 11.py &，kill 主进程 pid 结束父进程，ps -ux | grep python 查看子进程是否关闭。

可以看到在 Windows 和 Linux 中强制关闭父进程，子进程仍然存活，被称为孤立进程。

要想主进程被强制关闭时，也关闭子进程，应该怎么做呢。前面说过 Windows 上结束进程没有通知机制，因此暂时没有简单办法处理；在 linux 上 kill 会给被关闭进程发送通知，既然这样，我们响应这个消息并给子进程发送关闭消息即可，代码如下：

```null
from multiprocessing import Processdef term(sig_num, addtion):print '%d to term child process' % os.getpid()print 'process %d-%d terminate' % (os.getpid(), p.pid)print('Run child process %s (%s)...' % (name, os.getpid()))print('Exit child process %s (%s)...' % (name, os.getpid()))if __name__ == '__main__':print('Parent process %s start.' % os.getpid())        p = Process(target=run_proc, args=('test %s' %i,))    signal.signal(signal.SIGTERM, term)print('Parent process end.')
```

在 linux 上执行 python 20.py &，kill 主进程 pid 结束父进程，输出如下，此时 ps -ux | grep python 查看子进程全部关闭。可以看到代码中使用 p.terminate 来通知子进程关闭，前面说了这是一种友好的通知方法，如果子进程响应但是不退出也可以，如果强制子进程关闭可以直接调用 os.kill(p.pid, signal.SIGKILL) 发送 SIGKILL 消息，相当于 shell 中 kill -9 效果。注意代码中注册 SIGTERM 信号的位置，不能在子进程创建前注册，否则子进程也会响应 term 中又会对子进程发送消息，会形成死循环。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-11-13%2011-14-53/7c9a38ee-0f9f-4971-b2d3-9ee36cde57e0.png?raw=true)

 现在再说回 Windows 上的进程关闭，如何实现父进程关闭子进程关闭呢？显然我们需要就是一个父进程响应关闭并在实际关闭前清理子进程，这一点可以借助 windows 的消息机制的来处理，可在父进程中内建一个窗口接收消息，父进程响应消息后关闭子进程，使用消息通知父进程关闭即可达到和 linux 上一样效果，这也是在 Windows 上实现优雅关闭的常用方法（一般是查找主窗口发送 WM_CLOSE 消息）。

## 3. 子进程被强制关闭时，关闭父进程

很自然思考子进程强制关闭时必须关闭父进程的情况，这种情况一般适用于父进程对子进程有严重依赖的地方，如父进程调试子进程等等。有了前面的探讨，很自然想到可以把父进程 pid 传给子进程，在通知子进程关闭时 (Windows 借助消息完成通知) 通知父进程关闭。还有别的方法吗？

先说 Linux，主进程中创建子进程时，父进程与其创建的子进程称为进程组，同一个进程组中的进程，它们的进程组 ID 是一致的。利用 python 标准库中 os.getpgid 方法，获取进程对应的组 ID，在子进程响应关闭消息中调用 os.killpg 方法，向进程的组 ID 发送信号即可，python 30.py 代码如下：

```null
from multiprocessing import Processdef term(sig_num, addtion):print 'current pid is %s, group id is %s' % (os.getpid(), os.getpgrp())    os.killpg(os.getpgid(os.getpid()), signal.SIGKILL)print('Run child process %s (%s)...' % (name, os.getpid()))print('Exit child process %s (%s)...' % (name, os.getpid()))if __name__ == '__main__':    signal.signal(signal.SIGTERM, term)print('Parent process %s start.' % os.getpid())        p = Process(target=run_proc, args=('test %s' %i,))print('Parent process end.')
```

运行结果如下：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-11-13%2011-14-53/d15e7ec1-b45a-4837-9c21-a6828f681116.png?raw=true)

Windows 中有个作业 (Job) 的概念，类似进程组，可以完成相同功能；还可以使用 windows 的调试功能，父进程 WaitForDebugEvent 接收子进程的调试事件，捕获到子进程退出时，父进程也退出，这些都可以完成功能，但是必须间接调用 windows 系统接口了，恕不赘述。

## 4. 父子进程没必然联系，关闭互不影响

这种情况不多，简单提下，其实 multiprocessing 就是对 linux 和 windows 上的接口做了包装，同时管理了父进程和子进程的关系，要想达到互不影响的目的，直接绕开 multiprocessing 模块调用对应系统接口接口。

参考 multiprocessing 源码，linux 上调用 os.fork()，windows 上调用 \_subprocess.CreateProcess 即可，恕不赘述。

演示代码[下载链接](https://download.csdn.net/download/wenzhou1219/10580379)

原创，转载请注明来自[http://blog.csdn.net/wenzhou1219](http://blog.csdn.net/wenzhou1219)
