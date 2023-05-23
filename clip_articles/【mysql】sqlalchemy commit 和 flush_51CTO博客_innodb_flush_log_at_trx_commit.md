# 【mysql】sqlalchemy commit 和 flush_51CTO博客_innodb_flush_log_at_trx_commit
[【mysql】sqlalchemy commit 和 flush_51CTO 博客\_innodb_flush_log_at_trx_commit](https://blog.51cto.com/u_15873544/5844059) 

 今天看到了 commit 和 flush 函数，想要弄清楚区别。

先看下对象的状态。总共 5 个，这里只谈 3 个。

transitant：刚 new 出来的对象，没有和 session 或者 orm 框架产生关联。

pending：transitant 的对象调用 add 后，就会变为 pending，加入了 orm 框架的监管范围。

persistant：调用 flush 以后就会变味 persistant，也就是被写到了数据库中。

查询官网后，发现：

flush 会把更改提交到数据库，commit 会默认调用 flush，然后标志这个事务的提交，也就是事务执行完毕。如果只调用 flush，那么更新虽然可以被写入数据库，但是事务是不完整的，没有提交。由于事务隔离型的存在，可能其他的事务是无法看到这次更新操作的。只有调用了 commit，才能被看成是事务完整的执行完毕。

为了验证 flush 确实把数据写入了数据库，进行测试：

```


from sqlalchemy import *  
from sqlalchemy.orm import *  
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()  
class People(Base):

    \_\_tablename\_\_ = 'people'

    id = Column(Integer, primary_key=True, unique=True, index=True, autoincrement=True)  
    name = Column(String(100))  
    age = Column(Integer, default=0)

    def \_\_repr\_\_(self):  
        return "<People(id={}, name={}, age={}')>".format(self.id, self.name, self.age)

engine = create_engine('mysql+mysqlconnector://root:@localhost:3306/test', echo=True)  
metadata = MetaData(engine)

people = Table('people', metadata,  
    Column('id', Integer, primary_key=True),  
    Column('name', String(100)),  
    Column('age', Integer))  
metadata.create_all(engine)

*   1.
*   2.
*   3.
*   4.
*   5.
*   6.
*   7.
*   8.
*   9.
*   10.
*   11.
*   12.
*   13.
*   14.
*   15.
*   16.
*   17.
*   18.
*   19.
*   20.
*   21.
*   22.
*   23.
*   24.


```

models.py 执行建表语句。  

然后我在数据库里添加了一条记录，name=“ly”，age=25，没有贴出这段代码。

```


from models import People  
import sqlalchemy.orm as o  
from sqlalchemy import create_engine  
import time  
from sqlalchemy import inspect

engine = create\_engine('mysql+mysqlconnector://root:@localhost:3306/test', isolation\_level='READ_UNCOMMITTED')  
#engine = create_engine('mysql+mysqlconnector://root:@localhost:3306/test')

Session = o.sessionmaker(bind=engine)  
s1 = Session()

p1 = People(name='ly1', age=25)

s1.add(p1)  
s1.flush()  
insp = inspect(p1)  
print(insp.transient)  
print(insp.persistent)  
print(insp.pending)

#s1.commit()

time.sleep(1000000)

*   1.
*   2.
*   3.
*   4.
*   5.
*   6.
*   7.
*   8.
*   9.
*   10.
*   11.
*   12.
*   13.
*   14.
*   15.
*   16.
*   17.
*   18.
*   19.
*   20.
*   21.
*   22.
*   23.
*   24.
*   25.


```

然后 add.py 插入另外一条数据，测试，插入以后，没有提交，只是 flush，然后进程停滞在这里。  

另外开一个进程，读取数据库，

```


from models import People  
import sqlalchemy.orm as o  
from sqlalchemy import create_engine  
import time  
from sqlalchemy import inspect

engine = create\_engine('mysql+mysqlconnector://root:@localhost:3306/test', isolation\_level='READ_UNCOMMITTED')  
#engine = create_engine('mysql+mysqlconnector://root:@localhost:3306/test')

Session = o.sessionmaker(bind=engine)  
s1 = Session()

r1 = s1.query(People).all()

print(r1)

*   1.
*   2.
*   3.
*   4.
*   5.
*   6.
*   7.
*   8.
*   9.
*   10.
*   11.
*   12.
*   13.
*   14.
*   15.
*   16.


```

最后可以读到两条数据。  

\[&lt;People(id=1, name=ly, age=25')>, &lt;People(id=31, name=ly1, age=25')>]  

而且，之前打印了 p1 的对象状态，在 flush 之后是 persist 的，说明 flush 确实是写入数据库了。

这个实验需要注意数据库隔离级别，这里设置为了读未提交，否则其他的事务是读不到的，因为之前的事务只有 flush 没有 commit。比如这时开一个 mysql 的命令行，就无法读取到添加的数据，因为 mysql 的默认是重复读。

se

关于这个例子继续延伸下：如果读取的进程在写入的进程执行完毕后才开始，比如 sleep 完了。结果会是什么。

结果是无法读到 add 的数据了，这是因为之前的事务结束了，而且写入了一条数据，但是该数据的写入是放在一个事务中的，事务并没有 commit 操作，根据事务的原子性特征，该事务要么全做，要么全不做，由于这个事务不完整，所以只能全不做，因此之前的添加不会生效的。

另外补充一个细节，在同一个事务中，可以读到 pending 状态的对象，也就是只 add，再 query，可以读到，这是为什么？add 不是没有写入数据库吗？怎么可以读到？这是因为 query 的时候，默认是要 autoflush 的，也就是会自动把当前事物 add 的数据 flush 到数据库中，所以就会读到了。如果设置 session 的 autoflush=Flase，那么只 add 然后 query，就读不到 add 的数据了。这个细节要注意。

结论就是，flush 会写入数据库，commit 会调用 flush，并且表示事务结束。

至于事务到底会读取到什么样的数据，要看数据库隔离级别。

补充：每一次连接就取得了连接池中的一个连接，也就是 conncetion 的构建，就会建立一个 tcp，在这一次的连接中，可能会执行多个事务，事务不会影响到连接。
