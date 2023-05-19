# (3条消息) SQLAlchemy_clickhouse_sqlalchemy_雪球干死黄旭东的博客-CSDN博客
[(3 条消息) SQLAlchemy_clickhouse_sqlalchemy_雪球干死黄旭东的博客 - CSDN 博客](https://blog.csdn.net/yycoolsam/article/details/90758321) 

```python

import os
import pandas as pd

import  pymysql

from sqlalchemy import create_engine,Sequence,text
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, Integer, String, ForeignKey, UniqueConstraint, Index
from sqlalchemy.orm import sessionmaker, relationship


os.chdir('E: /data')

sale = pd.read_csv('sale.csv',encoding = 'GB18030')
'''
sale.csv:
year   market sale   profit
2010   东      33912  2641
2010   南      32246  2699
2010   西      34792  2574
2010   北      31884  2673
2011   东      31651  2437
2011   南      30572  2853
2011   西      34175  2877
2011   北      30555  2749
2012   东      31619  2106
2012   南      32443  3124
2012   西      32103  2593
2012   北      31744  2962
'''

host = yourhost
user = yourid
password = yourpassword
db = yourdatbasename
port = 3306
charset = 'utf8'

import urllib.parse
password = urllib.parse.quote_plus(password)




















   
   
   

   
       











engine = create_engine('mysql+pymysql://{user}:{password}@{host}:{port}/{db}?charset={charset}'
               .format(user = user,
                       host = host,
                       password = password,
                       db = db,
                       port = port,
                       charset = charset),
               pool_size = 30,max_overflow = 0
               pool_pre_ping=True , pool_recycle=-1,
               connect_args={'connect_timeout': 30)

conn = engine.connect()



databaseVersion = conn.execute("SELECT VERSION()").fetchone()




conn.execute('CREATE TABLE sale ('
            'year CHAR(20) NOT NULL,'
            'market CHAR(20),'
            'sale INT ,'
            'profit_from_sale INT )')

conn.execute('DROP TABLE sale')

sale.to_sql('sale',conn,if_exists='replace',index = False)

conn.execute('select * from sale').fetchall()
conn.execute('select DISTINCT year,market from sale').fetchall()
conn.execute("select * from sale where year=2012 and market=%s",'东').fetchall()
conn.execute('select year,market,sale,profit from sale order by sale').fetchall()

conn.execute("insert into sale (year,market,sale) value (2010,'东',33912)")

conn.execute("delete from sale")





Base = declarative_base()



class Sale(Base):
   __tablename__ = 'sale'
   year = Column(String(20), primary_key=True, nullable=False)
   market = Column(String(10))
   sale = Column(Integer)
   profit_from_sale = Column(Integer)
   
   def __repr__(self):
       return '%s(%s,%s,%s,%s)'%\
              (self.__tablename__,self.year,self.market,self.sale,self.profit_from_sale)
   
   
   
   
'''
sale表在sql的DDL
CREATE TABLE `sale` (
 `year` varchar(20) NOT NULL,
 `market` varchar(10) DEFAULT NULL,
 `sale` int(11) DEFAULT NULL,
 `profit_from_sale` int(11) DEFAULT NULL,
 PRIMARY KEY (`year`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
'''

Sale.__table__



class Favor(Base):
   __tablename__ = 'favor'
   nid = Column(Integer, primary_key=True, autoincrement=True)
   caption = Column(String(50), default='red', unique=True)
   sale_year = Column(String(10), ForeignKey("sale.year"))
'''
favor表在sql的DDL
CREATE TABLE `favor` (
 `nid` int(11) NOT NULL AUTO_INCREMENT,
 `caption` varchar(50) DEFAULT NULL,
 `sale_year` varchar(10) DEFAULT NULL,
 PRIMARY KEY (`nid`),
 UNIQUE KEY `caption` (`caption`),
 KEY `sale_year` (`sale_year`),
 CONSTRAINT `favor_ibfk_1` FOREIGN KEY (`sale_year`) REFERENCES `sale` (`year`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
'''

Base.metadata.create_all(conn)

Base.metadata.drop_all(conn)



Session = sessionmaker(bind=conn)
session = Session()



new = Sale(year=2010, market='东', sale=33912)
session.add(new)

session.add_all([
   Sale(year=2011, market='东', sale=31651, profit_from_sale=2437),
   Sale(year=2012, market='东', sale=31619),
               ])
try:
   
   session.commit()
except:
   
   session.rollback()



   
session.query(Sale.year, Sale.sale).all()
   
for i in session.query(Sale).order_by(Sale.sale):
   print(i.year,i.sale)
   
for i in session.query(Sale,Sale.year).all():
   print(i.Sale,i.year)
   
for i in session.query(Sale.year.label('test')).all():
   print(i.test)
from sqlalchemy.orm import aliased
Sale_test = aliased(Sale, name='sale_test' )
for i in session.query(Sale_test).all()[1:3]:
   print(i)
   
   
for i in session.query(Sale).filter_by(year = 2012).all():
   print(i)


session.query(Sale).filter(text('year>2010')).order_by(text('sale')).all()
session.query(Sale.sale).filter(text('year>2010')).group_by(Sale.sale).all()
session.query(Sale).filter(text('year=:year and market=:market'))\
   .params(year = 2011,market ='东').order_by(Sale.sale).one()
session.query(Sale).from_statement(text('select * from sale where year={}'.format(2012))).all()


session.query(Sale.profit_from_sale).count() 
from sqlalchemy import func
session.query(func.count(Sale.profit_from_sale)).scalar()
session.query(func.count()).select_from(Sale).scalar()
session.query(Sale.profit_from_sale).distinct().count()


session.query(Sale).filter(Sale.year == 2012).delete()
session.commit()

session.query(Sale).filter(Sale.year == 2010).update({"profit_from_sale":666})
session.query(Sale).filter(Sale.year == 2010).update({Sale.profit_from_sale:Sale.profit_from_sale+333})
session.commit()



session.close()












sale.to_sql('sale1',conn,if_exists='append',index = 0,index_label='year',chunksize = None)









readSqlTable1 = pd.read_sql_table('sale1',conn,parse_dates=('year'))
type(readSqlTable1['year'][0])
readSqlTable2 = pd.read_sql_table('sale1',conn,parse_dates={'year':'%Y-%M-%D'},columns=['year','sale'])
type(readSqlTable2['year'][0])
readSqlTable3 = pd.read_sql_table('sale1',conn,parse_dates={'year':{'format':'%Y-%M-%D'}})
type(readSqlTable3['year'][0])

readSqlQuery = pd.read_sql_query('select * from sale1',conn)
readSql1 = pd.read_sql('select * from sale1',conn)
readSql2 = pd.read_sql('sale1',conn)


conn.close()




with sql_engine.connect() as conn:
   source = conn.execution_options(stream_results=True, max_row_buffer=100).execute(sql_str)
   for row in source:
       tmp = pd.DataFrame([row],columns=table_columns)
       tmp.fillna("",inplace=True)
       tmp = tmp.applymap(str)
       tmp = tmp.applymap(lambda x: x.replace("'", '"'))
       tmp = tmp.applymap(lambda x: None if x == '' else x)
       json_str = str(tmp.to_dict(orient="records"))


max_row_buffer = 1000
with engine.connect() as conn:
   conn = conn.execution_options(stream_results=True, max_row_buffer=max_row_buffer)
   source = conn.execute(sqlalchemy.text(sql_str))
   c_n = 0
   result = []
   for row in source:
       result.append(row)
       c_n += 1
       if c_n == max_row_buffer:
           c_n = 0
           result = pd.DataFrame(result, columns=table_columns)
           yield result
           result = []
   else:
       result = pd.DataFrame(result, columns=table_columns)
       yield result

max_row_buffer = 1000
with engine.connect() as conn:
   conn = conn.execution_options(stream_results=True)
   source = conn.execute(sqlalchemy.text(sql_str))
   
   for row in source.partitions(max_row_buffer):
       tmp = pd.DataFrame(row, columns=table_columns)
       yield tmp


from sqlalchemy.orm import sessionmaker
Session = sessionmaker()
Session.configure(bind=engine)
session = Session()
source = session.execute(sqlalchemy.text(sql_str).execution_options(stream_results=True))
for row in source.partitions(max_row_buffer):
   tmp = pd.DataFrame(row, columns=table_columns)
   yield tmp
session.close()



from sqlalchemy import text
a = text('select * from db_test.table_A where lng_lat_GCJ02_md5= :lng_lat_GCJ02_md5 ').bindparams(lng_lat_GCJ02_md5="5deb95b66bb8fd1ac0ecd8a2cba3c92f")
with engine.connect() as conn:
   b = pd.read_sql(a, conn)
a = text('select * from db_test.table_A where lng_lat_GCJ02_md5= :lng_lat_GCJ02_md5 ').bindparams(lng_lat_GCJ02_md5=None)
with engine.connect() as conn:
   b = pd.read_sql(a, conn)




from sqlalchemy.ext.compiler import compiles,deregister
from sqlalchemy.sql.expression import Insert

@compiles(Insert)
def _prefix_insert_with_ignore(insert, compiler, **kw):
   return compiler.visit_insert(insert.prefix_with('OR IGNORE'), **kw)

def _prefix_insert_with_ignore(insert, compiler, **kw):  
   return compiler.visit_insert(insert.prefix_with('IGNORE'), **kw)
compiles(Insert)(_prefix_insert_with_ignore) 
with target_engine.connect() as conn:
   source.to_sql(target_table,conn,schema=target_schema,if_exists="append",index=False)
deregister(Insert) 


def _prefix_insert_with_repalce(insert, compiler, **kw):
   s = compiler.visit_insert(insert, **kw)
   s = s.replace("INSERT INTO", "REPLACE INTO")
   return s
compiles(Insert)(_prefix_insert_with_repalce)
with target_engine.connect() as conn:
   source.to_sql(target_table,conn,schema=target_schema,if_exists="append",index=False)
deregister(Insert) 

def _prefix_insert_with_on_duplicate_key_update(insert, compiler, **kw):
   s = compiler.visit_insert(insert, **kw)
   fields = s[s.find("(") + 1:s.find(")")].replace(" ", "").split(",")
   generated_directive = ["{0}=VALUES({0})".format(field) for field in fields]
   
   return s + " ON DUPLICATE KEY UPDATE " + ",".join(generated_directive)
compiles(Insert)(_prefix_insert_with_on_duplicate_key_update)
with target_engine.connect() as conn:
   source.to_sql(target_table,conn,schema=target_schema,if_exists="append",index=False)
deregister(Insert) 


def reconnecting_engine(engine, num_retries, retry_interval):
   
   def _run_with_retries(fn, context, cursor, statement, *arg, **kw):
       retry = 0
       while 1:
           try:
               fn(cursor, statement, context=context, *arg)
           except engine.dialect.dbapi.Error as raw_dbapi_err:
               if retry == 0:
                   e = traceback.format_exc()
                   ex = raw_dbapi_err
               connection = context.root_connection
               
               if engine.dialect.is_disconnect(
                       raw_dbapi_err, connection, cursor
               ):
                   if num_retries == 0:
                       pass
                   else:
                       if retry > num_retries:
                           raise ValueError(e)

                   engine.logger.error(
                       "disconnection error, retrying operation. retry times: {}".format(retry),
                       exc_info=False,
                   )

                   
                   
                   
                   connection.invalidate()  

                   
                   if hasattr(connection, "rollback"):
                       connection.rollback()
                   else:
                       trans = connection.get_transaction()
                       if trans:
                           trans.rollback()

                   time.sleep(retry_interval)
                   

                   while 1:
                       try:
                           context.cursor = cursor_obj = connection.connection.cursor()  
                           cursor = context.cursor
                           break
                       except Exception as ex:
                           time.sleep(retry_interval)
                           print(ex)

               
               elif _lock_table_error(raw_dbapi_err):
                   if num_retries == 0:
                       pass
                   else:
                       if retry > num_retries:
                           raise ValueError(raw_dbapi_err)
                   engine.logger.error(
                       "lock error, retrying operation. retry times: {}".format(retry),
                       exc_info=False,
                   )
                   time.sleep(retry_interval)
               else:
                   raise
           else:
               return True
           retry += 1

   def _connect_with_retries(dialect, conn_rec, cargs, cparams):
       retry = 0
       while 1:
           try:
               connection = dialect.connect(*cargs, **cparams)
               return connection
           except engine.dialect.dbapi.Error as raw_dbapi_err:
               if _lost_connect_error(raw_dbapi_err):
                   if num_retries == 0:
                       pass
                   else:
                       if retry > num_retries:
                           raise ValueError(raw_dbapi_err)
                   engine.logger.error(
                       "disconnection error, retrying operation. retry times: {}".format(retry),
                       exc_info=False,
                   )
                   time.sleep(retry_interval)
               else:
                   raise
           retry += 1

   e = engine.execution_options(isolation_level="AUTOCOMMIT")

   @event.listens_for(e, "do_execute_no_params")
   def do_execute_no_params(cursor_obj, statement, context):
       return _run_with_retries(
           context.dialect.do_execute_no_params, context, cursor_obj, statement
       )

   @event.listens_for(e, "do_execute")
   def do_execute(cursor_obj, statement, parameters, context):
       return _run_with_retries(
           context.dialect.do_execute, context, cursor_obj, statement, parameters
       )

   @event.listens_for(e, 'do_connect')
   def do_connect(dialect, conn_rec, cargs, cparams):
       return _connect_with_retries(dialect, conn_rec, cargs, cparams)


   def _lock_table_error(raw_dbapi_err, db_model: str = "mysql"):
       if db_model not in ["mysql", "oracle", "msssql", "pgsql"]:
           raise ValueError(
               'Argument {} must be in {}'.format("db_model", "[mysql, oracle, mssql, pgsql]")
           )
       lock_table_error_code_dict = {
           "mysql": [1205, 1412]
       }
       error_code = raw_dbapi_err.args[0]
       error_message = raw_dbapi_err.args[1]
       if error_code in lock_table_error_code_dict[db_model]:
           return 1
       else:
           return 0

   def _lost_connect_error(raw_dbapi_err, db_model: str = "mysql"):
       if db_model not in ["mysql", "oracle", "msssql", "pgsql"]:
           raise ValueError(
               'Argument {} must be in {}'.format("db_model", "[mysql, oracle, mssql, pgsql]")
           )
       lock_table_error_code_dict = {
           "mysql": [2003, 2006]
       }
       error_code = raw_dbapi_err.args[0]
       error_message = raw_dbapi_err.args[1]
       if error_code in lock_table_error_code_dict[db_model]:
           return 1
       else:
           return 0

   return e

def do_a_thing(engine):
   
   

   with engine.connect() as conn:
       while True:
           conn.execute('''set lock_wait_timeout = 1''')
           
           print("ping: %s" % conn.execute("select * from db_test.a_test_003 limit 1").fetchone())
           


e = reconnecting_engine(
   engine,
   num_retries=5,
   retry_interval=1,
)

do_a_thing(e)

```

github 网址：[https://github.com/PyMySQL/mysqlclient](https://github.com/PyMySQL/mysqlclient)  
若直接使用 github 上的安装指令`pip install mysqlclient`会提示失败。github 上有提示需要根据操作系统安装【MariaDB C 连接器】  
失败图  
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-19%2011-18-08/d1b7ca5a-38d1-461d-b7d7-bc2bec8ea359.png?raw=true)

windows 的  
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-19%2011-18-08/181b6519-787d-4201-bed0-63838ab3e781.png?raw=true)

