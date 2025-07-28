# 高可用的半同步主从复制MySQL集群_mysql 半同步具体实现-CSDN博客
[高可用的半同步主从复制 MySQL 集群\_mysql 半同步具体实现 - CSDN 博客](https://blog.csdn.net/TyreBurst/article/details/137522666) 

## 项目信息

### 项目结构

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/ab1a4bcf-1639-4ddb-8e48-8031f6e5e2cf.png?raw=true)

### 项目描述

构建一个高可用、读写分离的 MySQL 服务器集群，提高业务数据的稳定性，能够监控、批量部署和维护整个集群

### 项目环境

8 台虚拟机，centos7.9，MySQL-5.7.37，mysql-router-8.0.21，keepalived，Prometheus，ansible 等

## 项目步骤

### IP 规划

    master 			192.168.121.131
    slave1 			192.168.121.132
    slave2 			192.168.121.133
    slave3 			192.168.121.134
    ansible 		192.168.121.135
    mysqlrouter1 	192.168.121.136
    mysqlrouter2 	192.168.121.137
    sysbench 		192.168.121.138
    prometheus		192.168.121.139 

### 部署一台 ansible 服务器，搭建好免密通道并定义主机清单，在四台机器上批量安装 MySQL，配置好相关环境

#### 搭建 ssh 免密通道

```
`[root@ansible ~]
[root@ansible ~]
[root@ansible ~]
[root@ansible ~]
[root@ansible ~]
[root@ansible ~]
[root@ansible ~]

[root@ansible ~]
[root@ansible ansible]
ansible.cfg  hosts  roles
[root@ansible ansible]
[mysql]
192.168.121.131 
192.168.121.132 
192.168.121.133 
192.168.121.134` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/9cabfe60-693a-423e-b962-9de50a4b16e5.png?raw=true)



```

#### 使用 ansible 批量安装 MySQL

```
`[root@ansible ~]
anaconda-ks.cfg                             mysql-router-community-8.0.21-1.el7.x86_64.rpm
mysql-5.7.37-linux-glibc2.12-x86_64.tar.gz  onekey_install_mysql_binary.sh

[root@ansible ~]

yum install net-tools ncurses-devel gcc gcc-c++ vim libaio lsof cmake bzip2 openssl-devel ncurses-compat-libs -y
 

tar xf mysql-5.7.37-linux-glibc2.12-x86_64.tar.gz
 

mv mysql-5.7.37-linux-glibc2.12-x86_64 /usr/local/mysql
 

groupadd mysql
useradd -r -g mysql -s /bin/false mysql
 

service firewalld stop
systemctl  disable  firewalld
 

setenforce 0

sed -i '/^SELINUX=/ s/enforcing/disabled/'  /etc/selinux/config
 

mkdir  /data/mysql -p


chown mysql:mysql /data/mysql/

chmod 750 /data/mysql/


cd /usr/local/mysql/bin/
 

./mysqld  --initialize --user=mysql --basedir=/usr/local/mysql/  --datadir=/data/mysql  &>passwd.txt
 

./mysql_ssl_rsa_setup --datadir=/data/mysql/
 

tem_passwd=$(cat passwd.txt |grep "temporary"|awk '{print $NF}')
 

export PATH=/usr/local/mysql/bin/:$PATH

echo  'PATH=/usr/local/mysql/bin:$PATH' >>/root/.bashrc

cp  ../support-files/mysql.server   /etc/init.d/mysqld
 

sed  -i '70c  datadir=/data/mysql'  /etc/init.d/mysqld
 

cat  >/etc/my.cnf  <<EOF
[mysqld_safe]
 
[client]
socket=/data/mysql/mysql.sock
 
[mysqld]
socket=/data/mysql/mysql.sock
port = 3306
open_files_limit = 8192
innodb_buffer_pool_size = 512M
character-set-server=utf8
 
[mysql]
auto-rehash
prompt=\\u@\\d \\R:\\m  mysql>
EOF
 

ulimit -n 1000000

echo "ulimit -n 1000000" >>/etc/rc.local
chmod +x /etc/rc.d/rc.local
 

/sbin/chkconfig --add mysqld

/sbin/chkconfig mysqld on
 
 

service mysqld start

mysql -uroot -p$tem_passwd --connect-expired-password   -e  "set password='mysql123';"
 

mysql -uroot -p'mysql123' -e "show databases;"

