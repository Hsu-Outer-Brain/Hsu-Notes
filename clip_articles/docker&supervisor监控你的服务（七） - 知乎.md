# docker&supervisor监控你的服务（七） - 知乎
*   系列其他内容

1.  [docker快速创建轻量级的可移植的容器](https://zhuanlan.zhihu.com/p/409186465)✓
2.  [docker&flask快速构建服务接口](https://zhuanlan.zhihu.com/p/409658311)✓
3.  docker&uwsgi高性能WSGI服务器生产部署必备✓
4.  docker&gunicorn高性能WSGI服务器生产部署必备✓
5.  docker&nginx&gunicorn实现负载均衡✓
6.  docker&ngxtop并实时解析nginx日志✓
7.  docker&supervisor监控你的服务✓
8.  docker&pyinstaller两步法构建小体积容器
9.  locust对你的服务做高并发测试
10.  postman热门的API调试工具

**环境依赖**
--------

*   本教程是基于redhat linux服务器的

```text
redhat: 4.8.5
python: 3.8.3
supervisor: 4.2.2
```

*   本文主要内容

*   通过supervisor管理docker上的服务效果展示
*   docker容器通过supervisor管理其启动的服务实战-各步骤拆解
*   supervisor安装使用及参数详细介绍

**Docker部署supervisor**
----------------------

### **通过supervisor管理容器内服务**

**1\. 服务管理展示**

*   我们可以看到，多个端口的服务都是在supervisor页面呈现的，可以在页面即对各个服务做，重启、停止以及查看日志的操作。
*   当服务比较多的时候，这种方式还是比较可取的

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img/2022-9-22%2019-16-35/16360633-8164-4d30-b132-a6b0f1828087.jpeg?raw=true)

**2\. 管理界面日志流查看**

*   此图是我们点击查看日志得到的界面，当有新的请求进入之后，日志页面的请求日志是会随之实时更新的

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img/2022-9-22%2019-16-35/1ee86afa-7094-47c9-845f-9d99c39814c5.jpeg?raw=true)

*   所有的服务都起来啦

*   以9090端口启动supervisor服务后，我们可以看到9093/9094/9095端口的gunicorn服务也是启动了的
*   下图展示的即是请求各个服务后，得到的请求日志

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img/2022-9-22%2019-16-35/220c517e-855d-4113-9414-9fbda6aa70af.jpeg?raw=true)

### **实战-步骤拆解**

**1\. python环境依赖**

*   对虚拟环境中，所涉及到的python库，导出库名和版本信息

```text
click==8.0.3
Flask==2.0.2
gunicorn==20.1.0
itsdangerous==2.0.1
Jinja2==3.0.3
MarkupSafe==2.0.1
supervisor==4.2.2
Werkzeug==2.0.2
```

**2\. Dockerfile**

*   这一部分可以省略啦`RUN echo_supervisord_conf > supervisor.conf`

```ini
FROM python:3.8 as build-supervisor
WORKDIR /home/env_supervisor/supervisor_test
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN rm -rf /var/cache/apk/*
RUN mkdir -p supervisor
RUN mkdir -p supervisor/conf.d

COPY . .

RUN pip install -r requirements.txt -q -i https://pypi.tuna.tsinghua.edu.cn/simple

EXPOSE 9090
CMD ["supervisord", "-c", "supervisor.conf"]
```

**3\. supervisor.conf文件**

*   注意事项

*   如果想要在docker的后台运行成功，需要设置`nodaemon=true`
*   如果想要可以直接查看日志，需要指定`redirect_stderr=true`