linux 的  
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-19%2011-18-08/d3880b0c-938f-4b38-a590-b51520e89b3f.png?raw=true)

进入下载网页后根据自己的系统选择  
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-19%2011-18-08/a6a58867-ddcd-445b-9f89-8d8bf5a2a56d.png?raw=true)

win 的安装界面  
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-19%2011-18-08/36424754-1fa5-4887-9153-db79f40838ad.png?raw=true)

然后直接安装即可（本来官方也提示了不要更改安装路径，使用默认路径）。  
然后再次使用安装指令即可。  
如果遇到`MySQLdb/_mysql.c(29): fatal error C1083: 无法打开包括文件: “mysql.h”: No such file or directory`这个错误，我查到的是说尝试在 64 位环境中为 python32 安装 mysqlclient 时发生此错误。卸载 python 并重新安装 64 位版本。然后 pip install mysqlclient 将运行。  
不过我直接使用[https://www.lfd.uci.edu/~gohlke/pythonlibs/#mysqlclient 来安装解决 (实在不想折腾重新安装 py 的问题](<https://www.lfd.uci.edu/~gohlke/pythonlibs/#mysqlclient来安装解决(实在不想折腾重新安装py的问题>))

```python
host = '123.11.11.11'
user = 'root'
password = '123456'
db = 'db_test'
port = 3308
charset = 'utf8mb4'


engine = create_engine('mysql+mysqldb://{user}:{password}@{host}:{port}/{db}?charset={charset}'
                .format(user = user,
                        host = host,
                        password = urllib.parse.quote_plus(password),
                        db = db,
                        port = port,
                        charset = charset),
                pool_size = 30,max_overflow = 0,
                pool_pre_ping=True , pool_recycle= 3600)
conn = engine .connect()
print(str(engine .engine))
print(conn.execute("SELECT VERSION()").fetchone())


```

在 ubuntu 上失败的问题。  
一般都是 apt-get 太久没有更新，使用`sudo apt-get update`即可。  
20230213 补充  
参考文章：[https://blog.webmatrices.com/error-command-errored-out-with-exit-status-1-mysqlclient/](https://blog.webmatrices.com/error-command-errored-out-with-exit-status-1-mysqlclient/)  
如果出现下面错误

```
`qhdata@qhdata-dev:~/tmp$ sudo python3 -m pip install mysqlclient
Collecting mysqlclient
  Using cached mysqlclient-2.1.1.tar.gz (88 kB)
    ERROR: Command errored out with exit status 1:
     command: /usr/bin/python3 -c 'import sys, setuptools, tokenize; sys.argv[0] = '"'"'/tmp/pip-install-zfhc_6m1/mysqlclient/setup.py'"'"'; __file__='"'"'/tmp/pip-install-zfhc_6m1/mysqlclient/setup.py'"'"';f=getattr(tokenize, '"'"'open'"'"', open)(__file__);code=f.read().replace('"'"'\r\n'"'"', '"'"'\n'"'"');f.close();exec(compile(code, __file__, '"'"'exec'"'"'))' egg_info --egg-base /tmp/pip-install-zfhc_6m1/mysqlclient/pip-egg-info
         cwd: /tmp/pip-install-zfhc_6m1/mysqlclient/
    Complete output (15 lines):
    /bin/sh: 1: mysql_config: not found
    /bin/sh: 1: mariadb_config: not found
    /bin/sh: 1: mysql_config: not found
    Traceback (most recent call last):
      File "<string>", line 1, in <module>
      File "/tmp/pip-install-zfhc_6m1/mysqlclient/setup.py", line 15, in <module>
        metadata, options = get_config()
      File "/tmp/pip-install-zfhc_6m1/mysqlclient/setup_posix.py", line 70, in get_config
        libs = mysql_config("libs")
      File "/tmp/pip-install-zfhc_6m1/mysqlclient/setup_posix.py", line 31, in mysql_config
        raise OSError("{} not found".format(_mysql_config_path))
    OSError: mysql_config not found
    mysql_config --version
    mariadb_config --version
    mysql_config --libs
    ----------------------------------------
ERROR: Command errored out with exit status 1: python setup.py egg_info Check the logs for full command output.` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-19%2011-18-08/03ec4846-0481-4512-83b9-b3f25412e3aa.png?raw=true)

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

安装下面的程序

```
sudo apt-get install python3-dev
sudo apt-get install libmysqlclient-dev

```

然后重新安装即可。

```
`import pandas as pd,os
from sqlalchemy import create_engine

os.environ['NLS_LANG'] = 'AMERICAN_AMERICA.ZHS16GBK'
host_2 = '10.210.128.130'
user_2 = 'resmg'
password_2 = 'zgbgt.db181'
db_2 = 'client'
port_2 = 1521
engine_2 = create_engine('oracle+cx_oracle://{user}:{password}@{host}:{port}/{db}'
                .format(user = user_2,
                        host = host_2,
                        password = password_2,
                        db = db_2,
                        port = port_2),
                pool_size = 30,max_overflow = 0)

conn_2 = engine_2.connect()

如果遇到：sqlalchemy.exc.DatabaseError: (cx_Oracle.DatabaseError) DPI-1047: 64-bit Oracle Client library cannot be loaded: "The specified module could not be found". See https://oracle.github.io/odpi/doc/installation.html

这个错误，链接Oracle的客户端包添加到环境变量，然后重启电脑即可。或者在代码中添加

LOCATION = r"C:\Oracle\instantclient_18_5"   




os.environ["PATH"] = LOCATION + ";" + os.environ["PATH"]

如果遇到MSVCR120.dll文件异常，可以参考以下原文解决
https://blog.csdn.net/zengmingen/article/details/83780711
到微软官网下载 VC redist packages for x64。
微软官网地址:https://www.microsoft.com/en-us/download/details.aspx?id=40784
vcredist_x64.exe
如果是32位，则选vcredist_x32.exe

Oracle 的编码AMERICAN_AMERICA.ZHS16GBK可以通过
select userenv('language') from dual;
在Oracle中查询。` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-19%2011-18-08/8fdf0201-1179-41dc-baa7-3adf04495ac7.png?raw=true)

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
*   30
*   31
*   32
*   33
*   34
*   35
*   36
*   37
*   38
*   39
*   40
*   41
*   42


```

```
`from sqlalchemy import create_engine
host_1 = '10.210.202.153'
user_1 = 'qhtest'
password_1 = 'Qhdata12#$'
db_1 = 'sz_mppdb'
port_1 = 25308
charset_1 = 'utf8'
engine_1 = create_engine('postgresql+psycopg2://{user}:{password}@{host}:{port}/{db}'
                .format(user = user_1,
                        host = host_1,
                        password = password_1,
                        db = db_1,
                        port = port_1),
                pool_size = 30,max_overflow = 0,client_encoding='utf8')

conn_1 = engine_1.connect()` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-19%2011-18-08/b417cbda-229a-4d75-a66d-715281dc64dc.png?raw=true)

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


