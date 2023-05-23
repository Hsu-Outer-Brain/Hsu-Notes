# SQLAlchemy使用和踩坑记 - 知乎
[SQLAlchemy 使用和踩坑记 - 知乎](https://zhuanlan.zhihu.com/p/466056973) 

## 1、SQLAlchemy 简介

Python 中，SQLAlchemy 是一个有名的 ORM 工具，和其他工具不同，SQLAlchemy 不再是直接操作 SQL 语句，而是操作 python 对象。在具体实现代码中，将数据库表转化为 Python 类，大大提高了数据库操作的便利性，但同时由于这种转换的存在，使得 SQLAlchemy 的性能不及原生 SQL。

## 2、SQLAlchemy 原理

官网原理图如下：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-23%2016-24-34/edcac8e8-6dc8-4296-adea-ab0c8235d37b.jpeg?raw=true)

如上可见，SQLAlchemy 分为三层：

-   SQLAlchemy ORM 层：把数据的 schema 转化成 Python 类
-   SQLAlchemy Core 层：新建 engine 和连接池，执行数据库的增、删、改、查等操作
-   DBAPI 层：这一层不属于 SQLAlchemy 代码，是 SQLAlchemy 使用的数据库驱动提供的 API

## 3、SQLAlchemy 概念

-   Engine：连接，服务和数据库直接建立的物理连接
-   Session：会话，记录通信双方从开始通信到通信结束期间的一个上下文（Context），位于服务器端的内存：记录了本次连接的客户端机器、应用程序、用户登录等信息。一个 Engine 可有多个 Session，也可没有 Session，各个 Session 互不相关
-   Model：模型，在 Python 中操作的类对象，对应于数据库的表
-   Column：列，再 Python 中类的字段，对应数据库中的列

## 4、SQLAlchemy 简单使用

### 安装

### 创建连接 &&Session

```python3
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import NullPool


db_addr = "mysql+mysqlconnector://user:password@host:port/db_name?auth_plugin=mysql_native_password&autocommit=true"
engine = create_engine(db_addr, poolclass=NullPool)
session = sessionmaker(bind=engine)()
```

SQLAlchemy 需要使用其他的 python 的数据库驱动来连接数据库，python 的数据库驱动有

-   MySQL-python：只能在 python2 使用，底层使用 C 实现，用 python 包装成 python package，只支持 myqsl 3.23- 5.5 的版本和 python 2.4-2.7 版本，安装和使用如下

```text
安装：pip install MySQL-python

使用：
import MySQLdb
```

-   mysql-connector-python：同时支持 python2 和 python3。纯 python 实现，性能没有 MySQL-python 好，但支持连接池，线程安全，可靠性高

```text
安装：pip install mysql-connector-python

使用
import mysql.connector

db_addr = "mysql+mysqlconnector:XXXX" # 使用mysql-connector-python时，指定使用mysqlconnector
```

-   pymysql：支持 python3，纯 python 实现，但不支持线程安全

```text
安装：pip install PyMySQL

使用：
import pymysql

db_addr = "mysql+pymysql" # 使用PyMySQL时，指定使用pymysql
```

### 创建实例对象（Modle）

Python 的模型对象就对应数据库的一张表，类的属性就对应数据表的一个列

```text
from sqlalchemy import Column, String, Text, Integer, TIMESTAMP, FLOAT, BigInteger
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class Job(Base):
    __tablename__ = 'job'

    id = Column(Integer, primary_key=True, autoincrement=True)
    job_status = Column(Integer)
    job_name = Column(Text)

    def __repr__(self):
        return "id:%d,name:%s" % (self.id, self.job_name)
```

如上对应的表的结构为

