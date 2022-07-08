# Python缓存神奇库cacheout全解，优于内存的性能 - 腾讯云开发者社区-腾讯云
**python 的缓存库 (cacheout)**

项目: [https://github.com/dgilland/cacheout](https://github.com/dgilland/cacheout)

文档地址: [https://cacheout.readthedocs.io](https://cacheout.readthedocs.io/)

PyPI(下载链接): [https://pypi.python.org/pypi/cacheout/](https://pypi.python.org/pypi/cacheout/)

TravisCI(下载链接): [https://travis-ci.org/dgilland/cacheout](https://travis-ci.org/dgilland/cacheout)

特性：

后端使用字典进行缓存

使用缓存管理轻松访问多个缓存对象

当使用模块级缓存对象，重构运行时的缓存设置

最大缓存大小限制

默认的缓存时间设置以及缓存项自定义存活时间

批量的设置、获取、删除操作

线程安全

**多种缓存机制的实现：** 

FIFO(先进先出)

LIFO(后进先出)

LRU (最近最少使用机制)

MRU (最近最多使用机制)

LFU (最小频率使用机制)

RR (随机替换机制)

解释一下，避免产生混淆，我在使用时就产生的歧义，后来通过小 demo 证实的!

LRU 是删除最近最少使用的，保留最近最多使用的。

线路图：

层级缓存 (多层级缓存)

支持缓存事件监听

获取缓存对象时的常规表示方法

获取缓存对象不存在时的回调处理支持

统计缓存

版本要求：

Python >= 3.4

**安装：** 

通过创建一个缓存对象来开始了解：

    # from cacheout import Cache# 如果选择LFUCache 就导入即可
    from cacheout import LFUCache
    cache = LFUCache()

默认的缓存的大小为 256，默认存活时间是关闭的，这些属性可以如下设置：

    cache = Cache(maxsize=256, ttl=0, timer=time.time, default=None) 

设置一个缓存可以通过 cache.set():

获取缓存键的值通过：cache.get():

    ret = cache.get(1)
     # 'foobar'

可以为每个键值对设置存活过期时间：

    cache.set(3, {'data': {}}, ttl=1)
    assert cache.get(3) == {'data': {}}
    time.sleep(1)
    assert cache.get(3) is None

为缓存函数提供了键值对的存活时间：

    @cache.memoize()
    def func(a, b):   
        pass

函数解除缓存：

    @cache.memoize()
    def func(a, b):   
       pass

    func.uncached(1, 2)

复制机制：

    assert cache.copy() == {1: 'foobar', 2: ('foo', 'bar', 'baz')}

删除缓存中的一个键值对

    cache.delete(1)
    assert cache.get(1) is None

清除整个缓存：

    cache.clear()
    assert len(cache) == 0

为 get、set、delete 设置了批量方法：

    # 设置
    cache.set_many({'a': 1, 'b': 2, 'c': 3})
    # 获取
    assert cache.get_many(['a', 'b', 'c']) 
    # 删除cache.delete_many(['a', 'b', 'c'])
    assert cache.count()

重置已经初始化的缓存对象

    cache.configure(maxsize=1000, ttl=5 * 60)

通过 cache.keys(), cache.values(), and cache.items() 获取所有的键、值、以及键值对：

    cache.set_many({'a': 1, 'b': 2, 'c': 3})
    assert list(cache.keys()) == ['a', 'b', 'c']
    assert list(cache.values()) == [1, 2, 3]
    assert list(cache.items()) == [('a', 1), ('b', 2), ('c', 3)]

迭代整个缓存的键:

    for key in cache:
        print(key, cache.get(key))
        # 'a' 1
        # 'b' 2
        # 'c' 3

检测键是否还存在于缓存中通过 cache.has() and key in cache 方法：

    assert cache.has('a')
    assert 'a' in cache

通过使用 CacheManager 来管理多个缓存对象：

    from cacheout import CacheManager, LFUCache

    # 设置多个缓存， 并设置缓存机制
    cacheman = CacheManager({'a': {'maxsize': 100},
                             'b': {'maxsize': 200, 'ttl': 900},
                             'c':{} },
                            cache_class= LFUCache
                            )

    cacheman['a'].set('key1', 'value1')
    value = cacheman['a'].get('key')

    cacheman['b'].set('key2', 'value2')
    assert cacheman['b'].maxsize == 200
    assert cacheman['b'].ttl == 900

    cacheman['c'].set('key3', 'value3')

    cacheman.clear_all()
    for name, cache in cacheman:
        assert name in cacheman
        assert len(cache) == 0

总结： 1、建立在内存上，其处理速度由于 redis，等同于内存 2、可以设置过期时间，以及缓存容量大小，控制占用内存的大小 3、可以选择适合自己的机制，进一步优化优先策略，优于内存 
 [https://cloud.tencent.com/developer/article/1380994](https://cloud.tencent.com/developer/article/1380994)