[root@ansible ansible]
- hosts: mysql
  remote_user: root
  tasks:
  - name: copy file
    copy: src=/root/mysql-5.7.37-linux-glibc2.12-x86_64.tar.gz  dest=/root/
  - name: install mysql
    script: /root/onekey_install_mysql_binary_v3.sh
  - name: change path
    shell: export PATH=/usr/local/mysql/bin/:$PATH

[root@ansible ansible]
[root@ansible ansible]` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/b063b4d6-3efc-46c5-b89f-3a923d90855f.png?raw=true)



```

### 规划 MySQL 集群，一台做 master，三台做 slave

#### 配置 / etc/my.cnf

```
`在master机器上

[mysqld_safe]
 
[client]
socket=/data/mysql/mysql.sock
 
[mysqld]
socket=/data/mysql/mysql.sock
port = 3306
open_files_limit = 8192
innodb_buffer_pool_size = 512M
character-set-server=utf8


general_log
slow_query_log = 1
long_query_time = 0.001
log_bin
server_id = 1
expire_logs_days = 15
rpl_semi_sync_slave_enabled=1
log_slave_updates=ON


 
[mysql]
auto-rehash
prompt=\u@\d \R:\m  mysql>

在slave机器上
其他配置相同，server_id按顺序加一


service mysqld restart
Shutting down MySQL.. SUCCESS! 
Starting MySQL. SUCCESS!` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/fd043102-62b0-4316-b3e8-f34cc07cf246.png?raw=true)



```

### 使用 mysqldump 导出 master 的基础数据并传到 ansible 上，ansible 下发到所有的 slave 上。

```
 `mysql> grant replication slave on *.* to 'agent'@'%' identified by '123456';

[root@master ~]

[root@master ~]

ansible mysql -m copy -a "src=/root/all_db.sql dest=/root"

mysql -uroot -p'Sanchuang123#' <all_db.sql


reset master；
stop slave；
reset slave all；` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/e1e3a29a-0946-4599-84b7-caa163e206b9.png?raw=true)



```

### master 和 slave 上开启 GTID 功能，实现数据库的主从复制，对 slave3 配置延迟备份作为 backup ，从 slave1 上获取二进制日志

#### master 和 slave 上安装半同步插件

    mysql>INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_slave.so';

    mysql>INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';

    SELECT PLUGIN_NAME, PLUGIN_STATUS FROM INFORMATION_SCHEMA.PLUGINS WHERE PLUGIN_NAME LIKE '%semi%';" 

#### 修改 master 和 slave 配置

##### master 上的配置

```
`[root@master ~]
[mysqld_safe]
 
[client]
socket=/data/mysql/mysql.sock
 
[mysqld]
socket=/data/mysql/mysql.sock
port = 3306
open_files_limit = 8192
innodb_buffer_pool_size = 512M
character-set-server=utf8
 

log_bin
server_id = 1
 

rpl_semi_sync_master_enabled=1
rpl_semi_sync_master_timeout=1000
 

gtid-mode=ON
enforce-gtid-consistency=ON
 
[mysql]
auto-rehash
prompt=\u@\d \R:\m  mysql>

[root@master ~]

[root@master ~]

mysql>grant replication slave on *.* to 'slave'@'192.168.121.%' identified by 'mysql123';
 

mysql>flush privileges;
Query OK, 0 rows affected (0.01 sec)
 

mysql>reset master;
Query OK, 0 rows affected (0.01 sec)` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/ee48cd6c-f71a-412c-8a2b-6a3a04e5ae00.png?raw=true)



```

##### slave1、2 上的配置

```
 `mysql>INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';

[root@slave1 ~]
[mysqld_safe]
 
[client]
socket=/data/mysql/mysql.sock
 
[mysqld]
socket=/data/mysql/mysql.sock
port = 3306
open_files_limit = 8192
innodb_buffer_pool_size = 512M
character-set-server=utf8
 

log_bin

server_id = 2
 

rpl_semi_sync_master_enabled=1
rpl_semi_sync_master_timeout=1000
 
rpl_semi_sync_slave_enabled=1
log_slave_updates=ON
 

gtid-mode=ON
enforce-gtid-consistency=ON
 
[mysql]
auto-rehash
prompt=\u@\d \R:\m  mysql>

[root@slave1 ~]

[root@slave1 ~]

grant replication slave on *.* to 'slave'@'192.168.121.%' identified by 'mysql123';
 

reset slave all;
 

change master to master_host='192.168.121.131', 
master_user='slave',
master_password='mysql123',
master_port=3306,
master_auto_position=1;
 

mysql>start slave;
Query OK, 0 rows affected (0.00 sec)
mysql>show slave status\G;
Slave_IO_Running: Yes
Slave_SQL_Running: Yes` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/f27136c5-5bd2-495f-80e2-81cb2119ea8a.png?raw=true)



```

