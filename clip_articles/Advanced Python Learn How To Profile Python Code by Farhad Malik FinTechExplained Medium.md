# Advanced Python: Learn How To Profile Python Code | by Farhad Malik | FinTechExplained | Medium
## Explaining How To Detect & Resolve Performance Bottlenecks

No one wants a slow data science application. Not when it is required to be consumed by high-demanding users who expect a quick turn around.

Profiling is one of those concepts that every Python programmer must be familiar with. It is required to become an expert in the programming field. This is an advanced level topic for Python developers and I recommend it to everyone who is/or intends in using the Python programming language.

We can learn the Python library and understand how to create objects and modules, but the true Python experts emerge when they encounter and fix tough technical issues. One of those issues requires resolving performance bottlenecks that revolve around profiling the code.

Let’s understand how profiling works.

![](https://miro.medium.com/max/1400/1*6CRHiRaClUnw8U6Et-pkBg.png)

Profiling requires analysing, assessing and understanding the bottlenecks in the code

Profiling is often performed to find performance bottlenecks in the code. Finding bottlenecks in the code is an art and with experience and knowledge, it gets easier. There are some expert tricks and tips that I will outline in this article.

> This article will demonstrate how to profile the Python code the right way.

One of the skills to master is the ability to find the bottlenecks in the Python code and then fix them the right way to give an application performance boost. The journey of fixing performance bottlenecks requires one to profile the code. Only then, the issues can be resolved.

This article will start by explaining what profiling is. It will then demonstrate the steps and techniques that are required to profile the Python code. The article will also provide an overview of what to look for and how to profile the Python programming code the right way.

I will present three profiling methodologies.

Lastly, the article will touch on how to optimise the code and the tips to remember whilst optimising the code.

> To be an expert level Python programmer, not only we need to know the ins and outs of the Python programming language, but we are also expected to learn to use Python as a tool.

If you want to understand the Python programming language from the beginner to an advanced level then I highly recommend the article below:

It is common for advanced level Python programmers to be able to profile and find bottlenecks in the code.

In a nutshell, profiling is all about finding bottlenecks in your code.

In simple words, the process of profiling concentrates on going over your application’s codebase and assessing or analyzing how long each function/line of code takes and how often it gets executed to understand how the code can be optimised.

Profiling is a quantitative technique. Therefore it produces a set of statistics.

> The exercise of profiling produces a set of statistical measures. These statistics enable us to understand how long each part of the code took and how many times it was executed.

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

If the data structures are appropriately chosen then one of the quick ways to gain a performance boost is to introduce concurrency and parallelism in your code.  
I highly recommend reading this article that explains the concept of concurrency in Python from the basics. This article is considered the must-read article of FinTechExplained publication by many readers:

An important tip is to only introduce parallelism and concurrency if you really have to introduce it. Plus, try not to introduce caching or multiple processing until your code works because these optimisations have other side effects that need to be considered. These include debugging and finding the root causes of issues when embarrassing bugs appear in the production environment.

Here are some of the optimisation tips I encourage everyone to read.

-   The first step in optimising the code is to ensure that the solution works and solves the problem we expect it to solve. For everything that works, write unit tests. Unit tests are meant to be atomic in nature. They should only test a single unit of code. Hence we can easily ensure that the code, pre-and-post optimisation, is working as desired. Unit tests will ensure that our intended changes haven’t broken the functionality. Always ensure every single path of the code branch is unit tested to increase the confidence in the refactor.
-   Whilst optimising the code, ensure it is easily readable. As an instance, if you want to introduce multiprocessing/threads or retry logic, abstract it out into other module or introduce a decorator and pass the function as an argument to ensure the code is cleaner, it is easier to understand for others in the team.
-   The use of decorators is encouraged as it reduces code complexity and makes the code readable.

Profiling is often performed to find performance bottlenecks in the code. Finding bottlenecks in the code is an art and with experience and knowledge, it gets easier.

![](https://miro.medium.com/max/1400/1*3Njf1zvfTii3xvmq9wMDPw.png)

Thank you for reading FinTechExplained

This article started by explaining what profiling is. It then demonstrated the steps and techniques required to profile Python code. The article also provided an overview of what to look for and how to profile the Python programming code the right way. I also presented three profiling methodologies.

Lastly, the article touched on how to optimise the code and the tips to remember whilst optimising the code. 
 [https://medium.com/fintechexplained/advanced-python-learn-how-to-profile-python-code-1068055460f9#id_token=eyJhbGciOiJSUzI1NiIsImtpZCI6IjFiZDY4NWY1ZThmYzYyZDc1ODcwNWMxZWIwZThhNzUyNGM0NzU5NzUiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2dsZS5jb20iLCJuYmYiOjE2NTcyNjE4NjIsImF1ZCI6IjIxNjI5NjAzNTgzNC1rMWs2cWUwNjBzMnRwMmEyamFtNGxqZGNtczAwc3R0Zy5hcHBzLmdvb2dsZXVzZXJjb250ZW50LmNvbSIsInN1YiI6IjEwOTExOTM2MjcwNzQ0NDI3MTM2MiIsImVtYWlsIjoic3dpdGNoeHVAZ21haWwuY29tIiwiZW1haWxfdmVyaWZpZWQiOnRydWUsImF6cCI6IjIxNjI5NjAzNTgzNC1rMWs2cWUwNjBzMnRwMmEyamFtNGxqZGNtczAwc3R0Zy5hcHBzLmdvb2dsZXVzZXJjb250ZW50LmNvbSIsIm5hbWUiOiJDaGlod2VpIEhzdSIsInBpY3R1cmUiOiJodHRwczovL2xoMy5nb29nbGV1c2VyY29udGVudC5jb20vYS0vQUZkWnVjb2phNmhRQUxINkpSYXZEVFRSaUhmenF2MXhjZEZvZkVFNi1Oc3I9czk2LWMiLCJnaXZlbl9uYW1lIjoiQ2hpaHdlaSIsImZhbWlseV9uYW1lIjoiSHN1IiwiaWF0IjoxNjU3MjYyMTYyLCJleHAiOjE2NTcyNjU3NjIsImp0aSI6IjYyMTAxNzc4YzM4YmVlODBkYzNlY2VhYmI0NWI3NDMyZjZiYjk0N2UifQ.DFd9Ae9Ow9W4BaG5hk1OopMRWi69RwzX-lwbTneL6ely8GePmnWUeTVh70oZe6JpR2ZnNatt8hefxb6Ruwq_I7h3foKmQEyuC-Zv-iSWAfRDUugDGrrI6qJAZdxOeSw1aGRqp2zqUAO0xhZsgYSqwafB4MXyCWLePScqtbDxlKr3Jf1IKMTqZ9j_-Vpb41lM5MmCGtqAffJlRu2ropxTwMpVhFlM1AQSYJuK9TcHEjyuCiBnRZJPkAgPk4uNobDcTD-cPC43mUDcTQ-gVYXaYo7AuoBEiDwCBEUp3fPcp5nnxpw1kfCd32S6wt49rkRXPe3j3SfxiTEh3XEarWIKyA](https://medium.com/fintechexplained/advanced-python-learn-how-to-profile-python-code-1068055460f9#id_token=eyJhbGciOiJSUzI1NiIsImtpZCI6IjFiZDY4NWY1ZThmYzYyZDc1ODcwNWMxZWIwZThhNzUyNGM0NzU5NzUiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2dsZS5jb20iLCJuYmYiOjE2NTcyNjE4NjIsImF1ZCI6IjIxNjI5NjAzNTgzNC1rMWs2cWUwNjBzMnRwMmEyamFtNGxqZGNtczAwc3R0Zy5hcHBzLmdvb2dsZXVzZXJjb250ZW50LmNvbSIsInN1YiI6IjEwOTExOTM2MjcwNzQ0NDI3MTM2MiIsImVtYWlsIjoic3dpdGNoeHVAZ21haWwuY29tIiwiZW1haWxfdmVyaWZpZWQiOnRydWUsImF6cCI6IjIxNjI5NjAzNTgzNC1rMWs2cWUwNjBzMnRwMmEyamFtNGxqZGNtczAwc3R0Zy5hcHBzLmdvb2dsZXVzZXJjb250ZW50LmNvbSIsIm5hbWUiOiJDaGlod2VpIEhzdSIsInBpY3R1cmUiOiJodHRwczovL2xoMy5nb29nbGV1c2VyY29udGVudC5jb20vYS0vQUZkWnVjb2phNmhRQUxINkpSYXZEVFRSaUhmenF2MXhjZEZvZkVFNi1Oc3I9czk2LWMiLCJnaXZlbl9uYW1lIjoiQ2hpaHdlaSIsImZhbWlseV9uYW1lIjoiSHN1IiwiaWF0IjoxNjU3MjYyMTYyLCJleHAiOjE2NTcyNjU3NjIsImp0aSI6IjYyMTAxNzc4YzM4YmVlODBkYzNlY2VhYmI0NWI3NDMyZjZiYjk0N2UifQ.DFd9Ae9Ow9W4BaG5hk1OopMRWi69RwzX-lwbTneL6ely8GePmnWUeTVh70oZe6JpR2ZnNatt8hefxb6Ruwq_I7h3foKmQEyuC-Zv-iSWAfRDUugDGrrI6qJAZdxOeSw1aGRqp2zqUAO0xhZsgYSqwafB4MXyCWLePScqtbDxlKr3Jf1IKMTqZ9j_-Vpb41lM5MmCGtqAffJlRu2ropxTwMpVhFlM1AQSYJuK9TcHEjyuCiBnRZJPkAgPk4uNobDcTD-cPC43mUDcTQ-gVYXaYo7AuoBEiDwCBEUp3fPcp5nnxpw1kfCd32S6wt49rkRXPe3j3SfxiTEh3XEarWIKyA)
