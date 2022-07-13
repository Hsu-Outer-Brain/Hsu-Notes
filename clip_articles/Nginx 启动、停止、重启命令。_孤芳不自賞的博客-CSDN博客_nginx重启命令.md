# Nginx:启动、停止、重启命令。_孤芳不自賞的博客-CSDN博客_nginx重启命令
## 重启[nginx](https://so.csdn.net/so/search?q=nginx&spm=1001.2101.3001.7020)

-   nginx -s reload  ：修改配置后重新加载生效
-   nginx -s reopen  ：重新打开日志文件
-   nginx -t -c /path/to/nginx.conf 测试 nginx 配置文件是否正确

## 关闭 nginx

-   nginx -s stop  :  快速停止 nginx
-   quit                ：完整有序的停止 nginx

## 其他的停止 nginx 方式

-   ps -ef | grep nginx
-   kill -QUIT 主进程号     ：从容停止 Nginx
-   kill -TERM 主进程号   ：快速停止 Nginx
-   pkill -9 nginx               ：强制停止 Nginx

## 启动 nginx

-   nginx -c /path/to/nginx.conf

## 平滑重启 nginx

-   kill -HUP 主进程号

## 启动

 启动代码格式：nginx 安装目录地址 -c nginx 配置文件地址

例如：

````null
[root@LinuxServer sbin]# /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf```

**停止**
------

 nginx的停止有三种方式：

### 从容停止

1、查看进程号

```null
[root@LinuxServer ~]```

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWFnZXMyMDE1LmNuYmxvZ3MuY29tL2Jsb2cvODQ4NTUyLzIwMTYwMS84NDg1NTItMjAxNjAxMDIxODI3NDQ4NTQtMTI5MTA1MzUxNy5wbmc?x-oss-process=image/format,png)

2、杀死进程

```null
[root@LinuxServer ~]```

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWFnZXMyMDE1LmNuYmxvZ3MuY29tL2Jsb2cvODQ4NTUyLzIwMTYwMS84NDg1NTItMjAxNjAxMDIxODI2NTIzNTQtOTYwMjgxMjc0LnBuZw?x-oss-process=image/format,png)

### 快速停止

1、查看进程号

```null
[root@LinuxServer ~]```

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWFnZXMyMDE1LmNuYmxvZ3MuY29tL2Jsb2cvODQ4NTUyLzIwMTYwMS84NDg1NTItMjAxNjAxMDIxODMxMDM2NTEtMTg1OTQ1MzIwOC5wbmc?x-oss-process=image/format,png)

2、杀死进程

```null
[root@LinuxServer ~]```

```null
或 [root@LinuxServer ~]```

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWFnZXMyMDE1LmNuYmxvZ3MuY29tL2Jsb2cvODQ4NTUyLzIwMTYwMS84NDg1NTItMjAxNjAxMDIxODMzNDAwMTAtMjAyNDIxMjQ1MS5wbmc?x-oss-process=image/format,png)

### 强制停止

```null
[root@LinuxServer ~]```

**重启**
------

### 验证nginx配置文件是否正确

**方法一：进入nginx安装目录sbin下，输入命令./nginx -t**

看到如下显示nginx.conf syntax is ok

nginx.conf test is successful

说明配置文件正确！

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWFnZXMyMDE1LmNuYmxvZ3MuY29tL2Jsb2cvODQ4NTUyLzIwMTYwMS84NDg1NTItMjAxNjAxMDIxODQ2MzM0MzItMTI2ODc4MjMzOC5wbmc?x-oss-process=image/format,png)

****方法二：在启动命令-c前加-t****

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWFnZXMyMDE1LmNuYmxvZ3MuY29tL2Jsb2cvODQ4NTUyLzIwMTYwMS84NDg1NTItMjAxNjAxMDIxODUwMjMzODUtNDU2NjEyMTgwLnBuZw?x-oss-process=image/format,png)

### 重启Nginx服务

**方法一：进入nginx可执行目录sbin下，输入命令./nginx -s reload 即可**

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWFnZXMyMDE1LmNuYmxvZ3MuY29tL2Jsb2cvODQ4NTUyLzIwMTYwMS84NDg1NTItMjAxNjAxMDIxODU1MjEwNTctMTM0MTM4MDkwNS5wbmc?x-oss-process=image/format,png)

**方法二：查找当前nginx进程号，然后输入命令：kill -HUP 进程号 实现重启nginx服务**

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWFnZXMyMDE1LmNuYmxvZ3MuY29tL2Jsb2cvODQ4NTUyLzIwMTYwMS84NDg1NTItMjAxNjAxMDIxODU4MzgxNjctMjM0ODU2NTA2LnBuZw?x-oss-process=image/format,png)

启动
--

cd /usr/local/nginx/sbin  
./nginx  
nginx服务启动后默认的进程号会放在/usr/local/nginx/logs/nginx.pid文件  
cat nginx.pid 查看进程号

关闭
--

*   kill -TERM pid  快速停止服务
*   kill -QUIT pid  平缓停止服务
*   kill -9 pid     强制停止服务

重启
--

cd /usr/local/nginx  
./nginx -HUP pid  
./nginx -s reload

./nginx -h 查看nginx所有的命令参数

| options | 说明 |
| --- | --- |
| \-?,-h | this help |
| \-v  | 显示nginx的版本号 |
| \-V | 显示nginx的版本号和编译信息 |
| \-t | 检查nginx配置文件的正确性 |
| \-T | 检查nginx配置文件的正确定及配置文件的详细配置内容 |
| \-q | suppress non-error messages during configuration testing |
| \-s signal | 向主进程发送信号，如:./nginx -s reload 配置文件变化后重新加载配置文件并重启nginx服务 |
| \-p prefix | 设置nginx的安装路径 |
| \-c filename | 设置nginx配置文件的路径 |
| \-g directives | 设置配置文件之外的全局指令 | 
 [https://blog.csdn.net/en_joker/article/details/107978716](https://blog.csdn.net/en_joker/article/details/107978716)
````
