# (转)Linux Futex的设计与实现 - i活着 - 博客园
**Linux 中的线程同步机制 (一) -- Futex**

引子  
在编译 2.6 内核的时候，你会在编译选项中看到\[\*] Enable futex support 这一项，上网查，有的资料会告诉你 "不选这个内核不一定能正确的运行使用 glibc 的程序"，那 futex 是什么？和 glibc 又有什么关系呢？  

1. 什么是 Futex  
Futex 是 Fast Userspace muTexes 的缩写，由 Hubertus Franke, Matthew Kirkwood, Ingo Molnar and Rusty Russell 共同设计完成。几位都是 linux 领域的专家，其中可能 Ingo Molnar 大家更熟悉一些，毕竟是 O(1) 调度器和 CFS 的实现者。

Futex 按英文翻译过来就是快速用户空间互斥体。其设计思想其实 不难理解，在传统的 Unix 系统中，System V IPC(inter process communication)，如 semaphores, msgqueues, sockets 还有文件锁机制 (flock()) 等进程间同步机制都是对一个内核对象操作来完成的，这个内核对象对要同步的进程都是可见的，其提供了共享 的状态信息和原子操作。当进程间要同步的时候必须要通过系统调用 (如 semop()) 在内核中完成。可是经研究发现，很多同步是无竞争的，即某个进程进入 互斥区，到再从某个互斥区出来这段时间，常常是没有进程也要进这个互斥区或者请求同一同步变量的。但是在这种情况下，这个进程也要陷入内核去看看有没有人 和它竞争，退出的时侯还要陷入内核去看看有没有进程等待在同一同步变量上。这些不必要的系统调用 (或者说内核陷入) 造成了大量的性能开销。为了解决这个问 题，Futex 就应运而生，Futex 是一种用户态和内核态混合的同步机制。首先，同步的进程间通过 mmap 共享一段内存，futex 变量就位于这段共享 的内存中且操作是原子的，当进程尝试进入互斥区或者退出互斥区的时候，先去查看共享内存中的 futex 变量，如果没有竞争发生，则只修改 futex, 而不 用再执行系统调用了。当通过访问 futex 变量告诉进程有竞争发生，则还是得执行系统调用去完成相应的处理(wait 或者 wake up)。简单的说，futex 就是通过在用户态的检查，（motivation）如果了解到没有竞争就不用陷入内核了，大大提高了 low-contention 时候的效率。 Linux 从 2.5.7 开始支持 Futex。

2. Futex 系统调用  
Futex 是一种用户态和内核态混合机制，所以需要两个部分合作完成，linux 上提供了 sys_futex 系统调用，对进程竞争情况下的同步处理提供支持。  
其原型和系统调用号为  
    #include &lt;linux/futex.h>  
    #include &lt;sys/time.h>  
    int futex (int \*uaddr, int op, int val, const struct timespec \*timeout,int \*uaddr2, int val3);  
    #define \_\_NR_futex              240

            虽然参数有点长，其实常用的就是前面三个，后面的 timeout 大家都能理解，其他的也常被 ignore。  
    uaddr 就是用户态下共享内存的地址，里面存放的是一个对齐的整型计数器。  
    op 存放着操作类型。定义的有 5 中，这里我简单的介绍一下两种，剩下的感兴趣的自己去 man futex  
    FUTEX_WAIT: 原子性的检查 uaddr 中计数器的值是否为 val, 如果是则让进程休眠，直到 FUTEX_WAKE 或者超时 (time-out)。也就是把进程挂到 uaddr 相对应的等待队列上去。  
    FUTEX_WAKE: 最多唤醒 val 个等待在 uaddr 上进程。

        可见 FUTEX_WAIT 和 FUTEX_WAKE 只是用来挂起或者唤醒进程，当然这部分工作也只能在内核态下完成。有些人尝试着直接使用 futex 系统调 用来实现进程同步，并寄希望获得 futex 的性能优势，这是有问题的。应该区分 futex 同步机制和 futex 系统调用。futex 同步机制还包括用户态 下的操作，我们将在下节提到。

        3\. Futex 同步机制  
