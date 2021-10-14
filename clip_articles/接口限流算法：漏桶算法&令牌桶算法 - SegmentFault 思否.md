# 接口限流算法：漏桶算法&令牌桶算法 - SegmentFault 思否
工作中对外提供的 API 接口设计都要考虑限流，如果不考虑限流，会成系统的连锁反应，轻者响应缓慢，重者系统宕机，整个业务线崩溃，如何应对这种情况呢，我们可以对请求进行引流或者直接拒绝等操作，保持系统的可用性和稳定性，防止因流量暴增而导致的系统运行缓慢或宕机。

在开发高并发系统时有三把利器用来保护系统：**缓存、降级和限流**

**缓存**：缓存的目的是提升系统访问速度和增大系统处理容量  
**降级**：降级是当服务器压力剧增的情况下，根据当前业务情况及流量对一些服务和页面有策略的降级，以此释放服务器资源以保证核心任务的正常运行  
**限流**：限流的目的是通过对并发访问 / 请求进行限速，或者对一个时间窗口内的请求进行限速来保护系统，一旦达到限制速率则可以拒绝服务、排队或等待、降级等处理

## 限流算法

常用的限流算法有**令牌桶和和漏桶**，而 Google 开源项目 Guava 中的 RateLimiter 使用的就是令牌桶控制算法。

### 漏桶算法

把请求比作是水，水来了都先放进桶里，并以限定的速度出水，当水来得过猛而出水不够快时就会导致水直接溢出，即拒绝服务。

