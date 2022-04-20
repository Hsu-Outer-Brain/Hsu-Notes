# 记一次docker mysql 数据库迁移过程_Min_Monk的博客-程序员宅基地_docker mysql 迁移 - 程序员宅基地
## 环境说明

-   源数据库  
    一个在 linux 服务器上运行的 5.6 版本的 Mysql
-   目标数据库  
    一个在 docker 上运行的 Mysql 5.7 镜像

## 迁移过程

### 导出源数据

这里就需要引用的 mysql 官方的一个工具**mysqldump**，关于这个工具的详细使用可以自行百度，无非就是一些参数的配置。这里列出几个常用的配置

| 参数                     | 含义          | 备注          |
| ---------------------- | ----------- | ----------- |
| -h                     | 指定服务器 IP 地址 | 本机的时候可以省略   |
| -P                     | 指定服务器端口     | 本机的时候可以省略   |
| -u                     | 用户名         |             |
| -p                     | 密码          |             |
| –ignore-table          | 导出时候排除的表    | 用法可以参见下面的实例 |
| -default-character-set | 指定导出时用的字符集  | 用法可以参见下面的实例 |

由于数据库中的 ESB_INSTANCE 和 ESB_INSTANCE_DETAIL 中存在大量的大字段信息，并且只是一些日志数据，可以不用迁移过去，否则导出的文件会很大。导出语句如下：

```bash
./mysqldump -u root -p esb_console_db_dev > /home/mysql/esb_console_db_dev.sql --ignore-table=esb_console_db_dev.ESB_INSTANCE --ignore-table=esb_console_db_dev.ESB_INSTANCE_DETAIL --default-character-set=utf8

```

如果不知道 mysql 装在哪个目录下，执行指令`ps -ef | grep mysql`，就可以看到 mysql 的安装目录了

### 将数据导入到目标数据库

1.  首先使用 docker 拉取一个 mysql 的镜像。关于 docker 指令的介绍可以参见之前的文章 ------[Docker 入门—docker 常用操作指令及运行第一个容器](https://editor.csdn.net/md/?articleId=103912224)。（PS：由于小编实际操作的时候，虚拟机上已有一个 5.7 版本的实例，就没有再去拉取一个 5.6 版本的镜像，同时也是想试试两个不同版本之间迁移的时候会不会遇到什么问题。）
2.  在宿主机创建以下的文件夹，用作运行 mysql 镜像时和镜像之间建立映射
    -   `mkdir -p /opt/docker-mysql/conf`
    -   `mkdir -p /opt/docker-mysql/data`
    -   `mkdir -p /opt/docker-mysql/logs`
3.  执行以下命令，将镜像运行起来。

    ```bash
    docker run  --name mysql  -p 3306:3306 -v /opt/docker-mysql/conf:/etc/mysql/mysql.conf.d/  -v /opt/docker-mysql/data:/var/lib/mysql  -v /opt/docker-mysql/logs:/logs  -e MYSQL_ROOT_PASSWORD=AAaa1234 -d mysql:5.7 

    ```
4.  将第一步生成的**esb_console_db_dev.sql**上传到宿主机 / apps/Download 目录下，然后通过指令`docer cp /apps/Download/esb_console_db_dev.sql mysql 镜像 ID:/home`将导出的 sql 脚本复制到容器内部的 / home 目录下
5.  执行命令`docker exec -it mysql 镜像 ID /bin/bash`进入容器内部
6.  执行命令`mysql -u root -p`连接上 mysql
7.  创建同名的数据库`create database esb_console_db_dev;`
8.  切换数据库`use esb_console_db_dev;`
9.  执行命令开始导入`source /home/esb_console_db_dev.sql;`
10. 至此，数据库的迁移就完成了

* * *

但是事情肯定没有那么顺利，因为在使用的时候发现，导入到容器中的数据中文存在乱码。

## 问题解决方案

### 中文乱码问题

中文乱码十有八九是字符集出了问题，我们进入到容器内部，连接上 mysql 后，执行命令`show variables like "%char%";`来查看数据库当前的字符集。如图所示：![](https://img-blog.csdnimg.cn/2020031000053716.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L01pbl9Nb25r,size_16,color_FFFFFF,t_70)

那么下面开始设置字符集。  
1. 在宿主机中的 / opt/docker-mysql/conf 目录下新增文件**mysq.cnf**，并添加以下内容：

```properties
[client]
default-character-set=utf8

[mysql]
default-character-set=utf8

[mysqld]
init_connect='SET collation_connection = utf8_unicode_ci'
init_connect='SET NAMES utf8'
character-set-server=utf8
collation-server=utf8_unicode_ci
skip-character-set-client-handshake


```

2.  保存之后，重启 mysql 镜像，重新查看下字符集，就可以看到字符集已经更正过来了，如下图所示：![](https://img-blog.csdnimg.cn/20200310000715869.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L01pbl9Nb25r,size_16,color_FFFFFF,t_70)
3.  然后重新导入下就可以了。 
    [https://www.cxyzjd.com/article/Min_Monk/104764221](https://www.cxyzjd.com/article/Min_Monk/104764221)