```text
CREATE TABLE IF NOT EXISTS `job` (
    `id` INT NOT NULL AUTO_INCREMENT,
    `job_status` INT NOT NULL,
    `job_name` VARCHAR(255),
    PRIMARY KEY (`id`)
)ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 数据操作

SQLAlchemy 对数据库的操作都是通过 Session 进行的，Session 的创建在 创建连接 &&Session 部分，Session 的除增、删、改、查操作外，还包括如下基本操作：

-   commit：事务提交到数据库
-   flush：预提交，提交到数据库文件，还未写入到数据库文件中
-   rollback：session 回滚
-   close：session 关闭

**新增数据**

```text
job = Job()
job.status = 0
job.name = "test"
session.add(job)
session.commit()
```

注：如果只是 add 到 session，没有 commit 则这次 add 操作没有真正落实到数据库中

**查询数据**

```text
jobs = session.query(Job).filter_by(id==1).all()
print(jobs)
```

注：

1.  筛选可使用 filter 和 filter_by，用法类似，其中 filter_by 可以支持组合查询，filter 不支持组合查询，实现组合查询要联系使用 filter 来实现
2.  筛选操作后，要执行 all() 查询全部数据，或者 first() 查询第一条数据，此时才是真正去查询操作数据库

**修改数据**

```text
job = session.query(Job).filter_by(id==1).first()
job.udpate({"name":"hello world"}) #使用update操作
session.commit()
```

注：update 操作也要使用 commit 后才是真实操作数据库

**删除数据**

```text
session.query(Job).filter_by(id==1).delete()
session.commit()
```

注：如上 delete 操作会把所以查询出的数据删除

## 5、SQLAlchemy 多个坑

**1、查询结果和数据库数据不一致**

现象：数据 id==1 的数据的 name==“test”，事务 A 修改 id==1 的数据 name 改为 “hello world” 并 commit，事务 B 在事务 A 执行玩之前 查询 id==1 的 name==“test”，等事务 A 执行完成，再从查询 id==1 的 name 还是“test”，而不是“hello world”

原因：由于数据事务机制（参考：[彻底搞懂 MySQL 事务的隔离级别 - 阿里云开发者社区](https://link.zhihu.com/?target=https%3A//developer.aliyun.com/article/743691)），导致事务 B 在第一次查询的时候，真正请求数据库进行查询，但后面的查询直接读取的已经缓存在内存中的数据，而没有真正去数据库中查询

解决方案：

-   方案一：在每次 query 之后执行 commit
-   方案二：把数据库的事情机制降级，不推荐，这样很容易导致读到脏数据、读取数据不一致等问题

```text
engine = create_engine(
    "XXXX",
    isolation_level="READ UNCOMMITTED"
)
```

**2、长时间未请求连接自动断开**

现象：长时间服务端没有连接数据库，数据库连接自动断开

原因：1、sqlalchemy 在 create_engine 时，使用连接池并没有指定连接池回收时间，则连接池的连接不会自动被回收，并默认使用 QueuePool 进行连接池管理，调用 session.close()，不会断开连接，2、数据库，例如 mysql 会设置一个 wait_timeout（默认 8 个小时），当连接空闲 8 个小时，则自动断开

解决方案：

-   方案一：修改数据库的设置 wait_timeout（不推荐）
-   方案二：新建连接池时，设置连接回收时间，使这个值小于 wait_timeout，sqlchemy 的 create_engine 一些重要参数如下：

```text
create_engine重要参数：
  pool_size：连接数，采用了惰性思想，例如：pool_size=10，如果项目中只使用了5个，则连接池中的连接数，只有5个，但当项目同时使用了10个连接，则后续连接池中的连接数为10个
  max_overflow：超出连接数时，允许再新建的连接数，例如：pool_size=10，max_overflow=8，最大连接数18个，但其中8个不在使用时，直接回收，连接池中的连接数为10个
  pool_timeout：等待可用连接时间，超时则报错，默认为30秒
  pool_recycle：连接生存时长，超过则该连接被回收，再生存新连接，可把这个值改成小于wait_timeout；设置-1时，则不回收连接
```

-   方案三：不使用链接池，在 create_engine 时指定连接池为 NullPool，则使用 session.close() 后断开数据库链接

```text
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import NullPool


class DbHandler(object):
    def __init__(self):
        self.db_addr = "XXXXXX"

    def create_session(self):
        engine = create_engine(self.db_addr, poolclass=NullPool)
        session = sessionmaker(bind=engine)()
        return session
```

**3、全局 session 在多进程下被断开**

想象：sqlalchemy 连接池，经常报错：mysql server has gone away

原因：父进程创建子进程，子进程会使用父进程的数据连接，当子进程执行完成，断开数据库的连接，则全局 session 和连接被释放，后续要使用连接数据库时，则报错

解决方案：多进程中，尽量没有进程新建连接和 session

## 6、测试视角

一方面对于测开的同学来说，写的代码也经常要和数据打交道，注意上面的坑，另一个方面，对于单纯测试的同学来说，了解开发的代码实现，如果使用了 sqlalchemy 或者类似的 ORM 组件，在代码修改时，也可能好的评估影响面，针对性的测试

## 7、参考

[https://docs.sqlalchemy.org/en/14/intro.html](https://link.zhihu.com/?target=https%3A//docs.sqlalchemy.org/en/14/intro.html)

[彻底搞懂 MySQL 事务的隔离级别 - 阿里云开发者社区](https://link.zhihu.com/?target=https%3A//developer.aliyun.com/article/743691)

[SQLAlchemy 长时间未请求数据库连接断开 - KubeLab](https://link.zhihu.com/?target=https%3A//www.cloudcared.cn/2949.html)

[Python：SQLalchemy 使用](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/f5b4b3644f94)

写在最后：水平有限，欢迎批评指正，如有侵权，联系速删～～
