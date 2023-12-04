# alembic迁移_alembic使用_Bruce-li__的博客-CSDN博客
[alembic 迁移\_alembic 使用\_Bruce-li\_\_的博客 - CSDN 博客](https://blog.csdn.net/qq_44623314/article/details/130651092) 

## alembic 作用

alembic 是 sqlalchemy 的作者开发的。用来做 OMR 模型与数据库的迁移与映射。alembic 使用方式跟[git](https://so.csdn.net/so/search?q=git&spm=1001.2101.3001.7020)有点了类似，表现在两个方面。

```
`第一个，alembic的所有命令都是以alembic开头
第二，alembic的迁移文件也是通过版本进行控制的。首先，通过pip install alembic进行安装。` 

*   1
*   2


```

以下将解释 alembic 的用法  
方便数据库与[ORM](https://so.csdn.net/so/search?q=ORM&spm=1001.2101.3001.7020)模型的迁移与映射

## 一. 项目开始前就应用

### 1.alembic 安装

```
`pip install alembic==1.10.4` 

*   1


```

```
`pip install pymysql==1.0.3` 

*   1


```

### 2.alembic 用法

先创建好 ORM 模型  
这里默认的 sqlalchemy2.0 版本

```
`from sqlalchemy import create_engine, Column, String, Integer
from sqlalchemy.orm import DeclarativeBase

DIALCT = "mysql"
DRIVER = "pymysql"
USERNAME = "root"
PASSWORD = "123456"
HOST = "127.0.0.1"
PORT = "3306"
DATABASE = "alembic_demo"

DB_URI = "{}+{}://{}:{}@{}:{}/{}?charset=utf8".format(DIALCT, DRIVER, USERNAME, PASSWORD, HOST, PORT, DATABASE)
"""
sqlalchemy2.0版本
"""

engine = create_engine(DB_URI)

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"
    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String(50), nullable=False)` 

*   1
*   2
*   3
*   4
*   5
*   6
*   7
*   8
*   9
*   10
*   11
*   12
*   13
*   14
*   15
*   16
*   17
*   18
*   19
*   20
*   21
*   22
*   23
*   24
*   25
*   26
*   27
*   28
*   29


```

### 3. 进入虚拟环境

### 4. 初始化 alembic 仓库

```
`alembic init alembic` 

*   1


```

```
`#注：第一个alembic是alembic语法，
#类似git。第二个init代表初始化，
#第三个alembic代表仓库名，你也可以命名其它名字，
#这里为了方便理解，所以采用了alembic这一名字
我们就可以看到我们的项目下多了一个alembic文件和alembic.ini文件` 

*   1
*   2
*   3
*   4
*   5


```

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-12-4%2018-14-33/76c727ea-255e-4db6-8af1-0c5f40926fb4.png?raw=true)

### 5. 修改 alembic 配置文件

```
`sqlalchemy.url = mysql+pymysql://root:123456@localhost/alembic_demo` 

*   1


```

\#注：和[数据库连接](https://so.csdn.net/so/search?q=%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%9E%E6%8E%A5&spm=1001.2101.3001.7020)信息一样

### 6. 修改 env.py 文件的 target_metadata 参数

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-12-4%2018-14-33/b182d5fb-e34c-4fa6-b724-d370355b30be.png?raw=true)

```
`import os
import sys
import alembic_demo
 

sys.path.append(os.path.dirname(os.path.dirname(__file__)))
 
target_metadata = alembic_demo.Base.metadata` 

*   1
*   2
*   3
*   4
*   5
*   6
*   7
*   8


```

### 7. 迁移

创建数据库迁移文件

```
`alembic revision --autogenerate -m "first commit"` 

*   1


```

```
`#创建成功会在version目录下创建一个迁移文件
#前面xxx...ad这段代表迁移版本号，后面first_commit代表迁移信息` 

*   1
*   2


```

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-12-4%2018-14-33/5b2a2f44-2303-46e2-9bb5-5f2f9882d372.png?raw=true)

将迁移文件映射到数据库中

```
`alembic upgrade head` 

*   1


```

出现下方信息，说明成功映射到表中  
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-12-4%2018-14-33/7f9a0e59-2360-4480-80fc-28ab9c2b9216.png?raw=true)
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-12-4%2018-14-33/bc3b8ec2-88ba-4e7b-bfc5-2f5f519b6d98.png?raw=true)

#### 注：alembic 映射到数据库流程，ORM 模型 ——迁移文件——映射到数据库中

#### 7.1 这里是针对使用 sqlalchemy1.4 的

##### 报错: TypeError: declarative_base() takes 0 positional arguments but 1 was given

```
`pip install sqlalchemy==1.4.48` 

*   1


```

```
`from sqlalchemy import create_engine,Column,String,Integer
from sqlalchemy.ext.declarative import declarative_base
 
DIALCT = "mysql"
DRIVER = "pymysql"
USERNAME = "root"
PASSWORD = "123456"
HOST = "127.0.0.1"
PORT = "3306"
DATABASE = "alembic_demo"
 
DB_URI = "{}+{}://{}:{}@{}:{}/{}?charset=utf8".format(DIALCT,DRIVER,USERNAME,PASSWORD,HOST,PORT,DATABASE)
"""
sqlalchemy1.4版本
"""
engine = create_engine(DB_URI)
Base = declarative_base(engine)
 
class User(Base):
    __tablename__ = "user"
    id = Column(Integer , primary_key=True , autoincrement=True)
    name = Column(String(50) , nullable=False)` 

*   1
*   2
*   3
*   4
*   5
*   6
*   7
*   8
*   9
*   10
*   11
*   12
*   13
*   14
*   15
*   16
*   17
*   18
*   19
*   20
*   21
*   22
*   23
*   24


```

### 8. 重复

如果以后修改了代码，则重复 **7** 的步骤。

### 9. 常用命令和参数

-   init：创建一个 alembic 仓库。
-   revision：创建一个新的版本文件。
-   –autogenerate：自动将当前模型的修改，生成迁移脚本。
-   \-m：本次迁移做了哪些修改，用户可以指定这个参数，方便回顾。
-   upgrade：将指定版本的迁移文件映射到数据库中，会执行版本文件中的 upgrade 函数。如果有多个迁移脚本没有被映射到数据库中，那么会执行多个迁移脚本。
-   \[head]：代表最新的迁移脚本的版本号。
-   downgrade：会执行指定版本的迁移文件中的 downgrade 函数。
-   heads：展示 head 指向的脚本文件版本号。
-   history：列出所有的迁移版本及其信息。
-   current：展示当前数据库中的版本号。

另外，在你第一次执行 upgrade 的时候，就会在数据库中创建一个名叫 alembic_version 表，这个表只会有一条数据，记录当前数据库映射的是哪个版本的迁移文件。

```
``- 更新数据库 	`alembic upgrade 版本号`
- 更新到最新版 	`alembic upgrade head`
- 降级数据库 	`alembic downgrade 版本号` 
- 更新到最初版 	`alembic downgrade head`
alembic upgrade 版本号 --sql > migration.sql`` 

*   1
*   2
*   3
*   4
*   5


```

## 二. 项目开发到一半也可以用 alembic 库

-   1\. 数据库字段类型和代码字段类型不一致
-   2\. 表中有数据了改代码, 数据不会丢失  
    都不会受到影响

## 三. 常见问题

#### 1. 因某些原因误删除了 alembic 迁移文件，导致数据库版本号（alembic_version）跟迁移文件版本号不一致所致![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-12-4%2018-14-33/6d995078-31fc-46f0-a525-bba622d1d4f7.png?raw=true)

解决方法: 打开 mysql 删除 alembic_version 版本号

#### 2.sqlalchemy.exc.OperationalError: (pymysql.err.OperationalError) (1170, “BLOB/TEXT column ‘c_advice’ used in key specification without a key length”) 的解决办法

```
 `email_send_person= Column(LONGTEXT(),index=True,  comment="收件人")` 

*   1


```

(1) 把 index=True 去掉

```
 `email_send_person= Column(LONGTEXT(),comment="收件人")` 

*   1


```

（2）尝试使用 VARCHAR 并设置长度

```
 `email_send_person= Column(VARCHAR(10000),index=True,comment="收件人")` 

*   1


```

### 四. 迁移小工具

创建一个 bat 文件，就不用每次敲命令行了  
migrate.bat

```
`alembic revision && alembic upgrade head` 

*   1


```

migrate.sh

```
`alembic revision && alembic upgrade head` 

*   1


```

### 五. 动态修改 alembic.ini 配置文件

```
 `from configparser import ConfigParser
    
  config_parser = ConfigParser()
   
  config_parser.read('alembic.ini')
  config_parser.set('alembic', 'sqlalchemy.url', 'mysql+pymysql://root:123456@127.0.0.1/test?charset=utf8mb4')
  with open('alembic.ini', 'w') as f:
       config_parser.write(f)` 

*   1
*   2
*   3
*   4
*   5
*   6
*   7
*   8


```