```

```
 `import os,re

import  pymysql,pymssql

from sqlalchemy import create_engine,Sequence,text
host = '123.11.11.11'
user = 'root'
password = '123456'
db1 = 'DC'  
port = 1433
charset = 'utf8'
engine = create_engine('mssql+pymssql://{user}:{password}@{host}:{port}/{db}?charset={charset}'
                        .format(user=user,
                                host=host,
                                password=password,
                                db=db1,
                                port=port,
                                charset=charset),
                        pool_size=100, max_overflow=0)
conn = engine.connect()` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-19%2011-18-08/230eae78-6f78-4393-837b-4febb3784603.png?raw=true)

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


```

clickhouse-sqlalchemy 的 api 网址:  
[https://github.com/xzkostyan/clickhouse-sqlalchemy](https://github.com/xzkostyan/clickhouse-sqlalchemy)  
clickhouse-driver 的 api 网址:  
[https://github.com/mymarilyn/clickhouse-driver](https://github.com/mymarilyn/clickhouse-driver)

```python
from sqlalchemy import create_engine
import urllib.parse

host = '1.1.1.1'
user = 'default'
password = 'default'
db = 'test'
port = 8123 
engine = create_engine('clickhouse://{user}:{password}@{host}:{port}/{db}'
                .format(user = user,
                        host = host,
                        password = urllib.parse.quote_plus(password),
                        db = db,
                        port = port),
                pool_size = 30,max_overflow = 0,
                pool_pre_ping=True , pool_recycle= 3600)
