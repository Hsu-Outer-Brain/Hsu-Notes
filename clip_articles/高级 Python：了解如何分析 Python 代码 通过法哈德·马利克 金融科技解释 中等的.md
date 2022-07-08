# 高级 Python：了解如何分析 Python 代码 | 通过法哈德·马利克 | 金融科技解释 | 中等的
## 解释如何检测和解决性能瓶颈

没有人想要一个缓慢的数据科学应用程序。当要求快速周转的高要求用户需要使用它时，则不是。

分析是每个 Python 程序员必须熟悉的概念之一。需要成为编程领域的专家。这是 Python 开发人员的高级主题，我向所有 / 或打算使用 Python 编程语言的人推荐它。

我们可以学习 Python 库并了解如何创建对象和模块，但真正的 Python 专家会在遇到并解决棘手的技术问题时出现。其中一个问题需要解决围绕代码分析的性能瓶颈。

让我们了解分析是如何工作的。

![](https://miro.medium.com/max/1400/1*6CRHiRaClUnw8U6Et-pkBg.png)

分析需要分析、评估和理解代码中的瓶颈

通常执行分析以发现代码中的性能瓶颈。找到代码中的瓶颈是一门艺术，有了经验和知识，它会变得更容易。我将在本文中概述一些专家技巧和技巧。

> 本文将演示如何以正确的方式分析 Python 代码。

要掌握的技能之一是能够找到 Python 代码中的瓶颈，然后以正确的方式修复它们以提高应用程序的性能。修复性能瓶颈的过程需要分析代码。只有这样，问题才能得到解决。

本文将首先解释什么是分析。然后，它将演示分析 Python 代码所需的步骤和技术。本文还将概述要查找的内容以及如何以正确的方式分析 Python 编程代码。

我将介绍三种分析方法。

最后，本文将介绍如何优化代码以及优化代码时要记住的技巧。

> 要成为专家级的 Python 程序员，我们不仅需要了解 Python 编程语言的来龙去脉，还需要学习使用 Python 作为工具。

如果你想从初学者到高级理解 Python 编程语言，那么我强烈推荐下面的文章：

高级 Python 程序员能够分析和发现代码中的瓶颈是很常见的。

简而言之，分析就是在代码中找到瓶颈。

简而言之，分析过程集中在检查应用程序的代码库并评估或分析每个函数 / 代码行需要多长时间以及执行频率，以了解如何优化代码。

剖析是一种定量技术。因此，它产生一组统计数据。

> 剖析的练习产生了一组统计测量。这些统计数据使我们能够了解代码的每个部分花费了多长时间以及执行了多少次。

The statistical measures can be presented to the team of programmers and they can then be used to channel the team effort in a calculator manner. Profiling can unveil the mystery of how and what is required to be improved.

## The Steps Of Profiling

Essentially, the first step is to decide whether the required profiling technique is going to be at the macro or micro level.

-   The macro profiling is about profiling the entire program and generating statistical information while it is running.
-   Micro profiling is profiling a specific part of the program. Micro-profiling is generally ad-hoc and manual in nature.

There are multiple ways and a number of helpful packages available to profile the Python code.

I recommend using the cProfile library. It is a C extension and is suitable for profiling long-running applications.

We can perform simple, intermediate, or advanced profiling. Let’s understand how to profile next.

The best way to understand profiling is to consider a real use-case as an example. This example is created to help us demonstrate the techniques of profiling better.

The function below is used to calculate the total risk and return that is allocated to a company.

    def get\_company\_details(symbol):  
        end\_date = datetime.now() - timedelta(days=1)  
        start\_date = end\_date - timedelta(days=10)  
        prices = yf.download(symbol, start=start\_date,   
                  end=end\_date)\['Close'\]  
        log\_returns = np.log(prices) - np.log(prices.shift(1))  
        total\_return = log\_returns.sum()  
        total\_risk = log\_returns.std()  
        return symbol, total\_return, total\_risk

-   The code snippet above takes in a company symbol as a function argument.
-   It then queries yahoo finance web service to get the last 10 days' prices of the company. This is an IO-bound operation.
-   From the prices, it computes the total return and risk of the company. Both of them are CPU intensive operations.

This function is called multiple times by the function get_all_companies_data() below.

    def get\_all\_companies\_data():  
        symbols = \['ADM', 'ADT', 'AHT', 'III'\]  
        for symbol in symbols:  
            symbol, total\_return, total\_risk =                                                          
                          get\_company\_details(symbol)  
            print(f'SYMBOL: {symbol}.   
              TOTAL RISK: {total\_risk}. TOTAL RETURN: {total\_return}')

We can run the code as shown below:

    get\_all\_companies\_data()

Running the code will print the Symbol, Total Risk and Total Return of all of the required companies:

    get\_all\_companies\_data()

Running the code will print the following results

    SYMBOL: ADM. TOTAL RISK: 0.021869461018335933. TOTAL RETURN: 0.030534911987234015SYMBOL: ADT. TOTAL RISK: 0.033611646937176394. TOTAL RETURN: 0.06830708186185985SYMBOL: AHT. TOTAL RISK: 0.06504280244333693. TOTAL RETURN: -0.1232326135175989SYMBOL: III. TOTAL RISK: 0.032170296104463876. TOTAL RETURN: 0.047178627683446606

This section will focus on the three techniques now.

The simplest form of profiling requires running the function and assessing how long it took to execute and get the results.

We can use the time package:

    import time  
    start = time.time()  
    get\_all\_companies\_data()  
    end = time.time()  
    print(end - start)

All we have done here is to store the current time before and after the execution of the code. It will give the elapsed time between two points, in seconds. Therefore, the code above took 3.19 seconds to complete.

We can add these times in individual lines of the function to get micro-level profiling results.

Now the question is regarding its accuracy.

As an instance, assume that the Yahoo finance web service was experiencing heavy traffic when this function was called and its response was slow because of that. Maybe, I was experiencing network overload due to other applications running on my machine, or maybe the CPU was busy executing the other tasks. Therefore we need a mechanism to run the code multiple times so that we can take the average.

This brings me to Intermediate Level Profiling

I have written a decorator below. Decorators are great in the sense that they can be shared, encourage code reuse, and make the code readable.

I am going to use the timeit Python package

    import timeit  
    def time\_me(number\_of\_times):  
        def decorator(func):  
            @functools.wraps(func)  
            def wraps(\*args, \*\*kwargs):  
                r = timeit.timeit(func, number=number\_of\_times)  
                print(r/number\_of\_times)  
            return wraps  
        return decorator

The decorator above is called time_me. It takes in a parameter: number_of_times.

Internally it executes the decorated function multiple times as specified by the input argument and then it calculates the average time the function took to complete.

This reduces the uncertainty that maybe there was a network latency or CPU was busy performing other tasks.

To use it, simply decorate the function get_all_companie()s with the decorator:

    @time\_me(100)  
    def get\_all\_companies\_data():  
        symbols = \['ADM', 'ADT', 'AHT', 'III'\]  
        ....

This decorator will execute the get_all_companies_data() 100 times and print the average time the function took.

On average, it took 3.01 seconds on my machine.

It’s important to note that the `timeit` module prints CPU cycles and those are not absolutely comparable from one machine to another.

There is an advanced level profiling technique.

In this section, I will introduce the cProfile library. We can directly execute the `cProfile.run(python code)` function and it will output the reasonable statistics for us.

The cProfiler outputs the following statistical measures for each line of Python code:

-   The number of function calls and how many times the function was recursed
-   Calls that were not induced via recursion. It is known as primitive calls.
-   Number of times the function was executed
-   The total time spent in the given function
-   The total time per call

As an instance, we can simply pass in the name of the file to profile and it will output the above stats for us:

    python -m cprofile fintechexplained\_profiling.py

To profile the code `get_all_companies_data()`, I have written a generic decorator.

We can decorate any function with the decorator below and it will profile the code and output statistical measures for us:

    import cProfile  
    import functools  
    import pstats  
    import tempfile  
    def profile\_me(func):  
        @functools.wraps(func)  
        def wraps(\*args, \*\*kwargs):  
            file = tempfile.mktemp()  
            profiler = cProfile.Profile()  
            profiler.runcall(func, \*args, \*\*kwargs)  
            profiler.dump\_stats(file)  
            metrics = pstats.Stats(file)  
            metrics.strip\_dirs().sort\_stats('time').print\_stats(100)  
        return wraps

This decorator takes in a function and outputs the profiling statistics.

-   The `pstats.Stats()` method produced the required metrics
-   The `[strip_dirs()](https://docs.python.org/3/library/profile.html#pstats.Stats.strip_dirs)` method removed the extraneous path from all the module names.
-   We then used the `[sort_stats()](https://docs.python.org/3/library/profile.html#pstats.Stats.sort_stats)` method. It sorted the metrics by the time column.
-   Lastly the `[print_stats()](https://docs.python.org/3/library/profile.html#pstats.Stats.print_stats)` method is called to print out all the statistics.

> I recommend the cProfiler to profile the target code to obtain performance bottlenecks

We can also print details about a spefiic function e.g. `stats.print_callees(‘function_one’)`

All we have to do now is to add the decorator to the function:

    @profile\_me  
    def get\_all\_companies\_data():  
        symbols = \['ADM', 'ADT', 'AHT', 'III'\]  
        ....

It will output the metrics ordered by the time:

![](https://miro.medium.com/max/1400/1*UQ_Qb-fzaHAKyBh2_kfaQA.png)

Let’s go over the statistics above:

-   There were 9221 function calls and 9125 amongst them were not induced by recursion.
-   For each line, we can see the number of times the function was executed (ncalls), the total time spent in the given function (tottime), the total time per call (percall), and cumulative time details (cumtime and its percall).
-   We can profile the entire code base. This information can help us understand where we need to speed up the code.

We can view the profiling stats as a diagram using gprof2dot package.

> Other than the speed, we should also look out for memory consumption.  
> If your process memory is increasing over time even whilst it’s idle then it’s best to find the root cause of the issue and fix the memory leak.

Now that we have the quantitative stats, we can focus the effort on improving the performance of the slow code. This is a quantitative methodology to optimise the code which we can present to the team and make calculated decisions on how and where to spend our efforts.

The next steps are about code optimisation.

Lastly, I wanted to provide a quick overview of code optimisation.

Over the years I have noticed a pattern in programming.

Most, if not all, of the times whenever we attempt to optimise a solution, we end up increasing its complexity.

Increasing complexity tends to make the solution harder to maintain and debug. Consequently, it increases the overall cost of maintaining the application. If your optimised code makes the code harder to read and understand then it’s better to not optimise it because it will be costly in the long-run.

There are a number of optimisation strategies that we can follow to optimise our Python application.

It includes introducing caching, concurrency and parallelism, enabling Cython, Numba, and so on. However, the most important yet overlooked optimisation strategy is to evaluate and choose the right data structure for the job.

I highly recommend reading this article that explains the time complexities of the Python data structures. It will solve most of the performance issues.

如果数据结构选择得当，那么获得性能提升的一种快速方法是在代码中引入并发性和并行性。  
我强烈推荐阅读这篇从基础开始解释 Python 中并发概念的文章。这篇文章被很多读者认为是 FinTechExplained 刊物的必读文章：

一个重要的提示是，只有在你真的必须引入并行性和并发性时才引入它。另外，在您的代码工作之前尽量不要引入缓存或多重处理，因为这些优化还有其他需要考虑的副作用。这些包括在生产环境中出现令人尴尬的错误时调试和查找问题的根本原因。

以下是一些我鼓励大家阅读的优化技巧。

-   优化代码的第一步是确保解决方案有效并解决我们期望它解决的问题。对于所有工作，编写单元测试。单元测试本质上是原子的。他们应该只测试一个代码单元。因此，我们可以很容易地确保代码，前后优化，按预期工作。单元测试将确保我们预期的更改没有破坏功能。始终确保代码分支的每条路径都经过单元测试，以增加重构的信心。
-   在优化代码的同时，确保它易于阅读。例如，如果您想引入多处理 / 线程或重试逻辑，将其抽象到其他模块中或引入装饰器并将函数作为参数传递以确保代码更干净，团队中的其他人更容易理解.
-   鼓励使用装饰器，因为它可以降低代码复杂性并使代码可读。

通常执行分析以发现代码中的性能瓶颈。找到代码中的瓶颈是一门艺术，有了经验和知识，它会变得更容易。

![](https://miro.medium.com/max/1400/1*3Njf1zvfTii3xvmq9wMDPw.png)

感谢您阅读 FinTechExplained

本文首先解释什么是分析。然后演示了分析 Python 代码所需的步骤和技术。本文还概述了要查找的内容以及如何以正确的方式分析 Python 编程代码。我还介绍了三种分析方法。

最后，文章谈到了如何优化代码以及优化代码时要记住的技巧。 
 [https://medium.com/fintechexplained/advanced-python-learn-how-to-profile-python-code-1068055460f9#id_token=eyJhbGciOiJSUzI1NiIsImtpZCI6IjFiZDY4NWY1ZThmYzYyZDc1ODcwNWMxZWIwZThhNzUyNGM0NzU5NzUiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2dsZS5jb20iLCJuYmYiOjE2NTcyNjE4NjIsImF1ZCI6IjIxNjI5NjAzNTgzNC1rMWs2cWUwNjBzMnRwMmEyamFtNGxqZGNtczAwc3R0Zy5hcHBzLmdvb2dsZXVzZXJjb250ZW50LmNvbSIsInN1YiI6IjEwOTExOTM2MjcwNzQ0NDI3MTM2MiIsImVtYWlsIjoic3dpdGNoeHVAZ21haWwuY29tIiwiZW1haWxfdmVyaWZpZWQiOnRydWUsImF6cCI6IjIxNjI5NjAzNTgzNC1rMWs2cWUwNjBzMnRwMmEyamFtNGxqZGNtczAwc3R0Zy5hcHBzLmdvb2dsZXVzZXJjb250ZW50LmNvbSIsIm5hbWUiOiJDaGlod2VpIEhzdSIsInBpY3R1cmUiOiJodHRwczovL2xoMy5nb29nbGV1c2VyY29udGVudC5jb20vYS0vQUZkWnVjb2phNmhRQUxINkpSYXZEVFRSaUhmenF2MXhjZEZvZkVFNi1Oc3I9czk2LWMiLCJnaXZlbl9uYW1lIjoiQ2hpaHdlaSIsImZhbWlseV9uYW1lIjoiSHN1IiwiaWF0IjoxNjU3MjYyMTYyLCJleHAiOjE2NTcyNjU3NjIsImp0aSI6IjYyMTAxNzc4YzM4YmVlODBkYzNlY2VhYmI0NWI3NDMyZjZiYjk0N2UifQ.DFd9Ae9Ow9W4BaG5hk1OopMRWi69RwzX-lwbTneL6ely8GePmnWUeTVh70oZe6JpR2ZnNatt8hefxb6Ruwq_I7h3foKmQEyuC-Zv-iSWAfRDUugDGrrI6qJAZdxOeSw1aGRqp2zqUAO0xhZsgYSqwafB4MXyCWLePScqtbDxlKr3Jf1IKMTqZ9j_-Vpb41lM5MmCGtqAffJlRu2ropxTwMpVhFlM1AQSYJuK9TcHEjyuCiBnRZJPkAgPk4uNobDcTD-cPC43mUDcTQ-gVYXaYo7AuoBEiDwCBEUp3fPcp5nnxpw1kfCd32S6wt49rkRXPe3j3SfxiTEh3XEarWIKyA](https://medium.com/fintechexplained/advanced-python-learn-how-to-profile-python-code-1068055460f9#id_token=eyJhbGciOiJSUzI1NiIsImtpZCI6IjFiZDY4NWY1ZThmYzYyZDc1ODcwNWMxZWIwZThhNzUyNGM0NzU5NzUiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2dsZS5jb20iLCJuYmYiOjE2NTcyNjE4NjIsImF1ZCI6IjIxNjI5NjAzNTgzNC1rMWs2cWUwNjBzMnRwMmEyamFtNGxqZGNtczAwc3R0Zy5hcHBzLmdvb2dsZXVzZXJjb250ZW50LmNvbSIsInN1YiI6IjEwOTExOTM2MjcwNzQ0NDI3MTM2MiIsImVtYWlsIjoic3dpdGNoeHVAZ21haWwuY29tIiwiZW1haWxfdmVyaWZpZWQiOnRydWUsImF6cCI6IjIxNjI5NjAzNTgzNC1rMWs2cWUwNjBzMnRwMmEyamFtNGxqZGNtczAwc3R0Zy5hcHBzLmdvb2dsZXVzZXJjb250ZW50LmNvbSIsIm5hbWUiOiJDaGlod2VpIEhzdSIsInBpY3R1cmUiOiJodHRwczovL2xoMy5nb29nbGV1c2VyY29udGVudC5jb20vYS0vQUZkWnVjb2phNmhRQUxINkpSYXZEVFRSaUhmenF2MXhjZEZvZkVFNi1Oc3I9czk2LWMiLCJnaXZlbl9uYW1lIjoiQ2hpaHdlaSIsImZhbWlseV9uYW1lIjoiSHN1IiwiaWF0IjoxNjU3MjYyMTYyLCJleHAiOjE2NTcyNjU3NjIsImp0aSI6IjYyMTAxNzc4YzM4YmVlODBkYzNlY2VhYmI0NWI3NDMyZjZiYjk0N2UifQ.DFd9Ae9Ow9W4BaG5hk1OopMRWi69RwzX-lwbTneL6ely8GePmnWUeTVh70oZe6JpR2ZnNatt8hefxb6Ruwq_I7h3foKmQEyuC-Zv-iSWAfRDUugDGrrI6qJAZdxOeSw1aGRqp2zqUAO0xhZsgYSqwafB4MXyCWLePScqtbDxlKr3Jf1IKMTqZ9j_-Vpb41lM5MmCGtqAffJlRu2ropxTwMpVhFlM1AQSYJuK9TcHEjyuCiBnRZJPkAgPk4uNobDcTD-cPC43mUDcTQ-gVYXaYo7AuoBEiDwCBEUp3fPcp5nnxpw1kfCd32S6wt49rkRXPe3j3SfxiTEh3XEarWIKyA)
