# python中的进程、线程、协程
[python 中的进程、线程、协程](https://baijiahao.baidu.com/s?id=1743401084104975929&wfr=spider&for=pc) 

 大家好，有时候会听到有人评价 python 编程执行效率方面相对 java 没有啥优势，其实是没有找到正确的打开方式，编程中无论是 api 还是执行脚本，无论是 I/O 密集型任务还是计算密集型任务，都有其提升执行效率的方式，通常，我们的优化手段就是并发编程，实现多任务同时执行，改善系统性能。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-3%2014-19-58/a1d3c2ec-4ce0-46ff-8164-7132d2840596.webp?raw=true)

python 中实现并发编程主要是依靠，进程、线程和协程，我们先来了解下三者的概念。

**进程（process）**: 是操作系统进行资源分配的基本单元，是 CPU 对程序的一次执行过程，每个进程都是独立的，有自己的内存空间、数据栈等。

**线程（Thread）**：被包含在进程之中，是操作系统进行程序调度执行的最小单元。一个进程中至少有一个线程。

**协程（Coroutine）**：比线程更小的执行单元，也被成为微线程。单个线程上执行多个任务，等待、切换执行任务，线程内切换没有切换开销，执行效率高。  

我们把生产工厂比作一个操作系统，一条生产线是一个进程，生产线上人就是一个线程，生产线上的工作就是需要完成的任务。如果我们想要提高生产的话。我们有下面几种办法：

第一种：增加生产线（相当于增加进程，多进程执行）；

第二种：增加生产线上的人，提高生产（相当于增加线程，多线程执行）；

第三种：生产线上总有做的快和慢的任务，会有一个等待的过程，那么，当出现任务等待的时候，可以先去做可以做的工作，待任务出现后再执行。充分里面劳动时间提升效率（任务等待、切换相当于使用协程完成任务）；

python 中依赖标准库 mutiprocessing 实现多进程任务，其中进程的常用方法：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-3%2014-19-58/9db0cee7-7468-4537-bb8c-7df3eff547d0.webp?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-3%2014-19-58/294220a9-8a92-4032-a853-dc02815e72c1.webp?raw=true)

**注意**：上面进程的执行逻辑需要再 run 方法中实现，启动进程需要调用 start 方法，调用之后 run 方法便会执行。

python 依赖标准库 Threading 实现多线程，其中线程的常用方法有：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-3%2014-19-58/9942cd98-b9db-4d55-ac39-9e1103eb4478.webp?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-3%2014-19-58/17879aaa-0457-4ac9-abb4-0a7cc755bdf5.webp?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-3%2014-19-58/93f8cb3b-50ca-43c0-98fa-1f0380dffb5f.webp?raw=true)

协程的运行效率极高，切换完全由程序控制，不像进程和线程的切换需要花费系统的开销，并且协程不需要多线程的锁机制，因为只有一个线程。协程是微线程，对于计算密集型的任务都不太适用，但是对于 I/O 密集型的任务非常适用。如果遇到多核 CPU 且是计算密集型的任务，建议使用多进程 + 协程的方式，可以极大的提到性能。  

我们一般通过 asynio 模块来实现协程及并发操作，代码实例如下：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-3%2014-19-58/442102b8-cef0-4dc7-85f2-7ea3f6da03d3.webp?raw=true)

**需要注意的是**：async 和 await 这两个关键字只能在协程中使用；协程不能直接执行，需要在 asyncio.run() 中执行，也可以跟在 await 后面。

**小结：为了提高执行效率，我们应该先分清任务类型。计算密集型任务更加适合多进程；I/O 密集型任务适合多线程和协程；当然高并发下的最佳实践还是多进程 + 协程，可以极大提高执行效率，并且兼容了计算密集型任务和 I/O 密集型任务。** 

“今天的分享就到这里，希望对大家有所帮助，欢迎大家点赞收藏、参与评论，谢谢~”