所有的 futex 同步操作都应该从用户空间开始，首先创建一个 futex 同步变量，也就是位于共享内存的一个整型计数器。  
当 进程尝试持有锁或者要进入互斥区的时候，对 futex 执行 "down" 操作，即原子性的给 futex 同步变量减 1。如果同步变量变为 0，则没有竞争发生， 进程照常执行。如果同步变量是个负数，则意味着有竞争发生，需要调用 futex 系统调用的 futex_wait 操作休眠当前进程。  
当进程释放锁或 者要离开互斥区的时候，对 futex 进行 "up" 操作，即原子性的给 futex 同步变量加 1。如果同步变量由 0 变成 1，则没有竞争发生，进程照常执行。如 果加之前同步变量是负数，则意味着有竞争发生，需要调用 futex 系统调用的 futex_wake 操作唤醒一个或者多个等待进程。

这里的原子性加减通常是用 CAS(Compare and Swap) 完成的，与平台相关。CAS 的基本形式是：CAS(addr,old,new), 当 addr 中存放的值等于 old 时，用 new 对其替换。在 x86 平台上有专门的一条指令来完成它: cmpxchg。

可见: futex 是从用户态开始，由用户态和核心态协调完成的。

4. 进 / 线程利用 futex 同步  
进程或者线程都可以利用 futex 来进行同步。  
对于线程，情况比较简单，因为线程共享虚拟内存空间，虚拟地址就可以唯一的标识出 futex 变量，即线程用同样的虚拟地址来访问 futex 变量。  
对 于进程，情况相对复杂，因为进程有独立的虚拟内存空间，只有通过 mmap() 让它们共享一段地址空间来使用 futex 变量。每个进程用来访问 futex 的 虚拟地址可以是不一样的，只要系统知道所有的这些虚拟地址都映射到同一个物理内存地址，并用物理内存地址来唯一标识 futex 变量。 

    小结：

1. Futex 变量的特征：1) 位于共享的用户空间中 2) 是一个 32 位的整型 3) 对它的操作是原子的  
2. Futex 在程序 low-contention 的时候能获得比传统同步机制更好的性能。  
3. 不要直接使用 Futex 系统调用。  
4. Futex 同步机制可以用于进程间同步，也可以用于线程间同步。

**Linux 中的线程同步机制 (二)--In Glibc**

　　在 linux 中进行多线程开发，同步是不可回避的一个问题。在 POSIX 标准中定义了三种线程同步机制: Mutexes(互斥量), Condition Variables(条件变量) 和 POSIX Semaphores(信号量)。NPTL 基本上实现了 POSIX，而 glibc 又使用 NPTL 作为自己的线程库。因此 glibc 中包含了这三种同步机制 的实现 (当然还包括其他的同步机制，如 APUE 里提到的读写锁)。

Glibc 中常用的线程同步方式举例:

Semaphore  
变量定义：    sem_t sem;  
初始化：      sem_init(&sem,0,1);  
进入加锁:     sem_wait(&sem);  
退出解锁:     sem_post(&sem);

Mutex  
变量定义：    pthread_mutex_t mut;  
初始化：      pthread_mutex_init(&mut,NULL);  
进入加锁:     pthread_mutex_lock(&mut);  
退出解锁:     pthread_mutex_unlock(&mut);

这些用于同步的函数和 futex 有什么关系？下面让我们来看一看:  
以 Semaphores 为例，  
进入互斥区的时候，会执行 sem_wait(sem_t \*sem)，sem_wait 的实现如下：  
int sem_wait (sem_t \*sem)  
{  
int \*futex = (int \*) sem;  
if (atomic_decrement_if_positive (futex) > 0)  
    return 0;  
int   err = lll_futex_wait (futex, 0);  
    return -1;  
)  
atomic_decrement_if_positive() 的语义就是如果传入参数是正数就将其原子性的减一并立即返回。如果信号量为正，在 Semaphores 的语义中意味着没有竞争发生，如果没有竞争，就给信号量减一后直接返回了。