##### slave3(backup) 上的配置

```
`[root@slave3 ~]
[mysqld_safe]
 
[client]
socket=/data/mysql/mysql.sock
 
[mysqld]
socket=/data/mysql/mysql.sock
port = 3306
open_files_limit = 8192
innodb_buffer_pool_size = 512M
character-set-server=utf8

server_id = 4
 

rpl_semi_sync_slave_enabled=1
log_slave_updates=ON
 

gtid-mode=ON
enforce-gtid-consistency=ON
 
[mysql]
auto-rehash
prompt=\u@\d \R:\m  mysql>

[root@slave3 ~]

Shutting down MySQL.. SUCCESS! 
Starting MySQL. SUCCESS! 


mysql> change master to master_host='192.168.121.132', 
master_user='slave',
master_password='mysql123',
master_port=3306,
master_auto_position=1;
 
mysql>change master to master_delay = 600;
 
mysql>start slave;` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/24e38f33-73dc-4f17-9b97-c2bdceaa2e64.png?raw=true)



```

#### 测试

```
 `mysql>create database db1;
mysql>show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| db1                |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

mysql>show databases;
+--------------------+
| Database           |
+--------------------+
| db1                |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/f16df175-2bca-4954-ac5c-b9e1aeac537d.png?raw=true)



```

### backup 和 ansible 服务器之间建立双向免密通道，把 ansible 作为一台异地备份机

```
`[root@slave3 ~]
[root@slave3 ~]

[root@slave3 ~]

mkdir /backup

mysqldump -uroot -p'123456' --all-databases --triggers --routines --events >/backup/$(date +%Y%m%d%H%M%S)all_db.sql

scp /backup/$(date +%Y%m%d%H%M%S)all_db.sql 192.168.121.135:/backup

[root@slave3 ~]
0 */2 * * * bash /backup/db_backup.sh` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/66e381a8-e1e2-41b9-8605-6c9d2e766fa5.png?raw=true)



```

### master 和 slave 上部署 MHA，ansible 作为管理节点，实现自动的故障切换，确保 master 宕机时，自动提升一台 slave 为新的 master。

#### 安装 MHA

```
`[root@ansible ~]
[root@ansible ~]
[root@ansible masterha]
mha4mysql-node-0.56-0.el6.noarch.rpm
mha4mysql-manager-0.56-0.el6.noarch.rpm

[root@ansible masterha]

[root@ansible masterha]

[root@ansible ~]
[root@ansible ~]
[root@ansible ~]

[root@ansible ~]

ssh-keygen -t rsa
ssh-copy-id -i id_rsa.pub root@192.168.121.%

grant all privileges on *.* to 'monitor'@'192.168.121.%' identified by 'mysql123';

flush privileges;` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/19eec247-8abf-40ef-ad48-ad0b62e106ed.png?raw=true)



```

#### 配置 MHA

```
 ``[root@ansible ~]
[root@ansible ~]

[root@ansible ~]
[root@ansible bin]

 
use strict;
use warnings FATAL =>'all';
use Getopt::Long;
my (
$command,          $ssh_user,        $orig_master_host, $orig_master_ip,
$orig_master_port, $new_master_host, $new_master_ip,    $new_master_port
);

my $vip = '192.168.121.200'; 	
my $brdc = '192.168.121.255';		
my $ifdev = 'ens33';				
my $key = "1";
my $ssh_start_vip = "/sbin/ip a add $vip dev ens33:$key";
my $ssh_stop_vip = "/sbin/ip a del $vip dev ens33:$key";
my $exit_code = 0;
GetOptions(
    'command=s'          => \$command,
    'ssh_user=s'         => \$ssh_user,
    'orig_master_host=s' => \$orig_master_host,
    'orig_master_ip=s'   => \$orig_master_ip,
    'orig_master_port=i' => \$orig_master_port,
    'new_master_host=s'  => \$new_master_host,
    'new_master_ip=s'    => \$new_master_ip,
    'new_master_port=i'  => \$new_master_port,
);
 
exit &main();
 
