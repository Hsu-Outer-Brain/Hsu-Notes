# 使用Python装饰器构造一个LRU本地缓存 - 墨天轮
本地缓存指的是在代码运行中动态分配的内存，一般存在于当前进程地址空间的**堆内存**中，如果不人为的显式释放，将会一直存在进程中。

本地缓存和 Memcached 或 Redis 这样的远程缓存区别是，**本地缓存有更高的执行效率，没有网络传输的消耗，缺点是没有自动过期的机制，存放的数据量也有限。** 

本地缓存可以显著提高代码的执行效率，比如，看下面这样一段代码：

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20220216_b5c4daca-8ef4-11ec-b108-fa163eb4f6be.png)

如果这个函数被频繁调用（比如放在循环中），每次都需要去查询数据库，效率必然很低，那么可以使用本地缓存来优化：

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20220216_b5ff62ee-8ef4-11ec-b108-fa163eb4f6be.png)

这样，已经查询过一次的用户信息将会存在于本地缓存 cache 中，而不用每次去数据库查询。不过，这样的本地缓存有两个缺点：

-   如果不重启服务，缓存将会一直存在于进程地址空间中，永不过期。如果在其他地方更新了用户信息，那么本地缓存中取到的将一直是旧的数据，导致数据不一致。
-   缓存大小没有限制。假设有 100w 个用户获取了用户信息，那么 cache 中将会缓存 100w 个用户信息，直接就导致内存泄露了，进程有可能被 OOM (Out Of Memery) 杀死。

为了解决这两个问题，我们来构造一个专门用于本地缓存的 Python 装饰器，提供两个功能：

-   支持缓存数据过期
-   内存大小有限制，超过后使用 LRU 缓存淘汰算法对缓存数据进行清理

具体实现看如下代码：

缓存工具类 mycache.py 

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20220216_b6356006-8ef4-11ec-b108-fa163eb4f6be.png)

LRUCache 类是标准的 LRU 缓存淘汰算法的实现， Cache 类先实例化了一个 LRU 对象，然后实现了 get 和 set 两个静态方法。

下面来应用这个工具类，在公共模块 common.py 中写一个装饰器：

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20220216_b6840fbc-8ef4-11ec-b108-fa163eb4f6be.png)

这个装饰器需要传入一个自定义的业务类型，方便构造与业务关联的 key，并且被装饰的函数只支持  args 类型的参数，不支持 kwargs 参数，因为 kwargs 类型的参数不太适合构造 key 值。

好了，到目前为止，本地缓存装饰器就构造完成了，用法如下：

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20220216_b6b3bad2-8ef4-11ec-b108-fa163eb4f6be.png)

以后这段代码就不怕在循环中调用了，你不妨写一段 for 循环测试一下使用缓存装饰器和不使用缓存装饰器的性能。  

* * *

学会了这个高级操作，写出的代码逼格瞬间就提高了。感觉简历中又多了一个可以吹牛逼的东西![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20220216_b6e70bb2-8ef4-11ec-b108-fa163eb4f6be.png)

好东西要分享，要不下面点个在看吧 ^\_^ 
 [https://www.modb.pro/db/326590](https://www.modb.pro/db/326590)