```ini
[program:gunicorn_main]
command=gunicorn -c gunicorn_conf.py flask_test1:app
directory=/home/env_supervisor/supervisor_test/jobs_test
startsecs=0
stopwaitsecs=0
autostart=true
autorestart=true
redirect_stderr=true
user=root
stdout_logfile=./log/stdout.log
stderr_logfile=./log/error.log

; 守护进程，可在 web 上访问
[inet_http_server]
port= :9090
; 指定用户名(默认是无用户名的)
username=xw
; 指定密码(默认是没有密码的)
password=xw123

;;supervisord日志配置
[supervisord]
nodaemon=true
; 主日志文件，默认是($CWD/supervisord.log)
logfile=/tmp/supervisord.log
; 主日志文件大小，默认是(50MB)
logfile_maxbytes=50MB
logfile_backups=1

[supervisorctl]

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

;包含其它配置文件
[include]
files = ./supervisor/conf.d/*.ini
```

**4\. 构建容器**

*   基础命令

*   判断端口是否占用`netstat -plntu |grep "9093|9094|9095|9090"`
*   构建名称为`test/docker_ngxtop:1.0`的镜像  
    docker build -t test/docker\_supervisor:1.0 .  
    

**5\. 所有服务都在一个容器里面**

*   结果说明

*   此种方式构建，仅仅服务9093的可以访问
*   若服务都在容器里面，则supervisor管理了多少服务，都需要通过`-p 9095:9095`指定并暴露端口才行，不指定则即使`expose 9095` 也无用。

```text
docker run -d -p 9090:9090 -p 9093:9093 -v /linux/supervisor_test/jobs_test:/container/supervisor_test/jobs_test --name docker_supervisor1 test/docker_supervisor:1.0 supervisord -c supervisor.conf
```

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img/2022-9-22%2019-16-35/12d389c6-7dcd-4282-a5f4-111568702482.jpeg?raw=true)

**6\. supervisor日志设置生效**

*   需要保证指定`supervisor`指定的日志目录和`gunicorn`或者其他服务重定向一致的才行

*   gunicorn中的请求的日志重定向

```text
"access_file": {
            "class": "logging.handlers.RotatingFileHandler",
            "maxBytes": 1024 * 1024 * 5,
            "backupCount": 2,
            "formatter": "access",
            "filename": "./log/2access.log",
        },
```

*   supervisor中的也应该改成一致的才行

```text
redirect_stderr=true
stdout_logfile=./log/2access.log
```

*   启动服务

*   此次，我们指定了9093和9094端口的服务

```text
docker run -d -p 9090:9090 -p 9095:9095 -p 9094:9094  -v /linux/supervisor_test/jobs_test:/container/supervisor_test/jobs_test --name docker_supervisor1 test/docker_supervisor:1.0 supervisord -c supervisor.conf
```

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img/2022-9-22%2019-16-35/1c3c5f7c-fff3-44ee-bfed-ef91c01ff82a.jpeg?raw=true)

*   结果说明

*   可以看到我们的日志是可以直接查看的了

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img/2022-9-22%2019-16-35/bda7fac7-d463-4618-a17c-4bcd53b873e9.jpeg?raw=true)

**supervisor介绍**
----------------

*   Supervisor：监控服务进程的工具；
*   Supervisor有两个可执行程序 ：supervisord 和 supervisorctl。

*   supervisord用来依据配置文件的策略管理后台守护进程；
*   supervisorctl管理员用于向后台守护进程发送“启动/重启/停止”等指令。

### **1、supervisor安装**

**安装**

*   直接通过python安装即可

*   `pip install supervisor`

**初始化配置**

*   初始化配置文件：

*   `echo_supervisord_conf > supervisor.conf`

**配置文件介绍**

```ini
[program:project]  # 项目名称
; 程序的启动目录  
directory = /home/Code/project 
; 启动命令
command = /home/bin/gunicorn -w 4 -worker-class gevent -bind 0.0.0.0:9600 app:app
; 启动的进程副本数量(默认:1) 
numprocs=1   
; 自启动 
autostart = true   
; 启动1秒后没有异常退出，就当作已经正常启动了 
startsecs = 1    
; 程序异常退出后自动重启   
autorestart = true   
; 启动失败自动重试次数，默认是3   
startretries = 3 
; 用哪个用户启动   
user = root   
; 把stderr重定向到stdout，默认 false   
redirect_stderr = true    
; stdout 日志文件大小，默认 50MB   
stdout_logfile_maxbytes = 20MB  
; stdout 日志文件备份数 
stdout_logfile_backups = 10  
; log日志
stdout_logfile=/home/log/gunicorn.log  
; 错误日志
stderr_logfile=/home/gunicorn.error     
```