如果传入参数不是正数，即意味着有竞争，调用 lll_futex_wait(futex,0),lll_futex_wait 是个宏，展开后为：  
#define lll_futex_wait(futex, val) \\  
({                                          \\  
    ...  
    \_\_asm \_\_volatile (LLL_EBX_LOAD                          \\  
              LLL_ENTER_KERNEL                          \\  
              LLL_EBX_LOAD                          \\  
              : "=a" (\_\_status)                          \\  
              : "0" (SYS_futex), LLL_EBX_REG (futex), "S" (0),          \\  
            "c" (FUTEX_WAIT), "d" (\_val),                  \\  
            "i" (offsetof (tcbhead_t, sysinfo))              \\  
              : "memory");                          \\  
    ...                                      \\  
})  
可以看到当发生竞争的时候，sem_wait 会调用 SYS_futex 系统调用，并在 val=0 的时候执行 FUTEX_WAIT, 让当前线程休眠。

　　从 这个例子我们可以看出，在 Semaphores 的实现过程中使用了 futex，不仅仅是说其使用了 futex 系统调用 (再重申一遍只使用 futex 系统调 用是不够的)，而是整个建立在 futex 机制上，包括用户态下的操作和核心态下的操作。其实对于其他 glibc 的同步机制来说也是一样, 都采纳了 futex 作为其基础。所以才会在 futex 的 manual 中说：对于大多数程序员不需要直接使用 futexes，取而代之的是依靠建立在 futex 之上 的系统库，如 NPTL 线程库 (most programmers will in fact not be using futexes directly but instead rely on system libraries built on them, such as the NPTL pthreads implementation)。所以才会有如果在编译内核的时候不 Enable futex support，就 "不一定能正确的运行使用 Glibc 的程序"。

小结:  
1. Glibc 中的所提供的线程同步方式，如大家所熟知的 Mutex,Semaphore 等，大多都构造于 futex 之上了，除了特殊情况，大家没必要再去实现自己的 futex 同步原语。  
2. 大家要做的事情，似乎就是按 futex 的 manual 中所说得那样: 正确的使用 Glibc 所提供的同步方式，并在使用它们的过程中，意识到它们是利用 futex 机制和 linux 配合完成同步操作就可以了。

　　上回说到 Glibc 中 (NPTL) 的线程同步方式如 Mutex,Semaphore 等都使用了 futex 作为其基础。那么实际使用是什么样子，又会碰到什么问题呢？  
先来看一个使用 semaphore 同步的例子。

sem_t sem_a;  
void \*task1();

int main(void){  
int ret=0;  
pthread_t thrd1;  
sem_init(&sem_a,0,1);  
ret=pthread_create(&thrd1,NULL,task1,NULL); // 创建子线程  
pthread_join(thrd1,NULL); // 等待子线程结束  
}

void \*task1()  
{  
int sval = 0;  
sem_wait(&sem_a); // 持有信号量  
sleep(5); //do_nothing  
sem_getvalue(&sem_a,&sval);  
printf("sem value = %d\\n",sval);  
sem_post(&sem_a); // 释放信号量  
}

　　程序很简单，我们在主线程 (执行 main 的线程) 中创建了一个线程，并用 join 等待其结束。在子线程中，先持有信号量，然后休息一会儿，再释放信号量，结束。  
因为这段代码中只有一个线程使用信号量，也就是没有线程间竞争发生，按照 futex 的理论，因为没有竞争，所以所有的锁操作都将在用户态中完成，而不会执行系统调用而陷入内核。我们用 strace 来跟踪一下这段程序的执行过程中所发生的系统调用:  
...  
20533 futex(0xb7db1be8, FUTEX_WAIT, 20534, NULL &lt;unfinished ...>  
20534 futex(0x8049870, FUTEX_WAKE, 1)   = 0  
20533 &lt;... futex resumed> )             = 0  
...   
20533 是 main 线程的 id,20534 是其子线程的 id。出乎我们意料之外的是这段程序还是发生了两次 futex 系统调用，我们来分析一下这分别是什么原因造成的。

