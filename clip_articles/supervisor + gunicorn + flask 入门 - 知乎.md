# supervisor + gunicorn + flask 入门 - 知乎
对与数据挖掘、算法同学来说最痛苦的事莫过于：高并发的接口 + 完整（标准）的日志部署；

而提供一个高并发的接口，给内部开发同学调用，可以说是模型的最终呈现， _gunicorn + supervisor + flask_ 可以快速帮我们解决此问题；

对此做了如下总结；

**单线程接口：flask服务**
-----------------

Flask是python版的轻量级web框架，易于上手，可满足绝大部分开发需求。

### 1.1 安装Flask

### 1.2 搭建flask服务

安装完成后，在测试目录创建一个wsgi.py文件，添加如下脚本，保存（建议使用[PyCharm](https://link.zhihu.com/?target=https%3A//www.jetbrains.com/pycharm)测试）

```python3
#coding:utf-8
#wsgi.py
from flask import Flask

#服务命名为app
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello World!'

if __name__ == '__main__':
    from werkzeug.contrib.fixers import ProxyFix
    app.wsgi_app = ProxyFix(app.wsgi_app)
    app.run()
```

### 1.3 启动

然后在终端（shell或cmd）中，cd到测试目录下，执行它

执行后，控制台（console）打印如下信息：

```prolog
 * Serving Flask app "__init__" (lazy loading)
 * Environment: production
   WARNING: Do not use the development server in a production environment.
   Use a production WSGI server instead.
 * Debug mode: on
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 566-470-617
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
127.0.0.1 - - [07/Jan/2019 10:53:27] "**** HTTP/1.1" 200 -
```

用浏览器访问 [http://127.0.0.1:5000](https://link.zhihu.com/?target=http%3A//127.0.0.1%3A5000/) 就能看到网页显示hello world

**多线程异步接口：gunicorn 部署**
-----------------------

Gunicorn文档

运行`python wsgi.py`脚本，即可 web 访问，但无法并发访问，如果需要多线程开启 web 服务，实现并发访问，则需要额外一个工具进行封装，如gunicorn

### 1.1 安装

### 2.1 启动

```bash
$ gunicorn -w4 -b0.0.0.0:5000 wsgi:app
```

> 没错，就是这么简单，接口就可以多线程访问了；

**\-w：** 表示开启多少个 worker;

**\-b：** 表示 gunicorn 开发的访问地址;

**wsgi：** 为启动的python文件;

**app：** 为 app = Flask(\_\_name\_\_)的app;

### 2.2 配置文件

当参数过多时，或需要额外注释说明，gunicorn 还支持使用文件配置参数，运行代码调整为：

```bash
$ gunicorn -c pygun.py wsgi:app --log-level=debug --preload
```

> \--log-level：日志 _debug_ 模式；  
> \--preload：出现错误，可打印[错误详细信息](https://link.zhihu.com/?target=https%3A//stackoverflow.com/questions/24488891/gunicorn-errors-haltserver-haltserver-worker-failed-to-boot-3-django)；

[pygun.py](https://link.zhihu.com/?target=http%3A//pygun.py/) 文件配置参考如下：

```python3
bind = '127.0.0.1:5000' #gunicorn监控的接口
workers = 1 #进程数
threads = 1 #每个进程开启的线程数
proc_name = 'app'
pidfile = './app.pid' #gunicorn进程id，kill掉该文件的id，gunicorn就停止
loglevel = 'debug'
logfile = './debug.log' #debug日志
errorlog = './error.log' #错误信息日志
timeout = 90

#https://github.com/benoitc/gunicorn/issues/1194
keepalive = 75 # needs to be longer than the ELB idle timeout
worker_class = 'gevent' # 工作模式协程，详看https://ox0spy.github.io/post/web-test/gunicorn-worker-class/
##about timeout issuses
#https://github.com/benoitc/gunicorn/issues/1440
#https://github.com/globaldigitalheritage/arches-3d/issues/54
#https://github.com/benoitc/gunicorn/issues/588
#https://github.com/benoitc/gunicorn/issues/1194
#https://github.com/benoitc/gunicorn/issues/942
#https://stackoverflow.com/questions/10855197/gunicorn-worker-timeout-error

```

> _worker\_class_ 非协程模式，可能出现接口被调用失败；  
> _详看 [https://ox0spy.github.io/post/web-test/gunicorn-worker-class/](https://link.zhihu.com/?target=https%3A//ox0spy.github.io/post/web-test/gunicorn-worker-class/)_

运行后，打印如下信息：

```bash
$ gunicorn -c pygun.py wsgi:app --log-level=debug --preload
[2020-08-19 18:22:51 +0800] [53911]: [DEBUG] Current configuration:
  config: pygun.py  #配置文件名
  bind: ['127.0.0.1:5000']  #端口
  backlog: 2048
  workers: 1  #进程数
  worker_class: gevent  #工作模式
  threads: 1  #每个进程开启的线程数
  worker_connections: 1000
  max_requests: 0
  max_requests_jitter: 0
  timeout: 90  #超时，秒
  graceful_timeout: 30
  keepalive: 75
  limit_request_line: 4094
  limit_request_fields: 100
  limit_request_field_size: 8190
  reload: False
  reload_engine: auto
  reload_extra_files: []
  spew: False
  check_config: False
  preload_app: True
  sendfile: None
  reuse_port: False
  chdir: /data/bigData/test/  #工作路径
  daemon: False
  raw_env: []
  pidfile: None
  worker_tmp_dir: None
  user: 1002
  group: 1002
  umask: 0
  initgroups: False
  tmp_upload_dir: None
  secure_scheme_headers: {'X-FORWARDED-PROTOCOL': 'ssl', 'X-FORWARDED-PROTO': 'https', 'X-FORWARDED-SSL': 'on'}
  forwarded_allow_ips: ['127.0.0.1']
  accesslog: None
  disable_redirect_access_to_syslog: False
  access_log_format: %(h)s %(r)s %(s)s %(a)s %(L)s  #access日志配置
  errorlog: -
  loglevel: debug
  capture_output: False
  logger_class: gunicorn.glogging.Logger
  logconfig: None
  logconfig_dict: {} # logconfig_dict配置，此处省略，下文将讲到
  syslog_addr: udp://localhost:514
  syslog: False
  syslog_prefix: None
  syslog_facility: user
  enable_stdio_inheritance: False
  statsd_host: None
  dogstatsd_tags: 
  statsd_prefix: 
  proc_name: app
  default_proc_name: test:app
  pythonpath: None
  paste: None
  on_starting: <function OnStarting.on_starting at 0x7f4c69c9a710>
  on_reload: <function OnReload.on_reload at 0x7f4c69c9a830>
  when_ready: <function WhenReady.when_ready at 0x7f4c69c9a950>
  pre_fork: <function Prefork.pre_fork at 0x7f4c69c9aa70>
  post_fork: <function Postfork.post_fork at 0x7f4c69c9ab90>
  post_worker_init: <function PostWorkerInit.post_worker_init at 0x7f4c69c9acb0>
  worker_int: <function WorkerInt.worker_int at 0x7f4c69c9add0>
  worker_abort: <function WorkerAbort.worker_abort at 0x7f4c69c9aef0>
  pre_exec: <function PreExec.pre_exec at 0x7f4c69b81050>
  pre_request: <function PreRequest.pre_request at 0x7f4c69b81170>
  post_request: <function PostRequest.post_request at 0x7f4c69b81200>
  child_exit: <function ChildExit.child_exit at 0x7f4c69b81320>
  worker_exit: <function WorkerExit.worker_exit at 0x7f4c69b81440>
  nworkers_changed: <function NumWorkersChanged.nworkers_changed at 0x7f4c69b81560>
  on_exit: <function OnExit.on_exit at 0x7f4c69b81680>
  proxy_protocol: False
  proxy_allow_ips: ['127.0.0.1']
  keyfile: None
  certfile: None
  ssl_version: 2
  cert_reqs: 0
  ca_certs: None
  suppress_ragged_eofs: True
  do_handshake_on_connect: False
  ciphers: None
  raw_paste_global_conf: []
  strip_header_spaces: False
[2020-08-19 18:23:00 +0800] [53911]: [INFO] Starting gunicorn 20.0.4
[2020-08-19 18:23:00 +0800] [53911]: [DEBUG] Arbiter booted
[2020-08-19 18:23:00 +0800] [53911]: [INFO] Listening at: http://127.0.0.1:5000 (53911)
[2020-08-19 18:23:00 +0800] [53911]: [INFO] Using worker: gevent
[2020-08-19 18:23:00 +0800] [53911]: [DEBUG] 1 workers
[2020-08-19 18:23:00 +0800] [54100]: [INFO] Booting worker with pid: 54100

```

此时可以并发访问，但存在一个问题，就是gunicron不支持重启，如果脚本需更新、服务异常关闭等，那么如何重启呢？

**自动化管理：supervisor**
--------------------

gunicorn无法重启，关闭进程麻烦，因此还需要一个程序（如supervisor）来管理gunicorn，以达到自动化

而supervisor是一个专门用来管理进程的工具，还可以管理系统的工具进程，甚至可设置web页面管理相关进程；

### 3.1 安装

supervisor设置配置文件（supervisor.conf）还可以将脚本打印信息输出到指定文件内，设置内存上限可自动清理等等；

### 3.2 配置文件

创建配置文件的默认路径

```text
sudo mkdir /etc/supervisor
sudo mkdir /etc/conf.d
```

创建配置文件

```bash
$ cat /etc/supervisor/conf.d/wsgi.ini
[program:wsgi]   ;program:名称
;启动命令，当然你可以直接 python run.py，此处使用gunicorn启动
command=gunicorn pygun.conf wsgi:app --log-level=debug --preload
;工作目录（脚本启动目录的全路径）
directory=/data/test 
startsecs=0
stopwaitsecs=0
autostart=true          ;supervisord守护程序启动时自动启动tornado
autorestart=true        ;supervisord守护程序重启时自动重启tornado
redirect_stderr=true   ;将stderr重定向到stdout
;日志标准输出路径，同时脚本print打印信息也会在改文件显示
stdout_logfile=./stdout.log
stderr_logfile=./error.log


;守护进程，可在 web 上访问
[inet_http_server]     ; inet (TCP) server disabled by default
port=127.0.0.1:9001    ; (ip_address:port specifier, *:port for all iface)
username=wialsh        ; (default is no username (open server))
password=wialsh        ; (default is no password (open server))

;supervisord日志配置
[supervisord]
logfile=/tmp/supervisord.log ; main log file; default $CWD/supervisord.log
logfile_maxbytes=50MB        ; max main logfile bytes b4 rotation; default 50MB
logfile_backups=1
```

> 若无特别需求，只需修改启动命令（command）、部分文件（directory、stdout\_logfile、stderr\_logfile、logfile）路径和守护进程（inet\_http\_server），其他参数默认即可；

### 3.3 启动

完成配置后，执行如下脚本启动（**supervisord -c** 启动指定路径的配置）：

```console
$ supervisord -c /etc/supervisor/conf.d/wsgi.ini
```

> 关于 service supervisor start，详看此处；

在完成配置后，查看监控日志，以检查启动是否成功，可执行如下操作

```bash
$ less error.log  #gunicorn的errorlog路径
$ less /tmp/supervisord.log  #supervisor的logfile路径
```

查看状态（如无错误情况，可查看状态）

```bash
$ supervisorctl -c /etc/supervisor/conf.d/wsgi.ini status
wsgi                         RUNNING   pid 274203, uptime 0:17:35
```

重启服务

```text
$ supervisorctl -c /etc/supervisor/conf.d/wsgi.ini reload
Restarted supervisord
```

关闭服务

```text
$ supervisorctl -c /etc/supervisor/conf.d/wsgi.ini shutdown
Shut down
```

Supervisor详细配置请看：

### 3.5 端口监听

若需要开启守护进程进行页面管理Supervisor，则修改 /etc/supervisor/conf.d/wsgi.ini 里面TCP协议配置，

那么打开页面127.0.0.1:9001按设置帐号密码登录即可访问，取下如下代码注释，并修改账号密码端口号等：

```text
[inet_http_server]         ; inet (TCP) server disabled by default
port=127.0.0.1:9001        ; ip_address:port specifier, *:port for all iface
username=user              ; default is no username (open server)
password=123               ; default is no password (open server)
```

完成配置后，那么在启动后，可以检查到监听到端口

> 9001为inet\_http\_server设置的port 无则无信息打印，有则打印监听信息，同时可进入网址Supervisor地址进行管理（可启动、重启、停止），如下：

```text
COMMAND     PID USER   FD   TYPE    DEVICE SIZE/OFF NODE NAME
supervisor 62453 root    4u  IPv4 104725777      0t0  TCP 127.0.0.1:9001 (LISTEN)
```

登录[http://127.0.0.1:9001](https://link.zhihu.com/?target=http%3A//127.0.0.1%3A9001/)，即可访问Supervisor的守护进程，可启动、重启、停止

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img/2022-9-21%2019-08-39/d8de7126-5760-402f-bc74-0f2b4cc605f2.jpeg?raw=true)

结语
--

至此 supervisor + gunicorn + flask 服务启动介绍完毕，

*   flask 提供接口服务；
*   gunicorn启动flask服务，可进行多进程访问；
*   supervisor管理程序，对程序进行启动、重启、关闭的操作简单化，程序异常关闭，还能自动重启；  
    

另外，关于

*   supervisor出现bug问题，继续往下浏览查看；
*   supervisor + gunicorn + flask达到了自动化管理，但上述的日志配置比较简略，需要了解更多日志信息，请看：

*   supervisor + gunicorn + flask虽然达到自动化管理服务，如果服务器重启，但supervisor并不会重启，若需要在正式环境使用，则需要systemd来重启supervisor，另外supervisor + gunicorn可以管理多种服务（甚至非python脚本服务），感兴趣的同学请看：

* * *

关于supervisor一些错误定位信息
--------------------

附录：关于如何强制重启（不建议）

*   若服务需要重启，但目前正在被调用，执行

```text
$ supervisorctl -c /etc/supervisor/conf.d/wsgi.ini reload
```

*   重启可能无效，_error.log_ 打印端口正在被使用（如下），无法重启；

```bash
[2020-08-18 16:01:36 +0800] [146877] [ERROR] Connection in use: ('127.0.0.1', 5000)
[2020-08-18 16:01:36 +0800] [146877] [DEBUG] connection to ('127.0.0.1', 5000) failed: [Errno 98] Address already in use
```

*   此时在执行完重启命令后，再执行，强制关闭端口即可；

```text
$ kill -9 `lsof -ti:5000`
```

> 此方式不建议使用，这样会使调用者出现请求错误；

* * *

错误问题
----

一般错误问题容易解决或google后解决,当出现如下错误，代码出错，需要关闭，调试重启（参考点击[此处](https://link.zhihu.com/?target=https%3A//stackoverflow.com/questions/25121838/supervisor-on-debian-wheezy-another-program-is-already-listening-on-a-port-that)）

```prolog
Starting supervisor: Error: Another program is already listening on a port that one of our HTTP servers is configured to use.  Shut this program down first before 
starting supervisord.
For help, use /Users/wialsh/anaconda2/bin/supervisorctl -h

```

或

```bash
Error: .ini file does not include supervisorctl section
```

或

```prolog
Error: Another program is already listening on a port that one of our HTTP servers is configured to use.  Shut this program down first before starting supervisord.
```

或日志监控出现如下

```prolog
2014-08-04 16:25:45,891 CRIT Supervisor running as root (no user in config file)
2014-08-04 16:25:45,891 WARN Included extra file "/etc/supervisor/conf.d/com.domain.subdomain.conf" during parsing
```

  
解决方案：取消.sock文件，然后关闭进程

取消 **.sock** 文件（注意，.sock文件是实际配置的路径和文件名）

```bash
$ sudo unlink /var/run/supervisor.sock 
```

或 （默认supervisor设置路径文件）

```bash
$ sudo unlink /tmp/supervisor.sock
```

关闭进程

根据unix\_http\_server的file配置路径（详看supervisor.conf）

其中.sock需要配置，若未配置则不存在，那么可在终端中键入此项

```bash
$ ps -ef | grep supervisord
***       37731  15217  0 11:25 pts/21   00:00:00 grep --color=auto wsgi
root      62453      1  0 Jan05 ?        00:00:20 /usr/bin/python /usr/local/bin/supervisord -c /etc/supervisor/conf.d/wsgi.conf
```

> 不能使用ps -aux | grep 'supervisord'，返回的是子进程，关闭子进程仍会被重启

关闭（注意，别把其他监控关了）

```bash
$ sudo kill -s SIGTERM 62453 
```

若gunicorn日志监控出现如错误（参考解决方案点击[此处](https://link.zhihu.com/?target=https%3A//stackoverflow.com/questions/4465959/python-errno-98-address-already-in-use/4466035)）：

```bash
OSError: [Errno 98] Address already in use
```

查看占用的端口：

杀死占用的端口：

略

注意command=gunicorn -w4 -b0.0.0.0:2170 **wsgi:app** 的 **wsgi** 为启动文件名，否则抛出如下错误

```bash
Traceback (most recent call last):
  File "/usr/local/lib/python2.7/dist-packages/gunicorn/arbiter.py", line 578, in spawn_worker
    worker.init_process()
…
    return util.import_app(self.app_uri)
  File "/usr/local/lib/python2.7/dist-packages/gunicorn/util.py", line 352, in import_app
    __import__(module)
  File "/usr/local/lib/python2.7/dist-packages/gevent/builtins.py", line 93, in __import__
    result = _import(*args, **kwargs)
ImportError: No module named run
```

文件特殊字符错误：

```text
$ supervisord -c /etc/supervisor/conf.d/wsgi.conf 
Error: not a valid boolean value: 'true.' in section 'program:wsgi' (file: '.../wsgi.conf')
For help, use /data/bigData/anaconda3/bin/supervisord -h

```

此处错误的是，我在 _true_ 结尾处加入了 `.` ，所以报了无效 _bool_ 型。删除即可，此问题容易发现；

如果是全角空格等，也会报错上述错误，而且你查看配置文件时，并不能找到，因为在linux中不能被识别的空格、空值、换行（如windows换行 _\\r\\n_ 与linux不同 _\\n_），均会被识别为 _unicode_ ，所以一般建议删除配置中后面空格，以避免出现此类错误；

补充（2020-06-28）：

```bash
Usage: gunicorn [options]

gunicorn: error: no such option: -w

gunicorn: error: no such option: -b

gunicorn: error: no such option: -c

...
```

说明，_import_ 的包包含了参数解析包，如：_optparse_ 等，导致 _gunicorn_ 参数失效；

因此需要替换参数解析包；