port = 9000 
engine1 = create_engine('clickhouse+native://{user}:{password}@{host}:{port}/{db}'
                .format(user = user,
                        host = host,
                        password = urllib.parse.quote_plus(password),
                        db = db,
                        port = port),
                pool_size = 30,max_overflow = 0,
                pool_pre_ping=True , pool_recycle= 3600)







class ClickhouseDf(object):
    def __init__(self, **kwargs):
        self.engines_dict = {
            "MergeTree": engines.MergeTree,
            "AggregatingMergeTree": engines.AggregatingMergeTree,
            "GraphiteMergeTree": engines.GraphiteMergeTree,
            "CollapsingMergeTree": engines.CollapsingMergeTree,
            "VersionedCollapsingMergeTree": engines.VersionedCollapsingMergeTree,
            "SummingMergeTree": engines.SummingMergeTree,
            "ReplacingMergeTree": engines.ReplacingMergeTree,
            "Distributed": engines.Distributed,
            "ReplicatedMergeTree": engines.ReplicatedMergeTree,
            "ReplicatedAggregatingMergeTree": engines.ReplicatedAggregatingMergeTree,
            "ReplicatedCollapsingMergeTree": engines.ReplicatedCollapsingMergeTree,
            "ReplicatedVersionedCollapsingMergeTree": engines.ReplicatedVersionedCollapsingMergeTree,
            "ReplicatedReplacingMergeTree": engines.ReplicatedReplacingMergeTree,
            "ReplicatedSummingMergeTree": engines.ReplicatedSummingMergeTree,
            "View": engines.View,
            "MaterializedView": engines.MaterializedView,
            "Buffer": engines.Buffer,
            "TinyLog": engines.TinyLog,
            "Log": engines.Log,
            "Memory": engines.Memory,
            "Null": engines.Null,
            "File": engines.File
        }
        self.table_engine = kwargs.get("table_engine", "MergeTree")  
        if self.table_engine not in self.engines_dict.keys():
            raise ValueError("No engine for this table")

    def _createORMTable(self, df, name, con, schema, **kwargs):
        col_dtype_dict = {
                "object": sqlalchemy.Text,
                "int64": sqlalchemy.Integer,
                "int32": sqlalchemy.Integer,
                "int16": sqlalchemy.Integer,
                "int8": sqlalchemy.Integer,
                "int": sqlalchemy.Integer,
                "float64": sqlalchemy.Float,
                "float32": sqlalchemy.Float,
                "float16": sqlalchemy.Float,
                "float8": sqlalchemy.Float,
                "float": sqlalchemy.Float,
            }
        primary_key = kwargs.get("primary_key", [])
        df_col = df.columns.tolist()
        metadata = MetaData(bind=con, schema=schema)

        _table_check_col = []
        for col in df_col:
            col_dtype = str(df.dtypes[col])
            if col_dtype not in col_dtype_dict.keys():
                if col in primary_key:
                    _table_check_col.append(Column(col, col_dtype_dict["object"], primary_key=True))
                else:
                    _table_check_col.append(Column(col, col_dtype_dict["object"]))
            else:
                if col in primary_key:
                    _table_check_col.append(Column(col, col_dtype_dict[col_dtype], primary_key=True))
                else:
                    _table_check_col.append(Column(col, col_dtype_dict[col_dtype]))
        _table_check = Table(name, metadata,
                        *_table_check_col,
                        self.engines_dict[self.table_engine](primary_key=primary_key)
                        )
        return _table_check


    def _checkTable(self, name, con, schema):
        sql_str = f"EXISTS {schema}.{name}"
        if con.execute(sql_str).fetchall() == [(0,)]:
            return 0
        else:
            return 1


    def to_sql(self, df, name: str, con, schema=None, if_exists="fail",**kwargs):
        '''
        将DataFrame格式数据插入Clickhouse中
        {'fail', 'replace', 'append'}, default 'fail'
        '''
        if self.table_engine in ["MergeTree"]:  
            self.primary_key = kwargs.get("primary_key", [df.columns.tolist()[0]])
        else:
            self.primary_key = kwargs.get("primary_key", [])

        orm_table = self._createORMTable(df, name, con, schema, primary_key=self.primary_key)
        tanle_exeit = self._checkTable(name, con, schema)
        
        if if_exists == "fail":
            if tanle_exeit:
                raise ValueError(f"table already exists :{name} ")
            else:
                orm_table.create()
        if if_exists == "replace":
            if tanle_exeit:
                orm_table.drop()
                orm_table.create()
            else:
                orm_table.create()
        if if_exists == "append":
            if not tanle_exeit:
                orm_table.create()

		
        df_dict = df.to_dict(orient="records") 
        con.execute(orm_table.insert(), df_dict)
        