1. 出人意料的 "sem_post()"  
20534 futex(0x8049870, FUTEX_WAKE, 1)   = 0  
子 线程还是执行了 FUTEX_WAKE 的系统调用，就是在 sem_post(&sem_a); 的时候，请求内核唤醒一个等待在 sem_a 上的线程, 其返回值是 0，表示现在并没有线程等待在 sem_a(这是当然的，因为就这么一个线程在使用 sem_a)，这次 futex 系统调用白做了。这似乎和 futex 的理论有些出入，我们再来看一下 sem_post 的实现。  
int sem_post (sem_t \*sem)  
{  
int \*futex = (int \*) sem;  
int nr = atomic_increment_val (futex);  
int err = lll_futex_wake (futex, nr);  
return 0;  
}  
　　我们看到，Glibc 在实现 sem_post 的时候给 futex 原子性的加上 1 后，不管 futex 的值是什么，都执行了 lll_futex_wake(), 即 futex(FUTEX_WAKE) 系统调用。  
在 第二部分中 (见前文)，我们分析了 sem_wait 的实现，当没有竞争的时候是不会有 futex 调用的，现在看来真的是这样，但是在 sem_post 的时 候，无论有无竞争，都会调用 sys_futex()，为什么会这样呢？我觉得应该结合 semaphore 的语义来理解。在 semaphore 的语义 中，sem_wait() 的意思是："挂起当前进程，直到 semaphore 的值为非 0，它会原子性的减少 semaphore 计数值。" 我们可以看到, semaphore 中是通过 0 或者非 0 来判断阻塞或者非阻塞线程。即无论有多少线程在竞争这把锁，只要使用了 semaphore，semaphore 的值都会是 0。这样，当线程推出互斥区，执行 sem_post(), 释放 semaphore 的时候，将其值由 0 改 1，并不知道是否有线程阻塞在这个 semaphore 上，所以只好不管怎么样都执行 futex(uaddr, FUTEX_WAKE, 1) 尝试着唤醒一个进程。而相反的，当 sem_wait()，如果 semaphore 由 1 变 0，则意味着没有竞争发生，所以不必去执行 futex 系统调 用。我们假设一下，如果抛开这个语义，如果允许 semaphore 值为负，则也可以在 sem_post() 的时候，实现 futex 机制。

2. 半路杀出的 "pthread_join()"  
　　那另一个 futex 系统调用是怎么造成的呢? 是因为 pthread_join(); 在 Glibc 中，pthread_join 也是用 futex 系统调用实现的。程序中的 pthread_join(thrd1,NULL); 就对应着 20533 futex(0xb7db1be8, FUTEX_WAIT, 20534, NULL &lt;unfinished ...>很 好解释，主线程要等待子线程 (id 号 20534 上) 结束的时候，调用 futex(FUTEX_WAIT)，并把 var 参数设置为要等待的子线程号 (20534)，然后等待在一个地址为 0xb7db1be8 的 futex 变量上。当子线程结束后，系统会负责把主线程唤醒。于是主线程就 20533 &lt;... futex resumed> ) = 0 恢复运行了。要注意的是，如果在执行 pthread_join() 的时候，要 join 的线程已经结束了，就不会再调用 futex() 阻塞当前进程了。

3. 更多的竞争。  
我们把上面的程序稍微改改:   
在 main 函数中:  
int main(void){  
...  
sem_init(&sem_a,0,1);  
ret=pthread_create(&thrd1,NULL,task1,NULL);  
ret=pthread_create(&thrd2,NULL,task1,NULL);  
ret=pthread_create(&thrd3,NULL,task1,NULL);  
ret=pthread_create(&thrd4,NULL,task1,NULL);  
pthread_join(thrd1,NULL);  
pthread_join(thrd2,NULL);  
pthread_join(thrd3,NULL);  
pthread_join(thrd4,NULL);  
...  
}

