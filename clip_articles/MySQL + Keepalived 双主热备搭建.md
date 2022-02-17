# MySQL + Keepalived 双主热备搭建
## 什么是双主复制

在传统的主从复制架构中，从库仅仅是作为主库数据的备份，当主库发生故障时，数据库将停止对外提供服务，并且主库故障后手动进行主从切换的过程也较为繁琐。为了解决这个问题，可以采用 MySQL 双主模式，其中一台主库提供服务，另一台作为热备。结合 keepalived 使用虚拟 IP 对外提供服务，一旦主库发生故障，备库可以在很短的时间内接管服务。

![](https://mmbiz.qpic.cn/mmbiz_png/vvsibFWkwqHqibb2w950E9c5QTdFsjxtEd2vPGskyPnJ6f9rH84jVYxChhohDXWc8MIFeSrdo4zWTkEBs6sGANNA/640?wx_fmt=png)

## 机器规划

| 主机名          | IP 地址        | 端口号  | 角色           |
| ------------ | ------------ | ---- | ------------ |
| mysql-master | 192.168.1.36 | 3308 | master（主库 A） |
| mysql-slave  | 192.168.1.37 | 3308 | slave（主库 B）  |

\|  
 | 192.168.1.38 | 3308 | 虚拟 IP |

## 搭建 MySQL 双主同步

### 准备工作

#### 创建相关目录

    #创建用户userdel -r mysqlgroupadd mysql useradd -r -g mysql -s /bin/false mysql #创建目录# /mysql/app/                                   MySQL 数据库软件根目录# /mysql/data/3308/data/                        MySQL 数据文件目录# /mysql/log/3308/binlog                        MySQL 二进制日志目录# /mysql/log/3308/relaylog                      MySQL 中继日志目录# /mysql/backup/3308/xtrabackup/target_dir      MySQL xtrabackup 物理备份目录# /mysql/backup/3308/mysqldump                  MySQL mysqldump 逻辑备份目录# /mysql/script                                 MySQL 常用脚本存放目录mkdir -p /mysql/app/            mkdir -p /mysql/data/3308/data/                                 mkdir -p /mysql/log/3308/binlog                                 mkdir -p /mysql/log/3308/relaylog                               mkdir -p /mysql/backup/3308/xtrabackup/target_dir               mkdir -p /mysql/backup/3308/mysqldump    mkdir -p /mysql/script                                                                      #给目录授权chown -R mysql:mysql /mysql

#### 下载并解压 MySQL 安装包

MySQL 压缩包下载地址：[https://dev.mysql.com/downloads/mysql/5.7.html](https://dev.mysql.com/downloads/mysql/5.7.html)

    #解压压缩包tar zxvf mysql-5.7.29-linux-glibc2.12-x86_64.tar.gz -C /mysql/appmv /mysql/app/mysql-5.7.29-linux-glibc2.12-x86_64 /mysql/app/mysqlchown -R mysql:mysql /mysql 

#### 配置环境变量

    ##将MySQL目录添加环境变量##cat >> ~/.bash_profile <<-EOFexport PATH=$PATH:/mysql/app/mysql/binEOFsource ~/.bash_profile

### 初始化主库 A

主库 A 重要配置如下：

-   开启 binlog：`log_bin=binlog 目录`
-   设置 server_id：`server_id = 1`，主库 A 的 server_id 和主库 B 要不一样。
-   针对 GTIP 的方式同步有两个参数必须设置：


-   `gtid_mode=on`
-   `enforce_gtid_consistency=on`


-   防止主键冲突：


-   设置自增主键步长，通常有几个主库 就写几，避免主键冲突：`auto_increment_increment=2`
-   设置自增主键起始值，第一个主库为 1，第二个主库为 2，以此类推：`auto_increment_offset=1`

关于防止主键冲突的两个参数的详解可以参考这篇博客 ([https://www.cnblogs.com/kerrycode/p/11150782.html](https://www.cnblogs.com/kerrycode/p/11150782.html))。

    #主机名和端口号作为目录名的一部分HostName=`hostname`MySql_Port=3308#IP地址Ip=192.168.1.36#master server_id 要和 slave 不一样Server_Id=1cat > /mysql/data/$MySql_Port/my.cnf <<-EOF#------------------------------------ #客户端设置#------------------------------------[client]port=$MySql_Portsocket =/mysql/data/$MySql_Port/mysql.sockdefault-character-set=utf8 #------------------------------------ #mysql连接工具设置#------------------------------------[mysql]prompt="\\u@\\h \\d \\r:\\m:\\s>" #登录时显示登录的用户名、服务器地址、默认数据库名、当前时间auto-rehash #读取表信息和列信息，可以在连上终端后开启tab补齐功能。default-character-set=utf8 #默认字符集#------------------------------------ #基本设置#------------------------------------[mysqld]bind_address=0.0.0.0  #监听本地所有地址port=$MySql_Port  #端口号user=mysql  #用户basedir=/mysql/app/mysql  #安装路径datadir=/mysql/data/$MySql_Port/data  #MySQL数据目录socket=/mysql/data/$MySql_Port/mysql.sock #用于本地连接的socket文件目录pid-file=/mysql/data/$MySql_Port/mysql.pid #进程ID文件的目录。character-set-server=utf8 #默认字符集#------------------------------------ #log setting 日志设置#------------------------------------long_query_time=10 #慢查询时间，超过 10 秒则认为是慢查询slow_query_log=ON #启用慢查询日志slow_query_log_file=/mysql/log/$MySql_Port/${HostName}-query.log #慢查询日志目录log_queries_not_using_indexes=1 #记录未使用索引的语句log_slow_admin_statements=1 #慢查询也记录那些慢的optimize table，analyze table和alter table语句log-error=/mysql/log/$MySql_Port/${HostName}-error.log #错误日志目录#------------------------------------ #master modify parameter 主库A复制更改参数#------------------------------------server_id=$Server_Id #master和slave server_id 需要不同#------------------------------------ #slave parameter 主库B参数#------------------------------------relay_log=/mysql/log/$MySql_Port/relaylog/${HostName}-relaylog #中继日志目录relay-log-index=/mysql/log/$MySql_Port/relaylog/${HostName}-relay.index  #中继日志索引目录log_slave_updates=1 #主库B从主库A复制的数据会写入主库B binlog 日志文件里，默认是不写入read_only=0  #主库B读写权限relay_log_purge=1 #自动清空不再需要中继日志#二进制日志参数配置log_bin=/mysql/log/$MySql_Port/binlog/${HostName}-binlog  #binlog目录log_bin_index=/mysql/log/$MySql_Port/binlog/${HostName}-binlog.index  #指定索引文件的位置binlog_format=row #行模式复制，默认是 rowbinlog_rows_query_log_events=on #在 row 模式下，开启该参数,可以将把 sql 语句打印到 binlog 日志里面，方便查看binlog_cache_size=1M #事务能够使用的最大 binlog 缓存空间。max_binlog_size=2048M #binlog 文件最大空间，达到该大小时切分文件expire_logs_days=7 #设置自动删除 binlog 文件的天数。sync_binlog=1 #表示每次事务的 binlog 都会fsync持久化到磁盘，MySQL 5.7.7 之后默认为1，之前的版本默认为0innodb_flush_log_at_trx_commit=1 #表示每次事务的 redo log 都直接持久化到磁盘，默认值为1#------------------------------------ #GTID Settings GTID 同步复制设置#------------------------------------gtid_mode=on  #开启GTID同步enforce_gtid_consistency=on #强制事务一致，确保 GTID 的安全，在事务中就不能创建和删除临时表binlog_gtid_simple_recovery=1 #这个变量用于在 MySQL 重启或启动的时候寻找 GTIDs 过程中，控制 binlog 如何遍历的算法#------------------------------------ #避免主键冲突设置#------------------------------------auto_increment_increment=2  #自增主键步长，通常有几个主库A就写几，避免主键冲突auto_increment_offset=1  #设置自增主键起始值，第一个主库A为1，第二个主库A为2，以此类推EOF

初始化主库 A：

    mysqld \--defaults-file=/mysql/data/3308/my.cnf \--initialize --user=mysql \--basedir=/mysql/app/mysql \--datadir=/mysql/data/3308/data

配置 MySQL 启动脚本：

    cp /mysql/app/mysql/support-files/mysql.server /etc/init.d/mysql_3308ln -sf /etc/init.d/mysql_3308 /usr/lib/systemd/system/mysql_3308#修改启动脚本##vim /etc/init.d/mysql_3308basedir=/mysql/app/mysqldatadir=/mysql/data/3308/datamysqld_pid_file_path=/mysql/data/3308/mysql.pid#在$bindir/mysqld_safe 后面添加，注意 --defaults-file 要放在第一个--defaults-file="/mysql/data/3308/my.cnf"systemctl daemon-reload

![](https://mmbiz.qpic.cn/mmbiz_png/vvsibFWkwqHqibb2w950E9c5QTdFsjxtEdZDVwBGF1OpWgE3c9QXrxhIaUO5XOcabozb46FAnTOz1Zz2QaeK8bOw/640?wx_fmt=png)

启动 MySQL，修改密码，运行远程登录：

    #启动、MySQL服务systemctl start mysql_3308#获取MySQL临时密码Passwd=`cat /mysql/log/3308/*-error.log |grep "root@localhost:"|awk -F ' ' '{print $11}'`echo $Passwd#通过本地 socket 登录、修改密码mysql -uroot -p$Passwd -S /mysql/data/3308/mysql.sockalter user 'root'@'localhost' identified by  "123456";#允许远程登录grant all privileges on *.* to root@'%' identified by '123456';#刷新权限flush privileges;

### 初始化主库 B

主库 B 配置文件，主要是 ip 地址，server_id 以及 auto_increment_offset 的配置和主库 A 不一样，其余配置和主库 A 一样。

    #主机名和端口号作为目录名的一部分HostName=`hostname`MySql_Port=3308#IP地址Ip=192.168.1.37#master server_id 要和 slave 不一样Server_Id=2cat > /mysql/data/$MySql_Port/my.cnf <<-EOF#------------------------------------ #客户端设置#------------------------------------[client]port=$MySql_Portsocket =/mysql/data/$MySql_Port/mysql.sockdefault-character-set=utf8 #------------------------------------ #mysql连接工具设置#------------------------------------[mysql]prompt="\\u@\\h : \\d\\r:\\m:\\s>" #登录时显示登录的用户名、服务器地址、默认数据库名、当前时间auto-rehash #读取表信息和列信息，可以在连上终端后开启tab补齐功能。default-character-set=utf8 #默认字符集#------------------------------------ #基本设置#------------------------------------[mysqld]bind_address=0.0.0.0  #监听本地所有地址port=$MySql_Port  #端口号user=mysql  #用户basedir=/mysql/app/mysql  #安装路径datadir=/mysql/data/$MySql_Port/data  #MySQL数据目录socket=/mysql/data/$MySql_Port/mysql.sock #用于本地连接的socket文件目录pid-file=/mysql/data/$MySql_Port/mysql.pid #进程ID文件的目录。character-set-server=utf8 #默认字符集#------------------------------------ #log setting 日志设置#------------------------------------long_query_time=10 #慢查询时间，超过 10 秒则认为是慢查询slow_query_log=ON #启用慢查询日志slow_query_log_file=/mysql/log/$MySql_Port/${HostName}-query.log #慢查询日志目录log_queries_not_using_indexes=1 #记录未使用索引的语句log_slow_admin_statements=1 #慢查询也记录那些慢的optimize table，analyze table和alter table语句log-error=/mysql/log/$MySql_Port/${HostName}-error.log #错误日志目录#------------------------------------ #master modify parameter 主库A复制更改参数#------------------------------------server_id=$Server_Id #master和slave server_id 需要不同#二进制日志参数配置log_bin=/mysql/log/$MySql_Port/binlog/${HostName}-binlog  #binlog目录log_bin_index=/mysql/log/$MySql_Port/binlog/${HostName}-binlog.index  #指定索引文件的位置binlog_format=row #行模式复制，默认是 rowbinlog_rows_query_log_events=on #在 row 模式下，开启该参数,可以将把 sql 语句打印到 binlog 日志里面，方便查看binlog_cache_size=1M #事务能够使用的最大 binlog 缓存空间。max_binlog_size=2048M #binlog 文件最大空间，达到该大小时切分文件expire_logs_days=7 #设置自动删除 binlog 文件的天数。sync_binlog=1 #表示每次事务的 binlog 都会fsync持久化到磁盘，MySQL 5.7.7 之后默认为1，之前的版本默认为0innodb_flush_log_at_trx_commit=1 #表示每次事务的 redo log 都直接持久化到磁盘，默认值为1#------------------------------------ #slave parameter 主库B参数#------------------------------------relay_log=/mysql/log/$MySql_Port/relaylog/${HostName}-relaylog #中继日志目录relay-log-index=/mysql/log/$MySql_Port/relaylog/${HostName}-relay.index  #中继日志索引目录log_slave_updates=1 #主库B从主库A复制的数据会写入主库B binlog 日志文件里，默认是不写入read_only=0  #主库B读写权限relay_log_purge=1 #自动清空不再需要中继日志# 并行复制参数#主库A上面怎么并行，主库B上面就怎么回放，基于逻辑时钟的概念#binlog 会记录组提交的信息，从回放的时候就可以知道哪些事务是一组里面的，#一组里面的就丢到不同线程去回放，不是一组里的就等待，以此来提升并行度slave-parallel-type=LOGICAL_CLOCK#多线程复制slave-parallel-workers=4#slave 上commit 的顺序保持一致，否则可能会有间隙锁产生slave-preserve-commit_order=1master_info_repository=TABLE #默认每接收到10000个事件，写一次master-info，默认是写在文件中的#修改 relay_log_info_repository 的好处#1.relay.info 明文存储不安全，把 relay.info 中的信息记录在 table 中相对安全。#2.可以避免 relay.info 更新不及时，slave 重启后导致的主从复制出错。relay_log_info_repository=TABLE #将回放信息记录在 slave_relay_log_info 表中，默认是记录在 relay-info.log 文件中relay_log_recovery=1  #当slave重启时，将所有 relay log 删除，通过 sql 线程重放的位置点去重新拉日志#------------------------------------ #Replication Filter 主库B复制过滤参数#------------------------------------#（过滤某个数据库、数据库.表）#replicate_do_db=yzjtestdb#replicate_wild_do_table=yzjtestdb.%#replicate_do_table=yzjtestdb.yzjtest_yg#replicate_wild_do_table=yzjtestdb.yzjtest_yg#------------------------------------ #GTID Settings GTID 同步复制设置#------------------------------------gtid_mode=on  #开启GTID同步enforce_gtid_consistency=on #强制事务一致，确保 GTID 的安全，在事务中就不能创建和删除临时表binlog_gtid_simple_recovery=1 #这个变量用于在 MySQL 重启或启动的时候寻找 GTIDs 过程中，控制 binlog 如何遍历的算法#------------------------------------ #避免主键冲突设置#------------------------------------auto_increment_increment=2  #自增主键步长，通常有几个主库A就写几，避免主键冲突auto_increment_offset=2  #设置自增主键起始值，第一个主库A为1，第二个主库A为2，以此类推EOF

初始化主库 B：

    mysqld \--defaults-file=/mysql/data/3308/my.cnf \--initialize --user=mysql \--basedir=/mysql/app/mysql \--datadir=/mysql/data/3308/data

配置 MySQL 启动脚本：

    cp /mysql/app/mysql/support-files/mysql.server /etc/init.d/mysql_3308ln -sf /etc/init.d/mysql_3308 /usr/lib/systemd/system/mysql_3308#修改启动脚本##vi /etc/init.d/mysql_3308basedir=/mysql/app/mysqldatadir=/mysql/data/3308/datamysqld_pid_file_path=/mysql/data/3308/mysql.pid#在$bindir/mysqld_safe 后面添加，注意 --defaults-file 要放在第一个--defaults-file="/mysql/data/3308/my.cnf"systemctl daemon-reload

启动 MySQL，修改密码，运行远程登录：

    #启动、MySQL服务systemctl start mysql_3308#获取MySQL临时密码Passwd=`cat /mysql/log/3308/*-error.log |grep "root@localhost:"|awk -F ' ' '{print $11}'`echo $Passwd#通过本地 socket 登录、修改密码mysql -uroot -p$Passwd -S /mysql/data/3308/mysql.sockalter user 'root'@'localhost' identified by  "123456";#允许远程登录grant all privileges on *.* to root@'%' identified by '123456';#刷新权限flush privileges;

### 创建复制用户

分别在主库 A 和主库 B 上创建一个用于数据复制的用户。

    grant replication slave on *.* to 'repuser'@'%' identified by 'repuser123';

### 建立主从关系

主库 A 和主库 B 都先清除下 binlog。

    reset master;

主库 A 配置主从，指向主库 B。

    stop slave;change master to    master_host='192.168.1.36',    master_port=3308,    master_user='repuser',    master_password='repuser123',    master_auto_position=1;start slave;

主库 B 配置主从，指向主库 A。

    stop slave;change master to    master_host='192.168.1.37',    master_port=3308,    master_user='repuser',    master_password='repuser123',    master_auto_position=1;start slave;

使用 `show slave status\G` 命令查看主从同步状态，IO 线程和 SQL 线程都为 YES 表示同步正常，主库 A 和主库 B 互为主从。

![](https://mmbiz.qpic.cn/mmbiz_png/vvsibFWkwqHqibb2w950E9c5QTdFsjxtEdM4BZLibvxD3BORmFiaTlhQXHviamjmnSGSkCbTfyTPJTd2yNallzkykmw/640?wx_fmt=png)

## 部署 Keepalived

### 下载并解压安装包

    wget https://www.keepalived.org/software/keepalived-2.2.4.tar.gztar -xzvf keepalived-2.2.4.tar.gz

### 安装相关依赖

    yum install kernel-devel openssl-devel popt-devel -y

### 安装 keepalived，设置开机自动启动

    mkdir /software/keepalivedcd keepalived-2.2.4./configure --prefix=/software/keepalivedmake && make installsystemctl enable keepalivedmkdir /etc/keepalived

### 配置 Keepalived

#### 主库 A 配置 Keepalived

主库 A keepalived 配置文件，编辑 /etc/keepalived/keepalived.conf 文件：

    global_defs {  router_id keep_mysql_repl_g1     # 负载均衡标识，在局域网内应该是唯一的 }# vrrp_script 级别和 vrrp_instance 一样vrrp_script chk_mysql {        # 配置虚拟脚本 chk_mysql script "/etc/keepalived/check_mysql.sh"   # 执行脚本,检查 mysql 服务是否存活 interval 3          # 脚本执行间隔:秒}# vrrp_instance vrrp_instance v_mysql_1 {  state BACKUP          # 指定该 keepalived 节点的初始状态（MASTER|BACKUP） interface ens192         # VRRP 实例绑定的网口，用于发送 VRRP 包 virtual_router_id 200        # 路由 ID，范围是 0-255，主备都一样 priority 100          # 指定优先级，优先级高的将成为 MASTER advert_int 1          # 指定发送 VRRP 广播的间隔。单位是秒 nopreempt           # 设置为不抢占。默认是抢占的         authentication {          # 身份验证 auth_type PASS                                  # 指定认证方式 auth_pass mysql                                 # 指定认证所使用的密码 mysql ，主备都一样}track_script {           # 调用"vrrp_script"的脚本 chk_mysql           # 增加一个跟踪脚本到网口上}virtual_ipaddress {         # 虚拟 IP 192.168.1.38/24  } }

主库 A 检查脚本，编辑 /etc/keepalived/check_mysql.sh 文件：

    #!/bin/bash#/etc/keepalived/check_mysql.sh#chmod u+x /etc/keepalived/check_mysql.sh#Linux 7 使用，如果是配置Linux 6 需要修改脚本# MySQL账号密码mysql_user="root"mysql_pass="123456"# MySQL错误日志输出mysql_err="/mysql/log/3308/check_mysql_err.log"# MySQL杀进程脚本mysql_kill_session="/tmp/kill.sql"# MySQL连接字符串mysql_con="mysql -u${mysql_user} -p${mysql_pass} -S /mysql/data/3308/mysql.sock"source ~/.bash_profileif [ `ps -ef|grep -w "$0"|grep "/bin/sh*"|grep "?"|grep "?"|grep -v "grep"|wc -l` -gt 2 ];then  #    exit 0fifunction excute_query {    $mysql_con -e "select 1 from dual;" 2>> $mysql_err}function service_error {    echo -e "`date "+%F  %H:%M:%S"`    -----mysql service error，now stop keepalived-----" >> $mysql_err    echo -e "\n@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@\n" >> $mysql_err}function query_error {    echo -e "`date "+%F  %H:%M:%S"`    -----query error, but mysql service ok, retry after 30s-----" >> $mysql_err    sleep 30    excute_query    if [ $? -ne 0 ];then        echo -e "`date "+%F  %H:%M:%S"`    -----still can't execute query-----" >> $mysql_err        echo -e "`date "+%F  %H:%M:%S"`    -----set read_only = 1 on DB1-----" >> $mysql_err        $mysql_con -e "set global read_only = 1;" 2>> $mysql_err        echo -e "`date "+%F  %H:%M:%S"`    -----kill current client thread-----" >> $mysql_err        rm -f $mysql_kill_session &>/dev/null        $mysql_con -NB -e 'select concat("kill ",id,";") from  information_schema.PROCESSLIST where command="Query" or command="Execute"' > $mysql_kill_session        $mysql_con -e "source $mysql_kill_session"        sleep 2         echo -e "`date "+%F  %H:%M:%S"`    -----stop keepalived-----" >> $mysql_err        systemctl stop keepalived &>> $mysql_err        echo -e "\n@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@\n" >> $mysql_err    else        echo -e "`date "+%F  %H:%M:%S"`    -----query ok after 30s-----" >> $mysql_err        echo -e "\n@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@\n" >> $mysql_err    fi}excute_queryif [ $? -ne 0 ];then    systemctl status mysql &>/dev/null    if [ $? -ne 0 ];then        service_error    else        query_error    fifi

给检查脚本赋与执行权限：

    chmod u+x /etc/keepalived/check_mysql.sh

#### 主库 B 配置 Keepalived

主库 B keepalived 配置文件，编辑 /etc/keepalived/keepalived.conf 文件：

    global_defs {  router_id keep_mysql_repl_g1        # 负载均衡标识，在局域网内应该是唯一的 }# vrrp_instance vrrp_instance v_mysql_1 {           state BACKUP            # 指定该 keepalived 节点的初始状态（MASTER|BACKUP）    interface ens192                                         # VRRP 实例绑定的网口，用于发送 VRRP 包 virtual_router_id 200                                   # 路由ID，范围是0-255，主备都一样 priority 90                                             # 指定优先级，优先级高的将成为 MASTER advert_int 1                                            # 指定发送VRRP广播的间隔。单位是秒 nopreempt                                               # 设置为不抢占。默认是抢占的 authentication {            # 身份验证 auth_type PASS                                          # 指定认证方式 auth_pass mysql                                         # 指定认证所使用的密码 mysql ，主备都一样}notify_master /etc/keepalived/notify_master_mysql.sh  # 转换成 master 时，执行的脚本virtual_ipaddress {  192.168.1.38/24  } }

主库 B 脚本，当发生主从切换时，会执行该脚本。编辑 /etc/keepalived/notify_master_mysql.sh 文件：

    #!/bin/bash#/etc/keepalived/notify_master_mysql.sh#chmod u+x /etc/keepalived/notify_master_mysql.sh# MySQL账号密码mysql_user="root"mysql_pass="123456"# 配置更变日志change_log="/mysql/log/3308/state_change.log"# 主库B状态日志slave_status_log="/mysql/log/3308/slave_status_log.log"# MySQL连接字符串mysql_conn="mysql -u${mysql_user} -p${mysql_pass} -S /mysql/data/3308/mysql.sock"source ~/.bash_profileecho -e "`date "+%F  %H:%M:%S"`   -----keepalived change to MASTER-----" >> $change_logecho -e "`date "+%F  %H:%M:%S"`   ----------" >> $slave_status_log$mysql_conn -e "show slave status\G;" >> $slave_status_logSlave_IO_Running=`$mysql_conn -e "show slave status\G;"|egrep -w "Slave_IO_Running|Slave_SQL_Running"|awk 'NR==1{print}' | awk '{print $2}'`Slave_SQL_Running=`$mysql_conn -e "show slave status\G;"|egrep -w "Slave_IO_Running|Slave_SQL_Running"|awk 'NR==2{print}' | awk '{print $2}'`Master_Log_File=`$mysql_conn -e "show slave status\G;" |egrep -w "Master_Log_File|Read_Master_Log_Pos|Exec_Master_Log_Pos"|awk 'NR==1{print}' | awk '{print $2}'`Read_Master_Log_Pos=`$mysql_conn -e "show slave status\G;" |egrep -w "Master_Log_File|Read_Master_Log_Pos|Exec_Master_Log_Pos"|awk 'NR==2{print}' | awk '{print $2}'`Exec_Master_Log_Pos=`$mysql_conn -e "show slave status\G;" |egrep -w "Master_Log_File|Read_Master_Log_Pos|Exec_Master_Log_Pos"|awk 'NR==3{print}' | awk '{print $2}'`action() {    echo -e "`date "+%F  %H:%M:%S"`    -----set read_only = 0 on `hostname`-slave-----" >> $change_log    $mysql_conn -e "set global read_only = 0;" 2>> $change_log        $mysql_conn -e "stop slave;" 2>> $change_log    echo "`hostname`-slave keepalived 转为 MASTER 状态，线上数据库切换至`hostname`-slave" >> $change_log    echo -e "@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@\n" >> $change_log}if [ "$Slave_IO_Running" = "Yes" -a "$Slave_SQL_Running" = "Yes" ];then        if [ $Read_Master_Log_Pos = $Exec_Master_Log_Pos ];then            echo -e "`date "+%F  %H:%M:%S"`    -----Master_Log_File=$Master_Log_File . Exec_Master_Log_Pos($Exec_Master_Log_Pos) is equal Read_Master_Log_Pos($Read_Master_Log_Pos)" >> $change_log                        action                        $mysql_conn -e "reset slave all;" 2>> $change_log                        else                    echo -e "`date "+%F  %H:%M:%S"`    -----Master_Log_File=$Master_Log_File . Exec_Master_Log_Pos($Exec_Master_Log_Pos) is behind Read_Master_Log_Pos($Read_Master_Log_Pos), The waits time is more than 10s,now force change." >> $change_log                    sleep 10                        action                        $mysql_conn -e "reset slave all;" 2>> $change_log            exit 0        fiaction else    echo -e "`hostname`-slave's slave status is wrong,now force change. Master_Log_File=$Master_Log_File Read_Master_Log_Pos=$Read_Master_Log_Pos  Exec_Master_Log_Pos=$Exec_Master_Log_Pos" >> $change_log  actionfi

给脚本赋与执行权限：

    chmod u+x /etc/keepalived/notify_master_mysql.sh 

### 启动 Keepalived

在主库 A 和主库 B 上分别启动 keepalived。

    systemctl start keepalived

查看 keepalived 状态。

    [root@mysql-master keepalived-2.2.4]# systemctl status keepalived.service ● keepalived.service - LVS and VRRP High Availability Monitor   Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; vendor preset: disabled)   Active: active (running) since 五 2021-09-10 21:13:37 CST; 18s ago     Docs: man:keepalived(8)           man:keepalived.conf(5)           man:genhash(1)           https://keepalived.org  Process: 8184 ExecStart=/software/keepalived/sbin/keepalived $KEEPALIVED_OPTIONS (code=exited, status=0/SUCCESS) Main PID: 8186 (keepalived)   Memory: 680.0K   CGroup: /system.slice/keepalived.service           ├─8186 /software/keepalived/sbin/keepalived -D           └─8187 /software/keepalived/sbin/keepalived -D9月 10 21:13:41 mysql-master Keepalived_vrrp[8187]: Sending gratuitous ARP on ens192 for 192.168.1.389月 10 21:13:41 mysql-master Keepalived_vrrp[8187]: Sending gratuitous ARP on ens192 for 192.168.1.389月 10 21:13:41 mysql-master Keepalived_vrrp[8187]: Sending gratuitous ARP on ens192 for 192.168.1.389月 10 21:13:41 mysql-master Keepalived_vrrp[8187]: Sending gratuitous ARP on ens192 for 192.168.1.389月 10 21:13:46 mysql-master Keepalived_vrrp[8187]: (v_mysql_1) Sending/queueing gratuitous ARPs on ens192 for 192.168.1.389月 10 21:13:46 mysql-master Keepalived_vrrp[8187]: Sending gratuitous ARP on ens192 for 192.168.1.389月 10 21:13:46 mysql-master Keepalived_vrrp[8187]: Sending gratuitous ARP on ens192 for 192.168.1.389月 10 21:13:46 mysql-master Keepalived_vrrp[8187]: Sending gratuitous ARP on ens192 for 192.168.1.389月 10 21:13:46 mysql-master Keepalived_vrrp[8187]: Sending gratuitous ARP on ens192 for 192.168.1.389月 10 21:13:46 mysql-master Keepalived_vrrp[8187]: Sending gratuitous ARP on ens192 for 192.168.1.38

查看网卡地址，此时虚拟 IP 在主库 A 上。

    [root@mysql-master keepalived-2.2.4]# ip addr1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00    inet 127.0.0.1/8 scope host lo       valid_lft forever preferred_lft forever    inet6 ::1/128 scope host        valid_lft forever preferred_lft forever2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000    link/ether 00:50:56:8b:1b:ca brd ff:ff:ff:ff:ff:ff    inet 192.168.1.36/24 brd 192.168.1.255 scope global ens192       valid_lft forever preferred_lft forever    #虚拟 IP    inet 192.168.1.38/24 scope global secondary ens192       valid_lft forever preferred_lft forever    inet6 fe80::2556:f369:b4e7:fb64/64 scope link tentative dadfailed        valid_lft forever preferred_lft forever    inet6 fe80::6b8d:29f7:a5fe:dbee/64 scope link tentative dadfailed        valid_lft forever preferred_lft forever    inet6 fe80::f387:57a3:4975:d8f2/64 scope link tentative dadfailed        valid_lft forever preferred_lft forever

## 验证高可用

客户端通过虚拟 IP 192.168.1.38 连接数据库，通过 `select @@hostname` 命令可以看到当前连接的为主库 A。

    ❯ mysql -uroot -h 192.168.1.38 -P 3308  -p123456;mysql: [Warning] Using a password on the command line interface can be insecure.Welcome to the MySQL monitor.  Commands end with ; or \g.Your MySQL connection id is 268Server version: 5.7.29-log MySQL Community Server (GPL)Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.Oracle is a registered trademark of Oracle Corporation and/or itsaffiliates. Other names may be trademarks of their respectiveowners.Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.root@192.168.1.38 (none) 09:11:13>select @@hostname;+--------------+| @@hostname   |+--------------+| mysql-master |+--------------+1 row in set (0.01 sec)root@192.168.1.38 (none) 09:11:26>

客户端创建表并插入数据。

    create database testdb;create table testdb.data01( id int not null primary key auto_increment, name varchar(60), age int); insert into testdb.data01 (name,age) values('tom',18),('jack',17),('rock',16),('james',15),('cris',20);

此时分别登录主库 A 和主库 B 查看 testdb.data01 表中的数据，可以确定主库 A 和主库 B 目前数据是同步的。并且查看表中的内容可以发现主键是以 2 为间隔递增的，这是为了防止主从切换时插入数据产生主键冲突。主库 A 的主键会以 1，3，5，7，9 的序号递增。假如在序号为 9 时发生主从切换，新的主库（主库 A）的主键会以 10，12，14，16，18 的序号递增。

![](https://mmbiz.qpic.cn/mmbiz_png/vvsibFWkwqHqibb2w950E9c5QTdFsjxtEd6qoz2hpAB6qL535CDGnYU4JCoaAPVZq03D7VtO2x1eDH3JxEMB5ykA/640?wx_fmt=png)

### 停止主库 A，模拟故障切换

    [root@mysql-master ~]# systemctl stop mysql_3308.service 

在主库 A 的机器上查看 keepalived 状态，可以看到 keepalived 的优先级被设置为 0，此时虚拟 IP 将会飘到主库 B 的机器上。

    [root@mysql-master tmp]# systemctl status keepalived.service ● keepalived.service - LVS and VRRP High Availability Monitor   Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; vendor preset: disabled)   Active: active (running) since 五 2021-09-10 21:42:11 CST; 3min 59s ago     Docs: man:keepalived(8)           man:keepalived.conf(5)           man:genhash(1)           https://keepalived.org  Process: 13365 ExecStart=/software/keepalived/sbin/keepalived $KEEPALIVED_OPTIONS (code=exited, status=0/SUCCESS) Main PID: 13367 (keepalived)   Memory: 1.0M   CGroup: /system.slice/keepalived.service           ├─13367 /software/keepalived/sbin/keepalived -D           ├─13368 /software/keepalived/sbin/keepalived -D           ├─14761 /bin/bash /etc/keepalived/check_mysql.sh           └─14778 sleep 309月 10 21:42:20 mysql-master Keepalived_vrrp[13368]: Sending gratuitous ARP on ens192 for 192.168.1.389月 10 21:42:20 mysql-master Keepalived_vrrp[13368]: Sending gratuitous ARP on ens192 for 192.168.1.389月 10 21:42:20 mysql-master Keepalived_vrrp[13368]: Sending gratuitous ARP on ens192 for 192.168.1.389月 10 21:42:20 mysql-master Keepalived_vrrp[13368]: Sending gratuitous ARP on ens192 for 192.168.1.389月 10 21:42:20 mysql-master Keepalived_vrrp[13368]: Sending gratuitous ARP on ens192 for 192.168.1.389月 10 21:45:47 mysql-master Keepalived_vrrp[13368]: Track script chk_mysql is already running, expect idle - skipping run9月 10 21:45:47 mysql-master Keepalived_vrrp[13368]: VRRP_Script(chk_mysql) timed_out9月 10 21:45:47 mysql-master Keepalived_vrrp[13368]: (v_mysql_1) Entering FAULT STATE9月 10 21:45:47 mysql-master Keepalived_vrrp[13368]: (v_mysql_1) sent 0 priority9月 10 21:45:47 mysql-master Keepalived_vrrp[13368]: (v_mysql_1) removing VIPs.

查看主库 B 机器网卡的地址，发现虚拟 IP 已经切换到主库 B 上了。当发生主从切换时，主库 B 的脚本会执行 `reset slave all`，停止向主库 A 的同步，防止原主库 A 恢复后数据意外同步。

    [root@mysql-slave keepalived]# ip addr1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00    inet 127.0.0.1/8 scope host lo       valid_lft forever preferred_lft forever    inet6 ::1/128 scope host        valid_lft forever preferred_lft forever2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000    link/ether 00:50:56:8b:71:df brd ff:ff:ff:ff:ff:ff    inet 192.168.1.37/24 brd 192.168.1.255 scope global ens192    #虚拟 IP       valid_lft forever preferred_lft forever    inet 192.168.1.38/24 scope global secondary ens192       valid_lft forever preferred_lft forever    inet6 fe80::2556:f369:b4e7:fb64/64 scope link tentative dadfailed        valid_lft forever preferred_lft forever    inet6 fe80::6b8d:29f7:a5fe:dbee/64 scope link tentative dadfailed        valid_lft forever preferred_lft forever    inet6 fe80::f387:57a3:4975:d8f2/64 scope link tentative dadfailed        valid_lft forever preferred_lft forever

客户端发生了重连，通过 `select @@hostname` 查看可以看到此时连接的是主库 B。

    root@192.168.1.38 (none) 09:42:14>select @@hostname;#重连ERROR 2006 (HY000): MySQL server has gone awayNo connection. Trying to reconnect...Connection id:    19Current database: *** NONE ***+-------------+| @@hostname  |+-------------+| mysql-slave |+-------------+1 row in set (0.04 sec)

客户端插入几条数据：

    insert into testdb.data01 (name,age) values('peter',28),('mark',27),('marry',26),('hule',25),('handson',20);

查询数据，可以看到在原主库 B 上插入的数据主键会以 10，12，14，16，18 的序号递增。

    root@192.168.1.38 (none) 09:44:19>select * from testdb.data01;+----+---------+------+| id | name    | age  |+----+---------+------+|  1 | tom     |   18 ||  3 | jack    |   17 ||  5 | rock    |   16 ||  7 | james   |   15 ||  9 | cris    |   20 || 10 | peter   |   28 || 12 | mark    |   27 || 14 | marry   |   26 || 16 | hule    |   25 || 18 | handson |   20 |+----+---------+------+10 rows in set (0.01 sec)

### 重新启动主库 A，观察数据同步

由于我们关闭了抢占模式，当主库 A 重新启动时，主从不会发送切换。

    [root@mysql-master]# systemctl start mysql_3308.service

主库 A 的数据可以和主库 B 同步。

![](https://mmbiz.qpic.cn/mmbiz_png/vvsibFWkwqHqibb2w950E9c5QTdFsjxtEdtR2DM2rAMeJNDCM0k4x92CVhJ3sVZcIXhp1Xz5nEwfQV9MwJib2h0icA/640?wx_fmt=png)

## 参考资料

-   [https://www.cnblogs.com/kerrycode/p/11150782.html](https://www.cnblogs.com/kerrycode/p/11150782.html)
-   [http://www.linuxe.cn/post-492.html](http://www.linuxe.cn/post-492.html)
-   [https://blog.csdn.net/JesseYoung/article/details/41942809](https://blog.csdn.net/JesseYoung/article/details/41942809) 
    [https://mp.weixin.qq.com/s/yb0aIwLPxiJuFV0ZJfCNvA](https://mp.weixin.qq.com/s/yb0aIwLPxiJuFV0ZJfCNvA)
