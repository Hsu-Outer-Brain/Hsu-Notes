# 杂项之pymysql连接池 - 随风飘雪012 - 博客园
## 本节内容

1.  本文的诞生
2.  连接池及单例模式
3.  多线程提升
4.  协程提升
5.  后记

## 1. 本文的诞生

由于前几天接触了 pymysql，在测试数据过程中，使用普通的 pymysql 插入 100W 条数据，消耗时间很漫长，实测 990s 也就是 16.5 分钟左右才能插完，于是，脑海中诞生了一个想法，能不能造出一个连接池出来，提升数据呢？就像一根管道太小，那就多加几根管道看效果如何呢？于是。。。前前后后折腾了将近一天时间，就有了本文的诞生。。。

## 2. 连接池及单例模式

先说单例模式吧，为什么要在这使用单例模式呢？使用单例模式能够节省资源。

其实单例模式没有什么神秘的，简单的单例模式实现其实就是在类里面定义一个变量，再定义一个类方法，这个类方法用来为调用者提供这个类的实例化对象。（ps: 个人对单例模式的一点浅薄理解...）

那么连接池是怎么回事呢？原来使用 pymysql 创建一个 conn 对象的时候，就已经和 mysql 之间创建了一个 tcp 的长连接，只要不调用这个对象的 close 方法，这个长连接就不会断开，这样，我们创建了一组 conn 对象，并将这些 conn 对象放到队列里面去，这个队列现在就是一个连接池了。

现在，我们先用一个连接，往数据库中插入 100W 条数据，下面是源码：