这样就有更的线程参与 sem_a 的争夺了。我们来分析一下，这样的程序会发生多少次 futex 系统调用。  
1) sem_wait()  
    第一个进入的线程不会调用 futex，而其他的线程因为要阻塞而调用，因此 sem_wait 会造成 3 次 futex(FUTEX_WAIT) 调用。  
2) sem_post()  
    所有线程都会在 sem_post 的时候调用 futex, 因此会造成 4 次 futex(FUTEX_WAKE) 调用。  
3) pthread_join()  
    别忘了还有 pthread_join()，我们是按 thread1, thread2, thread3, thread4 这样来 join 的，但是线程的调度存在着随机性。如果 thread1 最后被调度，则只有 thread1 这一次 futex 调用，所以 pthread_join() 造成的 futex 调用在 1-4 次之间。(虽然不是必然的，但是 4 次更常见一些)      
所以这段程序至多会造成 3+4+4=11 次 futex 系统调用，用 strace 跟踪，验证了我们的想法。  
19710 futex(0xb7df1be8, FUTEX_WAIT, 19711, NULL &lt;unfinished ...>  
19712 futex(0x8049910, FUTEX_WAIT, 0, NULL &lt;unfinished ...>  
19713 futex(0x8049910, FUTEX_WAIT, 0, NULL &lt;unfinished ...>  
19714 futex(0x8049910, FUTEX_WAIT, 0, NULL &lt;unfinished ...>  
19711 futex(0x8049910, FUTEX_WAKE, 1 &lt;unfinished ...>  
19710 futex(0xb75f0be8, FUTEX_WAIT, 19712, NULL &lt;unfinished ...>  
19712 futex(0x8049910, FUTEX_WAKE, 1 &lt;unfinished ...>  
19710 futex(0xb6defbe8, FUTEX_WAIT, 19713, NULL &lt;unfinished ...>  
19713 futex(0x8049910, FUTEX_WAKE, 1 &lt;unfinished ...>  
19710 futex(0xb65eebe8, FUTEX_WAIT, 19714, NULL &lt;unfinished ...>  
19714 futex(0x8049910, FUTEX_WAKE, 1)   = 0  
(19710 是主线程，19711，19712，19713,19714 是 4 个子线程)

4. 更多的问题  
事 情到这里就结束了吗？ 如果我们把 semaphore 换成 Mutex 试试。你会发现当自始自终没有竞争的时候，mutex 会完全符合 futex 机制，不管是 lock 还是 unlock 都不会调用 futex 系统调用。有竞争的时候，第一次 pthread_mutex_lock 的时候不会调用 futex 调用，看起来还正常。但 是最后一次 pthread_mutex_unlock 的时候，虽然已经没有线程在等待 mutex 了，可还是会调用 futex(FUTEX_WAKE)。原因是什么？欢迎讨论！！！

小结：  
1. 虽然 semaphore，mutex 等同步方式构建在 futex 同步机制之上。然而受其语义等的限制，并没有完全按 futex 最初的设计实现。  
2. pthread_join() 等函数也是调用 futex 来实现的。  
3. 不同的同步方式都有其不同的语义，不同的性能特征，适合于不同的场景。我们在使用过程中要知道他们的共性，也得了解它们之间的差异。这样才能更好的理解多线程场景，写出更高质量的多线程程序。

转载地址：  
[http://blog.csdn.net/Javadino/archive/2008/09/06/2891385.aspx](http://blog.csdn.net/Javadino/archive/2008/09/06/2891385.aspx)

[http://blog.csdn.net/Javadino/archive/2008/09/06/2891388.aspx](http://blog.csdn.net/Javadino/archive/2008/09/06/2891388.aspx)

[http://blog.csdn.net/Javadino/archive/2008/09/06/2891399.aspx](http://blog.csdn.net/Javadino/archive/2008/09/06/2891399.aspx) 
 [https://www.cnblogs.com/yysblog/archive/2012/11/03/2752728.html](https://www.cnblogs.com/yysblog/archive/2012/11/03/2752728.html)