cdf = ClickhouseDf()
with engine.connect() as conn:
   cdf.to_sql(df, table_name, conn, schema_name, if_exists="append")

```

参考文章：[https://blog.csdn.net/sxqinjh/article/details/109272042](https://blog.csdn.net/sxqinjh/article/details/109272042)  
**注意：由于人大金仓的 py 连接器跟本身的金仓数据库与 python 版本是强关联的关系，所以如果本文中的更多是记录一种连接方式，具体的连接器建议找人大金仓的工程师要。**  
**ps：不建议个人学习使用。**  
python 版本：3.7.4 64 位  
SQLAlchemy 版本：1.4.18  
人大金仓版本：KingbaseES V008R006C006B0013PS003 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.1.2 20080704 (Red Hat 4.1.2-46), 64-bit  
驱动器版本：3.7.2 64 位  
官方网址：[https://www.kingbase.com.cn/qd/index.htm](https://www.kingbase.com.cn/qd/index.htm)  
驱动下载地址：[https://kingbase.oss-cn-beijing.aliyuncs.com/KES/07-jiekouqudong/Python.rar](https://kingbase.oss-cn-beijing.aliyuncs.com/KES/07-jiekouqudong/Python.rar)  
下载完后解压得到如下文件 (红框为我这次主要使用到的文件)  
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-19%2011-18-08/90f0786a-5dfb-44a8-9eec-6504e6fe567a.png?raw=true)

解压 sqlalchemy.tar，将里面的 kingbase 文件夹的内容复制到`<PYTHON_HOME>\Lib\site-packages\sqlalchemy\dialects`中。  
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-19%2011-18-08/c6ae7867-2891-4649-93f4-abc612a5b235.png?raw=true)

将 ksycopg2-windows-amd64-MSVC2013-python3.7-64bit.zip 中解压出的 ksycopg2 文件夹放到所在项目的根目录下。  
ps：vcredist_msvc2017_x64.exe 与 vcredist_msvc2013_x64.exe 看情况安装即可。

```
 `import os,re
import  ksycopg2  

from sqlalchemy import create_engine
host = '123.11.11.11'
user = 'root'
password = '123456'
db1 = 'DC'  
port = 54321
charset = 'utf8'
engine = create_engine('kingbase+ksycopg2://{user}:{password}@{host}:{port}/{db}'
                .format(user = user,
                        host = host,
                        password = password,
                        db = db,
                        port = port))
conn = engine.connect()` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-19%2011-18-08/c3ff11cb-587b-4459-b74f-04bba6ae655e.png?raw=true)

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


```