### **2、supervisor常用命令**

*   supervisord参数介绍

*   `Usage: /usr/local/python3/bin/supervisord [options]`

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img/2022-9-22%2019-16-35/08f1f6be-7875-4299-840c-19633e68f48e.jpeg?raw=true)

*   supervisorctl 常用命令

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img/2022-9-22%2019-16-35/8dc1ddc4-22bb-4b7b-8dee-5334f7cc1706.jpeg?raw=true)

### **3、supervisor实战**

*   需要注意的地方

*   查看服务 `netstat -plntu |grep "9093|9094|9095|9090"`
*   另外，gunicorn最好不要设置日志重定向，否则supervisor获取不到标准输出的日志嘞

**创建文件夹**

*   创建文件目录

```text
sudo mkdir ./supervisor
sudo mkdir ./supervisor/conf.d
```

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img/2022-9-22%2019-16-35/96c53dea-5cfa-43fb-916d-4c0640c2e638.jpeg?raw=true)

**修改配置文件**

*   注意事项

*   supervisor的日志文件一般还是要给绝对路径；
*   `supervisorctl`和`supervisord`这两个内容是都需要的，即使是给空；
*   部分配置项，是可以放到子配置文件中的。

```ini
[program:gunicorn_main]
command=gunicorn -c gunicorn_conf.py flask_test1:app
directory=/home/env_supervisor/supervisor_test/jobs_test
startsecs=0
stopwaitsecs=0
autostart=true
autorestart=true
redirect_stderr=true
user=root
stdout_logfile=./log/stdout.log
stderr_logfile=./log/error.log

; 守护进程，可在 web 上访问
[inet_http_server]
port=ip1:9090
; 指定用户名(默认是无用户名的)
username=xw
; 指定密码(默认是没有密码的)
password=xw123

;;supervisord日志配置
[supervisord]
; 主日志文件，默认是($CWD/supervisord.log)
logfile=/tmp/supervisord.log
; 主日志文件大小，默认是(50MB)
logfile_maxbytes=50MB
logfile_backups=1

[supervisorctl]

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface


;包含其它配置文件
[include]
files = ./supervisor/conf.d/*.ini
```

**增加子配置文件**

*   文件可以放到`./supervisor/conf.d`下，以`.ini`作为扩展名

*   `gunicorn_test1.ini`

```ini
[program:gunicorn_test1] 
command=gunicorn -c gunicorn_conf1.py flask_test1:app
directory=/home/env_supervisor/supervisor_test/jobs_test
autostart=true
autorestart=true
redirect_stderr=true
user=root
stdout_logfile=./log/1stdout.log
stderr_logfile=./log/1error.log
```

*   `gunicorn_test2.ini`

```ini
[program:gunicorn_test2] 
command=gunicorn -c gunicorn_conf2.py flask_test1:app
directory=/home/env_supervisor/supervisor_test/jobs_test
autostart=true
autorestart=true
redirect_stderr=true
user=root
stdout_logfile=./log/2stdout.log
stderr_logfile=./log/2error.log
```

**启动supervisor**

*   启动

```ps1con
find / -name supervisord
/usr/local/python3/bin/supervisord -c supervisor.conf
```

*   管理界面如下  
    

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img/2022-9-22%2019-16-35/ad3dad33-409b-4d5e-b6e2-84870d4f97fa.jpeg?raw=true)

* * *

**★ 本公众号主要发布**

**● 深度学习实战**

**● 机器学习实战**

**● 算法工程师养成**

**● 建议扫码关注公众号哦！**

**谢谢关注不迷路**

**\*\*\* END \*\*\***