sub main {
    if ( $command eq "stop" || $command eq "stopssh" ) {
        my $exit_code = 1;
        eval {
            print "\n\n\n***************************************************************\n";
            print "Disabling the VIP - $vip on old master: $orig_master_host\n";
            print "***************************************************************\n\n\n\n";
            &stop_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn "Got Error: $@\n";
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "start" ) {
        my $exit_code = 10;
        eval {
            print "\n\n\n***************************************************************\n";
            print "Enabling the VIP - $vip on new master: $new_master_host \n";
            print "***************************************************************\n\n\n\n";
            &start_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn $@;
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "status" ) {
        print "Checking the Status of the script.. OK \n";
        `ssh $ssh_user\@$orig_master_host \" $ssh_start_vip \"`;
        exit 0;
    }
    else {
        &usage();
        exit 1;
    }
}
  

sub start_vip() {
    `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
}
 

sub stop_vip() {
    `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
}
 
sub usage {　
    print "Usage: master_ip_failover –command=start|stop|stopssh|status –orig_master_host=host –orig_master_ip=ip –orig_master_port=port –new_master_host=host –new_master_ip=ip –new_master_port=port\n";
}

chmod 777 /usr/local/bin/master_failover`` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/80fa99d3-4353-4a35-b735-72a21a5d0533.png?raw=true)



```

##### 创建配置文件 / etc/mha/app.cnf

```
`[root@ansible ~]
[server default]

manager_workdir=/var/log/masterha/app

manager_log=/var/log/masterha/app/manager.log

master_binlog_dir=/data/mysql/

master_ip_failover_script=/usr/local/bin/master_failover
master_ip_online_change_script=/usr/local/bin/master_ip_online_change

user=monitor
password=mysql123

ping_interval=1
remote_workdir=/tmp

repl_user=replication
repl_password=mysql123
report_script=/usr/local/send_report

shutdown_script=""
ssh_user=root
 

[server1]
hostname=192.168.121.131
port=3306
 

[server2]
hostname=192.168.121.132
port=3306

candidate_master=1
check_repl_delay=0
 
[server3]
hostname=192.168.121.133
port=3306` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/b0dad501-3b7a-4702-8776-b9ff496b312c.png?raw=true)



```

##### 查看状态

```
 `[root@ansible .ssh]

[root@ansible .ssh]

[root@ansible .ssh]

[root@ansible .ssh]
[root@ansible .ssh]` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/0c5aec05-0bea-4221-9919-8408cf9d7d65.png?raw=true)



```

#### 测试，关闭 master 的 mysqld

```
 `[root@master ~]
 

[root@ansible ~]

***
master_ip_failover_script=/usr/local/bin/master_failover
master_ip_online_change_script=/usr/local/bin/master_ip_online_change
user=monitor 
password=mysql123
ping_interval=1
remote_workdir=/tmp
***

[server1]
hostname=192.168.121.131
port=3306` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/c520a5cf-d502-4f44-8411-a559f0244402.png?raw=true)



```

### 部署两台 mysqlrouter 中间件机器，实现读写分离。

#### 部署 mysqlrouter

```
 `[root@mysqlrouter1 ~]

[root@mysqlrouter1 ~]
[root@mysqlrouter1 mysqlrouter]
mysqlrouter.conf
[root@mysqlrouter1 mysqlrouter]

[routing:slaves]

bind_address = 192.168.121.136:7001
 

destinations = 192.168.121.132:3306,192.168.121.133:3306
mode = read-only
connect_timeout = 1
 

[routing:masters]

bind_address = 192.168.121.136:7002
 

destinations = 192.168.121.131:3306
mode = read-write
connect_timeout = 1


service firewalld stop
systemctl  disable  firewalld


setenforce 0
sed -i '/^SELINUX=/ s/enforcing/disabled/'  /etc/selinux/config

[root@mysqlrouter1 mysqlrouter]
Redirecting to /bin/systemctl start mysqlrouter.service

监听7001和7002端口
[root@mysqlrouter1 mysqlrouter]
tcp        0      0 192.168.121.136:7001    0.0.0.0:*               LISTEN      2261/mysqlrouter    
tcp        0      0 192.168.121.136:7002    0.0.0.0:*               LISTEN      2261/mysqlrouter` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/6ad1a9a7-2dd9-4921-b42f-a179843d542e.png?raw=true)



```

#### 测试读写分离

```
`在master上新建2个账号
一个可读可写
mysql>grant all on *.*  to 'write'@'%' identified by 'mysql123';

一个只读
mysql>grant select on *.*  to 'read'@'%' identified by 'mysql123';

mysql -h 192.168.121.136 -P 7001 -uread -p'myqsl123'

mysql>show databases;
+--------------------+
| Database           |
+--------------------+
| db1                |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
 
mysql>create database db2;
Access denied for user 'read'@'%' to database 'db2'\` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/de4ee6d8-1369-4c0b-acba-7a09ff454a5d.png?raw=true)



```

