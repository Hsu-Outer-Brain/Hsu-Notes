# (6条消息) SQLAlchemy技术文档（中文版）（全）_sqlalchemy中文文档_Lotfee的博客-CSDN博客
[(6 条消息) SQLAlchemy 技术文档（中文版）（全）\_sqlalchemy 中文文档\_Lotfee 的博客 - CSDN 博客](https://blog.csdn.net/Lotfee/article/details/57406450) 

 原文链接：[http://www.cnblogs.com/iwangzc/p/4112078.html（感谢作者的分享）](http://www.cnblogs.com/iwangzc/p/4112078.html（感谢作者的分享）)  

sqlalchemy 官方文档：[http://docs.sqlalchemy.org/en/latest/contents.html](http://docs.sqlalchemy.org/en/latest/contents.html)

1\. 版本检查

    import sqlalchemy

```
sqlalchemy.\_\_version\_\_ 

```

2\. 连接

    from sqlalchemy import create_engine

    engine = create_engine('sqlite:///:memory:',echo=True)

echo 参数为 True 时，会显示每条执行的 SQL 语句，可以关闭。create_engine() 返回一个 Engine 的实例，并且它表示通过[数据库](http://lib.csdn.net/base/mysql "MySQL 知识库")语法处理细节的核心接口，在这种情况下，数据库语法将会被解释称[Python](http://lib.csdn.net/base/python "Python 知识库")的类方法。

3\. 声明映像

当使用 ORM【1】时，构造进程首先描述数据库的表，然后定义我们用来映射那些表的类。在现版本的 SQLAlchemy 中，这两个任务通常一起执行，通过使用 Declarative 方法，我们可以创建一些包含描述要被映射的实际数据库表的准则的映射类。

使用 Declarative 方法定义的映射类依据一个基类，这个基类是维系类和数据表关系的目录——我们所说的 Declarative base class。在一个普通的模块入口中，应用通常只需要有一个 base 的实例。我们通过 declarative_base() 功能创建一个基类：

    from sqlalchemy.ext.declarativeimportdeclarative_base

    Base = declarative_base()

有了这个 base，我们可以依据这个 base 定义任意数量的映射类。一个简单的 user 例子：

    from sqlalchemy import Column, Integer, String

    class User(Base):

    \_\_tablename\_\_= 'users'

    id= Column(Integer, primary_key=True)

    name = Column(String)

用 Declarative 构造的一个类至少需要一个\_\_tablename\_\_属性，一个主键行。

4\. 构造模式（项目中没用到）

5\. 创建映射类的实例

    ed_user = User(name='ed',fullname='Ed Jones', password='edspassword')

6\. 创建会话

现在我们已经准备毫和数据库开始会话了。ORM 通过 Session 与数据库建立连接的。当应用第一次载入时，我们定义一个 Session 类（声明 create_engine() 的同时），这个 Session 类为新的 Session 对象提供工厂服务。

    from sqlalchemy.orm import sessionmaker

    Session = sessionmaker(bind=engine)

这个定制的 Session 类会创建绑定到数据库的 Session 对象。如果需要和数据库建立连接，只需要实例化一个 Session：

    session = Session()

虽然上面的 Session 已经和数据库引擎 Engine 关联，但是还没有打开任何连接。当它第一次被使用时，就会从 Engine 维护的一个连接池中检索是否存在连接，如果存在便会保持连接知道我们提交所有更改并且 / 或者关闭 session 对象。

7\. 添加新对象（简略）

    ed_user = User(name='ed', fullname='Ed Jones', password='edspassword')

    session.add(ed_user)

至此，我们可以认为，新添加的这个对象实例仍在等待中；ed_user 对象现在并不代表数据库中的一行数据。直到使用 flush 进程，Session 才会让 SQL 保持连接。如果查询这条数据的话，所有等待信息会被第一时间刷新，查询结果也会立即发行。

    session.commit()

通过 commit() 可以提交所有剩余的更改到数据库。

8\. 回滚

    session.rollback()

9\. 查询

通过 Session 的 query() 方法创建一个查询对象。这个函数的参数数量是可变的，参数可以是任何类或者是类的描述的集合。下面是一个迭代输出 User 类的例子：

    for instance in session.query(User).order_by(User.id): 

    print instance.name,instance.fullname

Query 也支持 ORM 描述作为参数。任何时候，多个类的实体或者是基于列的实体表达都可以作为 query() 函数的参数，返回类型是元组：

    for name, fullname in session.query(User.name,User.fullname): print name, fullname

Query 返回的元组被命名为 KeyedTuple 类的实例元组。并且可以把它当成一个普通的 Python 数据类操作。元组的名字就相当于属性的属性名，类的类名一样。

     for row in session.query(User, User.name).all():

    print row.User,row.name

    <User(name='ed',fullname='Ed Jones', password='f8s7ccs')>ed

label() 不知道怎么解释，看下例子就明白了。相当于 row.name

```
for row in session.query(User.name.label('name_label')).all():

print(row.name_label)


```

aliased() 我的理解是类的别名，如果有多个实体都要查询一个类，可以用 aliased()

    from sqlalchemy.orm import aliased

    user_alias = aliased(User, name='user_alias')

    for row in session.query(user\_alias,user\_alias.name).all():

    print row.user_alias

Query 的 基本操作包括 LIMIT 和 OFFSET，使用 Python 数组切片和 ORDERBY 结合可以让操作变得很方便。

```
for u in session.query(User).order_by(User.id)\[1:3\]:

#只查询第二条和第三条数据


```

9.1 使用关键字变量过滤查询结果，filter 和 filter_by 都适用。【2】使用很简单，下面列出几个常用的操作：

    query.filter(User.name == 'ed') #equals

    query.filter(User.name != 'ed') #not equals

    query.filter(User.name.like('%ed%')) #LIKE

    uery.filter(User.name.in_(\['ed','wendy', 'jack'\])) #IN

    query.filter(User.name.in_(session.query(User.name).filter(User.name.like('%ed%'))#IN

    query.filter(~User.name.in_(\['ed','wendy', 'jack'\]))#not IN

    query.filter(User.name == None)#is None

    query.filter(User.name != None)#not None

    from sqlalchemy import and_

    query.filter(and_(User.name =='ed',User.fullname =='Ed Jones')) # and

    query.filter(User.name == 'ed',User.fullname =='Ed Jones') # and

    query.filter(User.name == 'ed').filter(User.fullname == 'Ed Jones')# and

    from sqlalchemy import or_

    query.filter(or_(User.name =='ed', User.name =='wendy')) #or

    query.filter(User.name.match('wendy')) #match

9.2. 返回列表和数量（标量？）

all() 返回一个列表：可以进行 Python 列表的操作。

    query = session.query(User).filter(User.name.like('%ed')).order_by(User.id)

```
query.all() 

```

    \[<User(name='ed',fullname='EdJones', password='f8s7ccs')>,<User(name='fred',
    fullname='FredFlinstone', password='blah')>\]

```
  

```

first() 适用于限制一个情况，返回查询到的第一个结果作为标量？：好像只能作为属性，类

    query.first()

    <User(name='ed',

    fullname='Ed Jones', password='f8s7ccs')>

one() 完全获取所有行，并且如果查询到的不只有一个对象或是有复合行，就会抛出异常。

    from sqlalchemy.orm.exc import MultipleResultsFound

    user = query.one()

    try:

    user = query.one()

    except
    　　MultipleResultsFound, e:

    　　print e

    Multiple rows were found for one()

如果一行也没有：

    from sqlalchemy.orm.exc import NoResultFound

    try:

    user = query.filter(User.id == 99).one()

    except
    NoResultFound, e:

    print e

    No row was found for one()

one()方法对于想要解决 “no items found” 和“multiple items found”是不同的系统是极好的。（这句有语病啊）例如 web 服务返回，本来是在 no results found 情况下返回”404“的，结果在多个 results found 情况下也会跑出一个应用异常。

scalar() 作为 one() 方法的依据，并且在 one() 成功基础上返回行的第一列。

    query = session.query(User.id).filter(User.name == 'ed')

```
query.scalar()

7


```

9.3. 使用字符串 SQL

字符串能使 Query 更加灵活，通过 text() 构造指定字符串的使用，这种方法可以用在很多方法中，像 filter() 和 order_by()。

    from sqlalchemy import text

```
for user in session.query(User).filter(text("id<224")).order_by(text("id")).all()

```

绑定参数可以指定字符串，用 params() 方法指定数值。

    session.query(User).filter(text("id<:value and name=:name")).\params(value=224, name='fred').order_by(User.id).one()

如果要用一个完整的 SQL 语句，可以使用 from_statement()。

    ession.query(User).from_statement(text("SELECT\* FROM users where name=:name")).\

     params(name='ed').all()

也可以用 from_statement() 获取完整的”raw”，用字符名确定希望被查询的特定列:

    session.query("id","name", "thenumber12").\from_statement(text("SELECT id, name, 12 as ""thenumber12 FROM users where name=:name")).\

　params(name='ed').all()

    \[(1,u'ed', 12)\]

    感觉这个不太符合ORM的思想啊。。。

```
  

```

9.4 计数

count() 用来统计查询结果的数量。

```
session.query(User).filter(User.name.like('%ed')).count() 

```

func.count() 方法比 count() 更高级一点【3】

    from sqlalchemy import func

```
 session.query(func.count(User.name),User.name).group_by(User.name).all() 

```

    \[(1,u'ed'), (1,u'fred'), (1,u'mary'), (1,u'wendy')\]

为了实现简单计数 SELECT count(\*) FROM table，可以这么写：

```
session.query(func.count('*')).select_from(User).scalar() 

```

如果我们明确表达计数是根据 User 表的主键的话，可以省略 select_from(User):

```
session.query(func.count(User.id)).scalar() 

```

上面两行结果均为 4。

[Go](http://lib.csdn.net/base/go "Go 知识库") to （下）

10\. 建立联系（外键）

是时候考虑怎样映射和查询一个和 Users 表关联的第二张表了。假设我们系统的用户可以存储任意数量的 email 地址。我们需要定义一个新表 Address 与 User 相关联。

    from sqlalchemyimport ForeignKeyfrom sqlalchemy.ormimport relationship, backref

    class Address(Base):

    \_\_tablename\_\_ = 'addresses'

    id= Column(Integer, primary_key=True)

    email_address = Column(String, nullable=False)

    user_id = Column(Integer, ForeignKey('users.id'))

    user = relationship("User", backref=backref('addresses',order_by=id))

    def\_\_repr\_\_(self):

     return"<Address(email_address='%s')>"%self.email_address

构造类和外键简单，就不过多赘述。主要说明以下 relationship() 函数：这个函数告诉 ORM，Address 类应该和 User 类连接起来，通过使用 addresses.user。relationship() 使用外键明确这两张表的关系。决定 Adderess.user 属性是多对一的。relationship() 的子函数 backref() 提供表达反向关系的细节：relationship() 对象的集合被 User.address 引用。多对一的反向关系总是一对多。更多的细节参考[Basic Rel](http://docs.sqlalchemy.org/en/rel_0_9/orm/relationships.html#relationship-patterns)[R](http://docs.sqlalchemy.org/en/rel_0_9/orm/relationships.html#relationship-patterns)[ational Patterns](http://docs.sqlalchemy.org/en/rel_0_9/orm/relationships.html#relationship-patterns)。

这两个互补关系：Address.user 和 User.addresses 被称为双向关系。这是 SQLAlchemy ORM 的一个非常关键的功能。更多关系 backref 的细节参见[Linking Relationships with Backref](http://docs.sqlalchemy.org/en/rel_0_9/orm/relationships.html#relationships-backref)。

假设声明的方法已经开始使用，relationship() 中和其他类关联的参数可以通过 strings 指定。在上文的 User 类中，一旦所有映射成功，为了产生实际的参数，这些字符串会被当做[Python](http://lib.csdn.net/base/python "Python 知识库")的表达式。下面是一个在 User 类中创建双向联系的例子：

    class User(Base):

    addresses = relationship("Address", order_by="Address.id", backref="user")

一些知识：

在大多数的外键约束（尽管不是所有的）关系[数据库](http://lib.csdn.net/base/mysql "MySQL 知识库")只能链接到一个主键列，或具有唯一约束的列。

外键约束如果是指向多个列的主键，并且它本身也具有多列，这种被称为 “复合外键”。

外键列可以自动更新自己来相应它所引用的行或者列。这被称为级联，是一种建立在关系数据库的功能。

外键可以参考自己的表格。这种被称为 “自引” 外键。

我们需要在数据库中创建一个 addresses 表，所以我们会创建另一个元数据，这将会跳过已经创建的表。

11\. 操作主外键关联的对象

现在我们已经在 User 类中创建了一个空的 addresser 集合，可变集合类型，例如 set 和 dict，都可以用，但是默认的集合类型是 list。

    jack = User(name='jack', fullname='Jack Bean', password='gjffdd')

    jack.addresses

    \[\]

现在可以直接在 User 对象中添加 Address 对象。只需要指定一个完整的列表：

    jack.addresses = \[Address(email_address='jack@google.com'),Address(email_address='j25@yahoo.com')\]

    当使用双向关系时，元素在一个类中被添加后便会自动在另一个类中添加。这种行为发生在Python的更改事件属性中而不是用SQL语句：

    >>\> jack.addresses\[1\]

    <Address(email_address='j25@yahoo.com')>

    >>\> jack.addresses\[1\].user

    <User(name='jack', fullname='Jack Bean', password='gjffdd')>

    把jack提交到数据库中，再次查询Jack，（No SQL is yet issued for Jack’s addresses:）这句实在是翻译不了了，看看代码就明白是什么意思：

    >>\> jack = session.query(User).\  
    ...

```
filter_by(name='jack').one() 

```

    >>\> jack

    <User(name='jack',fullname='Jack Bean', password='gjffdd')>

```
  

```

```
>>>jack.addresses 

```

    \[<Address(email_address='jack@google.com')>,
    <Address(email_address='j25@yahoo.com')>\]

    当我们访问uaddresses集合时，SQL会被突然执行，这是一个延迟加载（[lazy loading](http://docs.sqlalchemy.org/en/rel_0_9/glossary.html#term-lazy-loading)）关系的典型例子。现在addresses集合加载完成并且可以像对待普通列表一样对其进行操作。以后我们会优化这种加载方式。

    12.使用JOINS查询

    现在我们有了两张表，可以进行更多的查询操作，特别是怎样对两张表同时进行查询，[Wikipediapage on SQL JOIN](http://en.wikipedia.org/wiki/Join_%28SQL%29)提供了很详细的说明，其中一些我们将在这里说明。之前用Query.filter()时，我们已经用过JOIN了，filter是一种简单的隐式join：

    >>>for u, a in session.query(User, Address).filter(User.id==Address.user_id).filter(Address.email_address=='jack@google.com').all(): 

     print u

     print a

    <User(name='jack',fullname='JackBean', password='gjffdd')>

    <Address(email_address='jack@google.com')>

    用Query.join()方法会更加简单：

    >>>session.query(User).join(Address).\

    ...
        filter(Address.email_address=='jack@google.com').\

```
...
    all() 

```

    \[<User(name='jack',fullname='JackBean', password='gjffdd')>\]

    之所以Query.join()知道怎么join两张表是因为它们之间只有一个外键。如果两张表中没有外键或者有一个以上的外键，当下列几种形式使用的时候，Query.join()可以表现的更好：

    query.join(Address,User.id==Address.user_id)\# 明确的条件

    query.join(User.addresses)\# 指定从左到右的关系

    query.join(Address,User.addresses)    #同样，有明确的目标

    query.join('addresses') \# 同样，使用字符串

     outerjoin()和join()用法相同

    query.outerjoin(User.addresses)\# LEFT OUTER JOIN

    12.1使用别名

    当在多个表中查询时，如果同一张表需要被引用好几次，SQL通常要求对这个表起一个别名，因此，SQL可以区分对这个表进行的其他操作。Query也支持别名的操作。下面我们joinAddress实体两次，找到同时拥有两个不同email的用户：

    >>>from sqlalchemy.ormimport aliased

    >>>adalias1 = aliased(Address)

    >>>adalias2 = aliased(Address)

    >>>for username, email1, email2 in\

    ...
        session.query(User.name,adalias1.email_address,adalias2.email_address).\

    ...
        join(adalias1, User.addresses).\

    ...
        join(adalias2, User.addresses).\

    ...
        filter(adalias1.email_address=='jack@google.com').\

    ...
        filter(adalias2.email_address=='j25@yahoo.com'):

```
...
    print username, email1,
email2 

```

    jack
    jack@google.com j25@yahoo.com

    12.1使用子查询（暂时理解不了啊，多看代码研究吧：(）

    from sqlalchemy.sqlimport func

    stmt = session.query(Address.user_id,func.count('*').\

    ...         label('address_count')).\

    ...         group_by(Address.user_id).subquery()

    >>>
    for u, count in session.query(User,stmt.c.address_count).\

    ...
        outerjoin(stmt, User.id==stmt.c.user_id).order_by(User.id): 

     print u, count

    <User(name='ed',fullname='EdJones', password='f8s7ccs')>
    None

    <User(name='wendy',fullname='Wendy Williams', password='foobar')>
    None

    <User(name='mary',fullname='Mary Contrary', password='xxg527')>
    None

    <User(name='fred',fullname='Fred Flinstone', password='blah')>
    None

    <User(name='jack',fullname='Jack Bean', password='gjffdd')>
    2

    12.2从子查询中选择实体？

    上面的代码中我们只返回了包含子查询的一个列的结果。如果想要子查询映射到一个实体的话，使用aliased()设置一个要映射类的子查询别名：

    >>>
    stmt = session.query(Address).\

    ...
         filter(Address.email_address!= 'j25@yahoo.com').\

    ...
         subquery()

    >>>
    adalias = aliased(Address, stmt)
    #？为什么有两个参数？

    >>>
    for user, address in session.query(User, adalias).\

```
...
        join(adalias, User.addresses): 

```

    ...
        print user

    ...
        print address

    <User(name='jack',fullname='Jack Bean', password='gjffdd')>

    <Address(email_address='jack@google.com')>

12.3 使用 EXISTS（存在？）

如果表达式返回任何行，EXISTS 为真，这是一个布尔值。它可以用在 jions 中，也可以用来定位在一个关系表中没有相应行的情况：

    >>>from sqlalchemy.sqlimport exists

    >>> stmt = exists().where(Address.user_id==User.id)

```
>>>for name, in session.query(User.name).filter(stmt):  

```

     print name

    jack

等价于：

    >>>for name, in session.query(User.name).\

```
...
 　　filter(User.addresses.any()): 

```

    ...
        print name

    jack

any() 限制行匹配：

    >>>for name, in session.query(User.name).\

```
...
   
filter(User.addresses.any(Address.email_address.like('%google%'))): 

```

    ...
        print name

    jack

has() 和 any() 一样在应对多对一关系的情况下（注意 “～“意味着”NOT”）

    >>\> session.query(Address).\

```
...
        filter(~Address.user.has(User.name=='jack')).all() 

```

    \[\]

12.4 常见的关系运算符

== ！= None 都是用在多对一中，而 contains() 用在一对多的集合中：

    query.filter(Address.user == someuser)

    query.filter(User.addresses.contains(someaddress))

Any()（用于集合中）：

    query.filter(User.addresses.any(Address.email_address == 'bar'))#also takes keyword arguments:

    query.filter(User.addresses.any(email_address='bar'))

as()（用在标量？不在集合中）：

    query.filter(Address.user.has(name='ed'))

Query.with_parent()（所有关系都适用）：

    session.query(Address).with_parent(someuser,'addresses')

13 预先加载（跟性能有关）和 lazy loading 相对，建议直接查看文档吧

待补充。。。