![](https://common.cnblogs.com/images/copycode.gif)

 1 import pymysql 2 import time 3 start=time.time()
 4 conn = pymysql.connect(host="192.168.10.103",port=3306,user="root",passwd="123456",db="sql_example",charset="utf8")
 5 conn.autocommit(True)  # 设置自动 commit
 6 cursor = conn.cursor(cursor=pymysql.cursors.DictCursor)  # 设置返回的结果集用字典来表示，默认是元祖
 7 data=(("男",i,"张小凡 %s" %i) for i in range(1000000))  # 伪造数据，data 是个生成器
 8 cursor.executemany("insert into tb1(gender,class_id,sname) values(%s,%s,%s)",data)  # 可以使用 executemany 执行多条 sql
 9 # conn.commit()
10 cursor.close() 11 conn.close() 12 print("totol time:",time.time()-start)

![](https://common.cnblogs.com/images/copycode.gif)

执行结果为：

totol time: 978.7649309635162

## 3. 多线程提升

使用多线程，在启动时创建一组线程，每个线程去连接池里面获取一个连接，然后插入数据，这样将会大大提升执行 sql 的速度，下面是使用多线程实现的连接池源码：

![](https://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)

![](https://common.cnblogs.com/images/copycode.gif)

 1 from gevent import monkey 2 monkey.patch_all()
 3 
 4 import threading 5 
 6 import pymysql 7 from queue import Queue 8 import time 9 
 10 class Exec_db: 11 
 12     \_\_v\\=None
 13 
 14     def \_\_init\_\_(self,host=None,port=None,user=None,passwd=None,db=None,charset=None,maxconn=5):
 15         self.host,self.port,self.user,self.passwd,self.db,self.charset=host,port,user,passwd,db,charset
 16         self.maxconn=maxconn
 17         self.pool=Queue(maxconn)
 18         for i in range(maxconn): 19             try:
 20                 conn=pymysql.connect(host=self.host,port=self.port,user=self.user,passwd=self.passwd,db=self.db,charset=self.charset)
 21                 conn.autocommit(True)
 22                 # self.cursor=self.conn.cursor(cursor=pymysql.cursors.DictCursor)
 23                 self.pool.put(conn)
 24             except Exception as e: 25                 raise IOError(e) 26 
 27     @classmethod
 28     def get_instance(cls,\*args,\*\*kwargs):
 29         if cls.\_\_v:
 30             return cls.\_\_v
 31         else:
 32             cls.\_\_v\\=Exec_db(\*args,\*\*kwargs)
 33             return cls.\_\_v
 34 
 35     def exec_sql(self,sql,operation=None):
 36         """
 37             执行无返回结果集的 sql，主要有 insert update delete
 38         """
 39         try:
 40             conn=self.pool.get()
 41             cursor=conn.cursor(cursor=pymysql.cursors.DictCursor)
 42             response=cursor.execute(sql,operation) if operation else cursor.execute(sql) 43         except Exception as e: 44             print(e)
 45             cursor.close()
 46             self.pool.put(conn)
 47             return None 48         else:
 49             cursor.close()
 50             self.pool.put(conn)
 51             return response 52 
 53 
 54     def exec_sql_feach(self,sql,operation=None):
 55         """
 56             执行有返回结果集的 sql, 主要是 select
 57         """
 58         try:
 59             conn=self.pool.get()
 60             cursor=conn.cursor(cursor=pymysql.cursors.DictCursor)
 61             response=cursor.execute(sql,operation) if operation else cursor.execute(sql) 62         except Exception as e: 63             print(e)
 64             cursor.close()
 65             self.pool.put(conn)
 66             return None,None 67         else:
 68             data=cursor.fetchall()
 69             cursor.close()
 70             self.pool.put(conn)
 71             return response,data 72 
 73     def exec_sql_many(self,sql,operation=None):
 74         """
 75             执行多个 sql，主要是 insert into 多条数据的时候
 76         """
 77         try:
 78             conn=self.pool.get()
 79             cursor=conn.cursor(cursor=pymysql.cursors.DictCursor)
 80             response=cursor.executemany(sql,operation) if operation else cursor.executemany(sql) 81         except Exception as e: 82             print(e)
 83             cursor.close()
 84             self.pool.put(conn)
 85         else:
 86             cursor.close()
 87             self.pool.put(conn)
 88             return response 89 
 90     def close_conn(self): 91         for i in range(self.maxconn): 92             self.pool.get().close()
 93 
 94 obj=Exec_db.get_instance(host="192.168.10.103",port=3306,user="root",passwd="012615",db="sql_example",charset="utf8",maxconn=10)
 95 
 96 def test_func(num): 97     data=(("男",i,"张小凡 %s" %i) for i in range(num)) 98     sql="insert into tb1(gender,class_id,sname) values(%s,%s,%s)"
 99     print(obj.exec_sql_many(sql,data)) 100 
101 job_list=\[] 102 for i in range(10): 103     t=threading.Thread(target=test_func,args=(100000,)) 104 t.start() 105 job_list.append(t) 106 for j in job_list: 107 j.join() 108 obj.close_conn() 109 print("totol time:",time.time()-start)

![](https://common.cnblogs.com/images/copycode.gif)

开启 10 个连接池插入 100W 数据的时间：

totol time: 242.81142950057983

开启 50 个连接池插入 100W 数据的时间：

totol time: 192.49499201774597

开启 100 个线程池插入 100W 数据的时间：

totol time: 191.73923873901367

## 4. 协程提升

使用协程的话，在 I/O 阻塞时，将会切换到其他任务去执行，这样理论上来说消耗的资源应该会比多线程要少。下面是协程实现的连接池源代码：

![](https://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)

![](https://common.cnblogs.com/images/copycode.gif)

 1 from gevent import monkey 2 monkey.patch_all()
 3 import gevent 4 
 5 import pymysql 6 from queue import Queue 7 import time 8 
 9 class Exec_db: 10 
 11     \_\_v\\=None
 12 
 13     def \_\_init\_\_(self,host=None,port=None,user=None,passwd=None,db=None,charset=None,maxconn=5):
 14         self.host,self.port,self.user,self.passwd,self.db,self.charset=host,port,user,passwd,db,charset
 15         self.maxconn=maxconn
 16         self.pool=Queue(maxconn)
 17         for i in range(maxconn): 18             try:
 19                 conn=pymysql.connect(host=self.host,port=self.port,user=self.user,passwd=self.passwd,db=self.db,charset=self.charset)
 20                 conn.autocommit(True)
 21                 # self.cursor=self.conn.cursor(cursor=pymysql.cursors.DictCursor)
 22                 self.pool.put(conn)
 23             except Exception as e: 24                 raise IOError(e) 25 
 26     @classmethod
 27     def get_instance(cls,\*args,\*\*kwargs):
 28         if cls.\_\_v:
 29             return cls.\_\_v
 30         else:
 31             cls.\_\_v\\=Exec_db(\*args,\*\*kwargs)
 32             return cls.\_\_v
 33 
 34     def exec_sql(self,sql,operation=None):
 35         """
 36             执行无返回结果集的 sql，主要有 insert update delete
 37         """
 38         try:
 39             conn=self.pool.get()
 40             cursor=conn.cursor(cursor=pymysql.cursors.DictCursor)
 41             response=cursor.execute(sql,operation) if operation else cursor.execute(sql) 42         except Exception as e: 43             print(e)
 44             cursor.close()
 45             self.pool.put(conn)
 46             return None 47         else:
 48             cursor.close()
 49             self.pool.put(conn)
 50             return response 51 
 52 
 53     def exec_sql_feach(self,sql,operation=None):
 54         """
 55             执行有返回结果集的 sql, 主要是 select
 56         """
 57         try:
 58             conn=self.pool.get()
 59             cursor=conn.cursor(cursor=pymysql.cursors.DictCursor)
 60             response=cursor.execute(sql,operation) if operation else cursor.execute(sql) 61         except Exception as e: 62             print(e)
 63             cursor.close()
 64             self.pool.put(conn)
 65             return None,None 66         else:
 67             data=cursor.fetchall()
 68             cursor.close()
 69             self.pool.put(conn)
 70             return response,data 71 
 72     def exec_sql_many(self,sql,operation=None):
 73         """
 74             执行多个 sql，主要是 insert into 多条数据的时候
 75         """
 76         try:
 77             conn=self.pool.get()
 78             cursor=conn.cursor(cursor=pymysql.cursors.DictCursor)
 79             response=cursor.executemany(sql,operation) if operation else cursor.executemany(sql) 80         except Exception as e: 81             print(e)
 82             cursor.close()
 83             self.pool.put(conn)
 84         else:
 85             cursor.close()
 86             self.pool.put(conn)
 87             return response 88 
 89     def close_conn(self): 90         for i in range(self.maxconn): 91             self.pool.get().close()
 92 
 93 obj=Exec_db.get_instance(host="192.168.10.103",port=3306,user="root",passwd="123456",db="sql_example",charset="utf8",maxconn=10)
 94 
 95 def test_func(num): 96     data=(("男",i,"张小凡 %s" %i) for i in range(num)) 97     sql="insert into tb1(gender,class_id,sname) values(%s,%s,%s)"
 98     print(obj.exec_sql_many(sql,data))
 99 
100 start=time.time() 101 job_list=\[] 102 for i in range(10): 103     job_list.append(gevent.spawn(test_func,100000)) 104 
105 gevent.joinall(job_list) 106 
107 obj.close_conn() 108 
109 print("totol time:",time.time()-start)

![](https://common.cnblogs.com/images/copycode.gif)

开启 10 个连接池插入 100W 数据的时间：

totol time: 240.16892313957214

开启 50 个连接池插入 100W 数据的时间：

totol time: 202.82087111473083

开启 100 个线程池插入 100W 数据的时间：

totol time: 196.1710569858551

## 5. 后记

统计结果如下：

单线程一个连接使用时间：978.76s

|   | 10 个连接池 | 50 个连接池 | 100 个连接池 |
| 多线程版 | 242.81s | 192.49s | 191.74s |
| 协程版 | 240.17s | 202.82s | 196.17s |

通过统计结果显示, 通过协程和多线程操作连接池插入相同数据，相对一个连接提升速度明显，但是在将连接池开到 50 以及 100 时，性能提升并没有想象中那么大，这时候，瓶颈已经不在网络 I/O 上了，而在数据库中，mysql 在大量连接写入数据时，也会有锁的产生，这时候就需要优化数据库的相关设置了。

在对比中显示多线程利用线程池和协程利用线程池的性能差不多，但是多线程的开销比协程要大。

和大神讨论过，在项目开发中需要考虑到不同情况使用不同的技术，多线程适合使用在连接量较大，但每个连接处理时间很短的情况下，而协程适用于处理大量连接，但同时活跃的链接比较少，并且每个连接的时间量比较大的情况下。

在实际生产应用中，创建连接池可以按需分配，当连接不够用时，在连接池没达到上限的情况下，在连接池里面加入新的连接，在连接池比较空闲的情况下，关闭一些连接，实现这一个操作的原理是通过 queue 里面的超时时间来控制，当等待时间超过了超时时间时，说明连接不够用了，需要加入新的连接。 
 [https://www.cnblogs.com/huxianglin/p/6011535.html](https://www.cnblogs.com/huxianglin/p/6011535.html)