![](https://segmentfault.com/img/remote/1460000015967925?w=443&h=299)

漏斗有一个进水口 和 一个出水口，出水口以一定速率出水，并且有一个最大出水速率：

在漏斗中没有水的时候，

-   如果进水速率小于等于最大出水速率，那么，出水速率等于进水速率，此时，不会积水
-   如果进水速率大于最大出水速率，那么，漏斗以最大速率出水，此时，多余的水会积在漏斗中

在漏斗中有水的时候

-   出水口以最大速率出水
-   如果漏斗未满，且有进水的话，那么这些水会积在漏斗中
-   如果漏斗已满，且有进水的话，那么这些水会溢出到漏斗之外

### 令牌桶算法

对于很多应用场景来说，除了要求能够限制数据的平均传输速率外，还要求允许某种程度的突发传输。这时候漏桶算法可能就不合适了，令牌桶算法更为适合。

令牌桶算法的原理是系统以恒定的速率产生令牌，然后把令牌放到令牌桶中，令牌桶有一个容量，当令牌桶满了的时候，再向其中放令牌，那么多余的令牌会被丢弃；当想要处理一个请求的时候，需要从令牌桶中取出一个令牌，如果此时令牌桶中没有令牌，那么则拒绝该请求。

![](https://segmentfault.com/img/remote/1460000015967926?w=363&h=215)

## RateLimiter 用法

[https://github.com/google/guava](https://link.segmentfault.com/?enc=WwISWAXVeX156N6TI5YUmA%3D%3D.0A5TonOuNVoWp3OQ8xD5tsxrmV8KF%2BV6OQBPTQpVPbc%3D)

添加依赖

    <dependency\>
      <groupId\>com.google.guava</groupId\>
      <artifactId\>guava</artifactId\>
      <version\>26.0-jre</version\>
      
      <version\>26.0-android</version\>
    </dependency\>

    public class Test {

        public static void main(String\[\] args) {
            ListeningExecutorService executorService \= MoreExecutors.listeningDecorator(Executors.newFixedThreadPool(100));
            
      RateLimiter limiter \= RateLimiter.create(1);
            for (int i \= 1; i < 50; i++) {
                
      
      Double acquire \= null;
                if (i == 1) {
                    acquire = limiter.acquire(1);
                } else if (i == 2) {
                    acquire = limiter.acquire(10);
                } else if (i == 3) {
                    acquire = limiter.acquire(2);
                } else if (i == 4) {
                    acquire = limiter.acquire(20);
                } else {
                    acquire = limiter.acquire(2);
                }
                executorService.submit(new Task("获取令牌成功，获取耗：" + acquire + " 第 " + i + " 个任务执行"));
            }
        }
    }
    class Task implements Runnable {
        String str;
        public Task(String str) {
            this.str = str;
        }
        @Override
      public void run() {
            SimpleDateFormat sdf \= new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS");
            System.out.println(sdf.format(new Date()) + " | " + Thread.currentThread().getName() + str);
        }
    }

响应

    2018\-08\-11 00:26:22.953 | pool-1\-thread-1获取令牌成功，获取耗：0.0 第 1 个任务执行
    2018\-08\-11 00:26:23.923 | pool-1\-thread-2获取令牌成功，获取耗：0.98925 第 2 个任务执行
    2018\-08\-11 00:26:33.920 | pool-1\-thread-3获取令牌成功，获取耗：9.996993 第 3 个任务执行
    2018\-08\-11 00:26:35.920 | pool-1\-thread-4获取令牌成功，获取耗：1.999051 第 4 个任务执行
    2018\-08\-11 00:26:55.920 | pool-1\-thread-5获取令牌成功，获取耗：19.999726 第 5 个任务执行
    2018\-08\-11 00:26:57.920 | pool-1\-thread-6获取令牌成功，获取耗：1.999139 第 6 个任务执行
    2018\-08\-11 00:26:59.920 | pool-1\-thread-7获取令牌成功，获取耗：1.999806 第 7 个任务执行
    2018\-08\-11 00:27:01.919 | pool-1\-thread-8获取令牌成功，获取耗：1.999433 第 8 个任务执行

`acquire`函数主要用于获取 permits 个令牌，并计算需要等待多长时间，进而挂起等待，并将该值返回

一个 RateLimiter 主要定义了发放 permits 的速率。如果没有额外的配置，permits 将以固定的速度分配，单位是每秒多少 permits。默认情况下，Permits 将会被稳定的平缓的发放。

## 预消费能力

从输出结果可以看出，指定每秒放 1 个令牌，RateLimiter 具有预消费的能力：

`acquire 1` 时，并没有任何等待 0.0 秒 直接预消费了 1 个令牌  
`acquire 10`时，由于之前预消费了 1 个令牌，故而等待了 1 秒，之后又预消费了 10 个令牌  
`acquire 2` 时，由于之前预消费了 10 个令牌，故而等待了 10 秒，之后又预消费了 2 个令牌  
`acquire 20` 时，由于之前预消费了 2 个令牌，故而等待了 2 秒，之后又预消费了 20 个令牌  
`acquire 2` 时，由于之前预消费了 20 个令牌，故而等待了 20 秒，之后又预消费了 2 个令牌  
`acquire 2` 时，由于之前预消费了 2 个令牌，故而等待了 2 秒，之后又预消费了 2 个令牌  
`acquire 2` 时 .....

通俗的讲「前人\_挖坑\_后人跳」, 也就说上一次请求获取的 permit 数越多，那么下一次再获取授权时更待的时候会更长，反之，如果上一次获取的少，那么时间向后推移的就少，下一次获得许可的时间更短。可见，都是有代价的。正所谓：要浪漫就要付出代价。马上就七夕了，浪漫的代价可能要花钱啊，单身狗们。

## 令牌桶算法 VS 漏桶算法

**漏桶**

漏桶的出水速度是恒定的，那么意味着如果瞬时大流量的话，将有大部分请求被丢弃掉（也就是所谓的溢出）。

**令牌桶**

生成令牌的速度是恒定的，而请求去拿令牌是没有速度限制的。这意味，面对瞬时大流量，该算法可以在短时间内请求拿到大量令牌，而且拿令牌的过程并不是消耗很大的事情。

## 最后

不论是对于令牌桶拿不到令牌被拒绝，还是漏桶的水满了溢出，都是为了保证大部分流量的正常使用，而牺牲掉了少部分流量，这是合理的，如果因为极少部分流量需要保证的话，那么就可能导致系统达到极限而挂掉，得不偿失。

本文讲的单机的限流，是 JVM 级别的的限流，所有的令牌生成都是在内存中，在分布式环境下不能直接这么用，可用使 redis 限流。

-   作者：鹏磊
-   出处：[http://www.ymq.io/2018/08/011/RateLimiter](https://link.segmentfault.com/?enc=Yo644rqFC8%2BSbKo0450kPg%3D%3D.I4lr2SWRrTZuVneeqYWcPcBMAuTERXPC8R1t%2FoufCBCBMS63jzlQjDy9exAttHD5)
-   版权归作者所有，转载请注明出处
-   Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享

![](https://segmentfault.com/img/remote/1460000012226137?w=510&h=253) 
 [https://segmentfault.com/a/1190000015967922](https://segmentfault.com/a/1190000015967922)
