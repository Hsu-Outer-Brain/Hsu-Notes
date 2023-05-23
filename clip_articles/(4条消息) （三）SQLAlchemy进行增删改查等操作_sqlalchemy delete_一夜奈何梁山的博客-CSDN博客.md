# (4条消息) （三）SQLAlchemy进行增删改查等操作_sqlalchemy delete_一夜奈何梁山的博客-CSDN博客
[(4 条消息) （三）SQLAlchemy 进行增删改查等操作_sqlalchemy delete_一夜奈何梁山的博客 - CSDN 博客](https://blog.csdn.net/qq_41341757/article/details/119875450) 

-   在 ORM 中直接定义类对象就可以操作数据库，不再使用底层的 Insert 对象了。
-   注意： query 和 exec 中执行 select 语句都可以进行查询，两者的适用场景不同。
    -   对于复杂查询我们还是推荐使用 select 语句。
    -   对于简单的语句，我们推荐使用 query 语句。（写法简单）

## 一：增加操作

-   1： 数据准备： 创建一个简单的 User 模型类

    ```python
    from sqlalchemy.orm import registry
    from sqlalchemy import Column, Integer, VARCHAR

    from main import engine

    my_registry = registry()
    Base = my_registry.generate_base()


    class User(Base):
        __tablename__ = "users"
        id = Column(Integer, primary_key=True)
        name = Column(VARCHAR(255), nullable=False)
        age = Column(VARCHAR(255), nullable=False)
        sex = Column(Integer, nullable=True)


    my_registry.metadata.create_all(engine) 

    ```
-   2： 我们可以使用**类的对象来表示一行数据**，如果我们打印这个对象会发现，这个对象会补充默认的数据。

    -   本质： **模型类会映射出一个 SQL 语句**。

    ```python
    from sqlalchemy import create_engine
    import settings
    from module import User

    engine = create_engine(settings.DB_URI, echo=True)

    user = User(name='梁山', age=23, sex=1)
    print(user.id, user.name, user.age, user.sex)


    ```
-   3： 原来我们使用 with 上下文来管理资源，现在我们直接使用[session](https://so.csdn.net/so/search?q=session&spm=1001.2101.3001.7020)来管理资源。

    -   基于引擎创建 session: session = Session(engine)
    -   将对象添加进入 session： session.add(user)
    -   查看所有已经添加到会话的对象 (没有提交的)：session.new
    -   注意： **session 会将对象添加到一个 IdentitySet 的字典中，这个字典会对所有添加的对象进行哈希处理，因此同一个 session 对象，不管你加入多少次 user，最终只会提交到数据库一次**。

    ```python
    engine = create_engine(settings.DB_URI, echo=True)

    user = User(name='梁山', age=23, sex=1)
    session = Session(engine)
    session.add(user)
    print(session.new)


    ```

    ```python
    engine = create_engine(settings.DB_URI, echo=True)

    user = User(name='梁山', age=23, sex=1)
    session = Session(engine)

    session.add(user)
    session.add(user)
    print(session.new)


    ```
-   4：理解 Flush 刷新：

    -   作用： 预提交
    -   特点： **在数据库中执行 sql 语句，返回结果，但是不会持久化到数据库。** 
    -   对于插入来讲，**flush 刷新会导致对象存在了主键值**。

    ```python
    engine = create_engine(settings.DB_URI, echo=True)

    user = User(name='梁山', age=23, sex=1)
    session = Session(engine)
    session.add(user)
    session.flush()

    ```

    ```python
    2021-08-23 11:20:52,657 INFO sqlalchemy.engine.Engine BEGIN (implicit)
    2021-08-23 11:20:52,658 INFO sqlalchemy.engine.Engine INSERT INTO users (name, age, sex) VALUES (%(name)s, %(age)s, %(sex)s) RETURNING users.id
    2021-08-23 11:20:52,658 INFO sqlalchemy.engine.Engine [generated in 0.00018s] {'name': '梁山', 'age': 23, 'sex': 1}

    ```

    ```python
    user = User(name='梁山', age=23, sex=1)
    print(user.id)

    session = Session(engine)
    session.add(user)
    session.flush()
    print(user.id)


    ```
-   5: 提交与关闭会话:

    -   作用： 持久化到数据库，关闭会话。

    ```python
    engine = create_engine(settings.DB_URI, echo=True)

    user = User(name='梁山', age=23, sex=1)
    print(user.id)

    session = Session(engine)
    session.add(user)
    session.commit()
    session.close()

    ```

    ```python
    2021-08-23 11:27:56,195 INFO sqlalchemy.engine.Engine INSERT INTO users (name, age, sex) VALUES (%(name)s, %(age)s, %(sex)s) RETURNING users.id
    2021-08-23 11:27:56,196 INFO sqlalchemy.engine.Engine [generated in 0.00025s] {'name': '梁山', 'age': 23, 'sex': 1}
    2021-08-23 11:27:56,254 INFO sqlalchemy.engine.Engine COMMIT

    ```

## 二：更新操作：

-   1： 普通的更新操作：

    ```python
    engine = create_engine(settings.DB_URI, echo=True)


    session = Session(engine)

    session.execute(update(User).where(User.name == "梁山").values(name="王老五"))

    session.commit()
    session.close()

    ```
-   2：基于查询更新：

    -   先查询出这个对象，然后修改这个对象的属性，此时会话层数据和持久层数据发生不一致了。
    -   当 session.commit() 的时候，会话层数据持久化到持久层，就保持了一致。

    ```python
    from sqlalchemy import create_engine
    from sqlalchemy.orm import Session
    import settings
    from module import User

    engine = create_engine(settings.DB_URI, echo=True)


    session = Session(engine)

    user = session.query(User).filter(User.name == "王老五").first()
    user.name = "梁山"

    session.commit()
    session.close()

    ```

## 三：删除操作：

-   1： 普通的删除操作：

    ```python

    session = Session(engine)

    session.execute(delete(User).where(User.name == "梁山"))

    session.commit()
    session.close()

    ```
-   2： 基于查询进行删除：

    ```python

    session = Session(engine)

    user = session.query(User).filter(User.name == "梁山").first()
    session.delete(user)

    session.commit()
    session.close()

    ```

## 四： 简单查询操作

### 4.1： Session.query 进行查询：

#### 1： 条件查询：

-   注意：与 select 不同的是，使用 query 直接得到对象列表，遍历这个列表直接就是对象。

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import Session
import settings
from module import User

engine = create_engine(settings.DB_URI, echo=True)


session = Session(engine)

users = session.query(User).filter(User.name == "梁山").all()
print(type(users))

for user in users:
    print(user)
    
session.close()


```

-   返回部分数据：

```python

session = Session(engine)

user = session.query(User.name).first()
print(type(user))

print(user.name)

session.close()

```

#### 2： 查询单一数据返回：

-   query().first()

```python
engine = create_engine(settings.DB_URI, echo=True)


session = Session(engine)

user = session.query(User).filter(User.name == "梁山").first()
print(user)

session.close()

```

#### 4： 条件组合：

```python
engine = create_engine(settings.DB_URI, echo=True)


session = Session(engine)

user = session.query(User.name).filter(and_(User.name == "梁山", User.age == "23")).first()
print(user.name)

session.close()

```

#### 5： filter_by 进行查询：

-   注意： filter_by 子句中使用的是属性赋值的方式。

```python

session = Session(engine)

user = session.query(User).filter_by(name="梁山", age="23").first()
print(user.name)

session.close()

```

#### 6： 连表查询：

-   使用 join 进行连表查询： 注意 join 中写后面的那个表。

```python

session = Session(engine)

user_and_address = session.query(User, Address.address).filter(User.name == "梁山").join(Address).all()
print(type(user_and_address))

print(user_and_address)


session.close()

```

-   全连接：

```python
engine = create_engine(settings.DB_URI, echo=True)


session = Session(engine)

user_and_address = session.query(User, Address.address).join(Address, full=True)
for ua in user_and_address:
    print(ua)
    
    
    
    
    
    
session.close()

```

-   外连接：

```python
engine = create_engine(settings.DB_URI, echo=True)


session = Session(engine)

user_and_address = session.query(User, Address.address).join(Address, isouter=True)
for ua in user_and_address:
    print(ua)
session.close()

```

### 4.2： select() 函数进行查询：

#### 1： 条件查询：

-   注意： select 语句返回的是 result 对象， 遍历 result 对象得到的是 row 对象，这个对象是个元组，需要使用下标访问元组内容，才能得到对象。

```python

session = Session(engine)

users = session.execute(select(User).where(User.name == "梁山"))
print(type(users))

for user in users:
    print(type(user))
    
    print(type(user[0]))
    
    print(user[0].name)

session.close()

```

-   返回部分数据而不是全部数据：

```python


```

#### 2： 查询单一数据返回：

-   注意： 是对 execute 执行的结果再使用 first() 调用取其中一个。
-   注意：此时拿到的也是 Row 对象。

```python
engine = create_engine(settings.DB_URI, echo=True)


session = Session(engine)

user = session.execute(select(User).where(User.name == "梁山")).first()
print(type(user))

print(user[0])

session.close()

```

#### 3： 对列起别名：

-   注意： 使用 label 对列进行改造和标识。

```python

session = Session(engine)

user = session.execute(select(("姓名：" + User.name).label("username")).where(User.name == "梁山")).first()
print(type(user))

print(user.username)

session.close()

```

#### 4: 条件组合：

-   and_ 与 or_的使用：
    -   注意： 虽然数据库存储的 age 是 int 类型但是我们编写代码也要用字符串类型。

```python

session = Session(engine)

user = session.execute(select(User).where(and_(User.name == "梁山", User.age == "23"))).first()
print(user[0].name)

session.close()

```

#### 5： filter_by 进行查询：

-   注意： filter_by 使用的方式是属性赋值的方式。

```python
engine = create_engine(settings.DB_URI, echo=True)


session = Session(engine)

user = session.execute(select(User).filter_by(name="梁山", age="23")).first()
print(user[0].name)

session.close()

```

#### 6： 连表查询：

-   join_from()
-   join()

数据准备：

```python
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(VARCHAR(255), nullable=False)
    age = Column(VARCHAR(255), nullable=False)
    sex = Column(Integer, nullable=True)


class Address(Base):
    __tablename__ = "address"
    id = Column(Integer, primary_key=True)
    address = Column(VARCHAR(255), nullable=True)
    user_id = Column(Integer, ForeignKey("users.id"))

```

-   使用 join_from 进行连表查询： 查询姓名为梁山的人的所有信息和详细地址信息。

```python

session = Session(engine)

user_and_address = session.execute(select(User, Address.address)
                                   .filter(User.name == "梁山")
                                   .join_from(User, Address)
                                   ).all()
for ua in user_and_address:
    print(ua[0].name, ua[1])
    
    
    

```

-   使用 join 进行连表查询：
    -   区别： join() 里面只用写后面的那个表，而 join_from 需要两个表都要写。
    -   **join_from 显示的指定那个表是左表，那个是右表。而 join 是指定右表，自动推测左表**。

```python

session = Session(engine)

user_and_address = session.execute(select(User, Address.address)
                                   .filter(User.name == "梁山")
                                   .join(Address)
                                   ).all()
for ua in user_and_address:
    print(ua[0].name, ua[1])
    
    
    

```

-   内连接，外连接（左连接）， 全连接区别：
    -   内连接： 两个表的公共部分
    -   外连接（左连接）：左表保留和公共部分保留。
    -   全连接： 笛卡尔乘积
-   join 和 join_from 都是内连接。
-   外连接和全连接的使用：
    -   外连接

```python
engine = create_engine(settings.DB_URI, echo=True)


session = Session(engine)

user_and_address = session.execute(select(User, Address.address)
                                   .join(Address, isouter=True)
                                   ).all()
for ua in user_and_address:
    print(ua[0].name, ua[1])
    
    
    
    
    
    

session.close()

```

```python

engine = create_engine(settings.DB_URI, echo=True)


session = Session(engine)

user_and_address = session.execute(select(User, Address.address)
                                   .join(Address, full=True)
                                   ).all()
for ua in user_and_address:
    print(ua[0].name, ua[1])

session.close()

```

## 五： 复杂查询：

### 1： 排序：

-   **列. asc() 正序， 列. sesc() 逆序**。
-   **asc(列名)， desc(列名)**(推荐使用)
-   select 语句进行排序：

```python

session = Session(engine)

users = session.execute(select(User.name, User.age).order_by(User.age.desc()))
for row in users:
    print(row[0], row[1])
    
    
    
session.close()

```

-   query 进行排序： 当然如果多个排序字段，后面直接追加就可以了。

```python
engine = create_engine(settings.DB_URI, echo=True)


session = Session(engine)

users = session.query(User.name, User.age).order_by(User.age.desc()).all()
print(users)

users2 = session.query(User.name, User.age).order_by(User.age.asc()).all()
print(users2)

session.close()

```

### 2： 分组：

-   1： 使用 select 进行简单分组：

    ```python
    engine = create_engine(settings.DB_URI, echo=True)


    session = Session(engine)

    users = session.execute(select(User.sex, func.count(User.id).label("数量")).group_by(User.sex))
    for row in users:
        print(row[0], row[1])
        
        

    session.close()

    ```
-   2： 使用 query 进行简单分组：

    ```python
    engine = create_engine(settings.DB_URI, echo=True)


    session = Session(engine)

    users = session.query(User.sex, func.count(User.id).label("性别")).group_by(User.sex).all()
    print(users)


    session.close()

    ```
-   3：分组后，对分组的数据进行条件过滤：

    ```python
    engine = create_engine(settings.DB_URI, echo=True)


    session = Session(engine)

    users = session.query(User.sex, func.count(User.id).label("数量")).group_by(User.sex).having(
        func.count(User.id) > 1).all()

    for row in users:
        print(row[0], row[1])
        

    session.close()

    ```

    ```python
    engine = create_engine(settings.DB_URI, echo=True)


    session = Session(engine)

    users = session.execute(
        select(User.sex, func.count(User.id).label("数量")).group_by(User.sex).having(func.count(User.id) > 1))
    for row in users:
        print(row[0], row[1])
        

    ```
-   4： 分组完在组内排序 ：

    ```python

      session = Session(engine)

      users = session.query(User.name, User.age).group_by("id").order_by(desc("age"))
    for row in users:
          print(row)
          
          
          

    ```

    ```python

    session = Session(engine)

    users = session.execute(select(User.name, User.age).group_by("id").order_by(desc("age")))
    for row in users:
        print(row)
        
        
        

    ```

### 3： 别名：

-   场景： 子关联查询：
-   需求： where 子句中需要同一个表出现多次，为了区分需要给表起别名：

    ```python
    engine = create_engine(settings.DB_URI, echo=True)


    session = Session(engine)

    user_alias_1 = aliased(User)
    user_alias_2 = aliased(User)

    users = session.query(user_alias_1.name, user_alias_2.name).where(user_alias_1.name == user_alias_2.name).all()


    for row in users:
        print(row)
        
        
        

    ```

    ```python

    session = Session(engine)

    user_alias_1 = aliased(User)
    user_alias_2 = aliased(User)

    users = session.execute(select(user_alias_1.name, user_alias_2.name).where(user_alias_1.name == user_alias_2.name)).all()


    for row in users:
        print(row)
        
        
        

    session.close()

    ```

### 4： 子查询：

#### 4.1: subquery

-   1: 使用 subquery 生成一个子查询语句：

    ```python

    session = Session(engine)

    subq = select(User.name, User.age).where(User.age > "20")
    print(subq)
    """
    SELECT users.name, users.age 
    FROM users 
    WHERE users.age > :age_1
    """
    users = session.execute(subq)
    for row in users:
        print(row)
        
        
        

    ```
-   2：使用子查询：

    ```python
    engine = create_engine(settings.DB_URI, echo=True)


    session = Session(engine)

    subq = select(User.name, User.age).where(User.age > "20")

    users = session.execute(select(User.name).where(User.name == subq.c.name))


    for row in users:
        print(row)
        
        
        
    session.close()

    ```

#### 4.2: 通用表表达式（CTE）：

-   **与 subquery 的区别是，CTE 可以递归使用**。