### mysqlrouter 机器上安装 keepalived ，配置 2 个 vrrp 实例，实现双 vip 的高可用功能。

```
 `[root@mysqlrouter1 ~]
[root@mysqlrouter1 ~]
[root@mysqlrouter1 keepalived]
keepalived.conf
[root@mysqlrouter1 keepalived]
! Configuration File for keepalived
：
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.121.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
  
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}
 
vrrp_instance VI_1 {
    state MASTER
    interface ens33  
    virtual_router_id 80 
    priority 200         
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.121.31
    }   
}
vrrp_instance VI_2 {
    state BACKUP
    interface ens33
    virtual_router_id 100
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.121.32
    }
}
[root@mysqlrouter1 keepalived]

[root@mysqlrouter2 ~]

! Configuration File for keepalived
 
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.121.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
  
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}
 
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 80
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.121.31
    }
}
vrrp_instance VI_2 {
    state MASTER
    interface ens33
    virtual_router_id 100
    priority 200
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.121.32
    }
}

[root@mysqlrouter2 ~]

[root@mysqlrouter1 keepalived]
Redirecting to /bin/systemctl stop keepalived.service
 
[root@mysqlrouter2 keepalived]
发现mysqlrouter1的vip漂移到mysqlrouter2上了` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/22ab06e0-040d-444a-b75c-efbaf334fe80.png?raw=true)



```

### 部署监控机，安装 prometheus 和 grafana 监控集群性能

#### 被监控的机器安装 mysqld_exporter

```
 `mysql>grant all on *.* to 'mysqld_exporter'@'%' identified by 'mysql123';

[root@slave1 ~]
[root@slave1 ~]

[root@slave1 ~]
[root@slave1 ~]

[root@slave1 ~]
[root@slave1 ~]
[root@slave1 ~]

[root@slave1 mysqld_exporter]
[client]
user=mysqld_exporter 
password=123456

[root@slave1 mysqld_exporter]
[root@slave1 mysqld_exporter]

[root@slave1 mysqld_exporter]

[root@slave1 ~]
[root@slave1 ~]` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/8b19661c-6f6a-4f52-8b87-96a046ff26e4.png?raw=true)



```

#### 部署 Prometheus

```
`[root@prometheus ~]
anaconda-ks.cfg prometheus-2.43.0.linux-amd64.tar.gz

[root@prometheus ~]
[root@prometheus ~]

[root@prometheus prometheus]
[root@prometheus prometheus]

[root@prometheus prometheus]
[root@prometheus prometheus]

[root@prometheus prometheus]
[root@prometheus prometheus]

[root@prometheus ~]
[Unit]
Description=prometheus
[Service]
ExecStart=/prom/prometheus/prometheus --config.file=/prom/promethe
us/prometheus.yml
ExecReload=/bin/kill -HUP $MAINPID
killMode=process
Restart=on-failure
[Install]
WantedBy=multi-user.target

[root@prometheus ~]
[root@prometheus prometheus]
scrape_configs:
  
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "mysqlrouter1"
    static_configs:
      - targets: ["192.168.121.136:9106"]
  - job_name: "mysqlrouter2"
    static_configs:
      - targets: ["192.168.121.137:9106"]
  - job_name: "master"
    static_configs:
      - targets: ["192.168.121.131:9106"]
  - job_name: "slave1"
    static_configs:
      - targets: ["192.168.121.132:9106"]
  - job_name: "slave2"
    static_configs:
      - targets: ["192.168.121.133:9106"]

[root@prometheus ~]
Redirecting to /bin/systemctl start prometheus.service` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/c61eb464-a57e-404a-b8f5-6c848013ab94.png?raw=true)



```

#### 安装 grafana

```
`[root@prometheus ~]

[root@prometheus ~]
Starting grafana-server (via systemctl):                   [  确定  ]

默认用户名:admin
默认密码:admin

Configuration --> Add data source -> 选择Prometheus
填 http://192.168.121.139:9090
dashboard -> import -> 导入14057模板 -> 选择Prometheus数据源` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2025-7-28%2017-16-14/b5d82449-04c9-448f-b33e-d808ecd1dbe4.png?raw=true)



```

### 部署压力测试机，使用 sysbench 软件测试集群的性能。

     [root@test ~]
    [root@test ~]

    [root@test ~]
    [root@test ~] 

### 项目心得

1\. 对集群架构的规划能力得到了提升  
2. 对 MySQL 集群有了更深入的理解  
3. 对 ansible 自动化运维有了新的认识  
4. 对 keepalived，MHA 等高可用组件有了更多的理解
