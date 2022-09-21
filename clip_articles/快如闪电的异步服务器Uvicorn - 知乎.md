# 快如闪电的异步服务器Uvicorn - 知乎
* * *

什么是 Uvicorn ？
-------------

答：Uvicorn 是基于 uvloop 和 httptools 构建的非常快速的 ASGI 服务器。

什么是 uvloop 和 httptools ？
------------------------

答： uvloop 用于替换标准库 asyncio 中的事件循环，使用 Cython 实现，它非常快，可以使 asyncio 的速度提高 2-4 倍。asyncio 不用我介绍吧，写异步代码离不开它。

httptools 是 nodejs HTTP 解析器的 Python 实现。

什么是 ASGI 服务器？
-------------

答： 异步网关协议接口，一个介于网络协议服务和 Python 应用之间的标准接口，能够处理多种通用的协议类型，包括 HTTP，HTTP2 和 WebSocket。

请简单介绍下 Uvicorn
--------------

答：目前，Python 仍缺乏异步的网关协议接口，ASGI 的出现填补了这一空白，现在开始，我们能够使用共同的标准为所有的异步框架来实现一些工具，ASGI 帮助 Python 在 Web 框架上和 Node.JS 及 Golang 相竟争，目标是获得高性能的 IO 密集型任务，ASGI 支持 HTTP2 和 WebSockets，WSGI 是不支持的。

Uvicorn 目前支持 HTTP1.1 和 WebSocket，计划支持 HTTP2。

使用方法：
-----

创建一个文件 example.py

```python
async def app(scope, receive, send):
    assert scope['type'] == 'http'
    await send({
        'type': 'http.response.start',
        'status': 200,
        'headers': [
            [b'content-type', b'text/plain'],
        ]
    })
    await send({
        'type': 'http.response.body',
        'body': b'Hello, world!',
    })
```

启动 Uvicorn

你也可以不使用命令行，直接运行你的脚本也是可以的，如下：

```python
import uvicorn

async def app(scope, receive, send):
    ...

if __name__ == "__main__":
    uvicorn.run("example:app", host="127.0.0.1", port=5000, log_level="info")
```

使用命令行时，你可以使用 `uvicorn --help` 来获取帮助。

```python
Usage: uvicorn [OPTIONS] APP

Options:
  --host TEXT                     Bind socket to this host.  [default:
                                  127.0.0.1]
  --port INTEGER                  Bind socket to this port.  [default: 8000]
  --uds TEXT                      Bind to a UNIX domain socket.
  --fd INTEGER                    Bind to socket from this file descriptor.
  --reload                        Enable auto-reload.
  --reload-dir TEXT               Set reload directories explicitly, instead
                                  of using the current working directory.
  --workers INTEGER               Number of worker processes. Defaults to the
                                  $WEB_CONCURRENCY environment variable if
                                  available. Not valid with --reload.
  --loop [auto|asyncio|uvloop|iocp]
                                  Event loop implementation.  [default: auto]
  --http [auto|h11|httptools]     HTTP protocol implementation.  [default:
                                  auto]
  --ws [auto|none|websockets|wsproto]
                                  WebSocket protocol implementation.
                                  [default: auto]
  --lifespan [auto|on|off]        Lifespan implementation.  [default: auto]
  --interface [auto|asgi3|asgi2|wsgi]
                                  Select ASGI3, ASGI2, or WSGI as the
                                  application interface.  [default: auto]
  --env-file PATH                 Environment configuration file.
  --log-config PATH               Logging configuration file.
  --log-level [critical|error|warning|info|debug|trace]
                                  Log level. [default: info]
  --access-log / --no-access-log  Enable/Disable access log.
  --use-colors / --no-use-colors  Enable/Disable colorized logging.
  --proxy-headers / --no-proxy-headers
                                  Enable/Disable X-Forwarded-Proto,
                                  X-Forwarded-For, X-Forwarded-Port to
                                  populate remote address info.
  --forwarded-allow-ips TEXT      Comma separated list of IPs to trust with
                                  proxy headers. Defaults to the
                                  $FORWARDED_ALLOW_IPS environment variable if
                                  available, or '127.0.0.1'.
  --root-path TEXT                Set the ASGI 'root_path' for applications
                                  submounted below a given URL path.
  --limit-concurrency INTEGER     Maximum number of concurrent connections or
                                  tasks to allow, before issuing HTTP 503
                                  responses.
  --backlog INTEGER               Maximum number of connections to hold in
                                  backlog
  --limit-max-requests INTEGER    Maximum number of requests to service before
                                  terminating the process.
  --timeout-keep-alive INTEGER    Close Keep-Alive connections if no new data
                                  is received within this timeout.  [default:
                                  5]
  --ssl-keyfile TEXT              SSL key file
  --ssl-certfile TEXT             SSL certificate file
  --ssl-version INTEGER           SSL version to use (see stdlib ssl module's)
                                  [default: 2]
  --ssl-cert-reqs INTEGER         Whether client certificate is required (see
                                  stdlib ssl module's)  [default: 0]
  --ssl-ca-certs TEXT             CA certificates file
  --ssl-ciphers TEXT              Ciphers to use (see stdlib ssl module's)
                                  [default: TLSv1]
  --header TEXT                   Specify custom default HTTP response headers
                                  as a Name:Value pair
  --help                          Show this message and exit.
 ```


## 使用进程管理器

使用进程管理器确保你以弹性方式运行运行多个进程，你可以执行服务器升级而不会丢弃客户端的请求。

一个进程管理器将会处理套接字设置，启动多个服务器进程，监控进程活动，监听进程重启、关闭等信号。

Uvicorn 提供一个轻量级的方法来运行多个工作进程，比如 `--workers 4`，但并没有提供进行的监控。

### 使用 Gunicorn

Gunicorn 是成熟的，功能齐全的服务器，Uvicorn 内部包含有 Guicorn 的 workers 类，允许你运行 ASGI 应用程序，这些 workers 继承了所有 Uvicorn 高性能的特点，并且给你使用 Guicorn 来进行进程管理。

这样的话，你可能动态增加或减少进程数量，平滑地重启工作进程，或者升级服务器而无需停机。

在生产环境中，Guicorn 大概是最简单的方式来管理 Uvicorn 了，生产环境部署我们推荐使用 Guicorn 和 Uvicorn 的 worker 类：

```python
gunicorn example:app -w 4 -k uvicorn.workers.UvicornWorker
```

执行上述命令将开户 4 个工作进程，其中 UvicornWorker 的实现使用 uvloop 和httptools 实现。在 PyPy 下运行，你可以使用纯 Python 实现，可以通过使用UvicornH11Worker 类来做到这一点。

```python
gunicorn -w 4 -k uvicorn.workers.UvicornH11Worker
```

Gunicorn 为 Uvicorn 提供了不同的配置选项集，但是一些配置暂不支持，如`--limit-concurrency` 。

### 使用 Supervisor

Supervisor 作为进程管理器，以下两点二选一：

*   使用其文件描述符将套接字移交给 uvicorn，supervisor 始终将文件描述符置 0，并且必须在本 fcgi-program 中进行设置。
*   为每个 uvicorn 进程使用 UNIX 套接字。

一个简单的主管配置可能看起来像这样： administratord.conf：

```python
[supervisord] 

[fcgi-program：uvicorn] 
socket = tcp：// localhost：8000 
命令= venv / bin / uvicorn --fd 0示例：App 
numprocs = 4 
process_name = uvicorn-％（process_num）d 
stdout_logfile = / dev /标准输出
stdout_logfile_maxbytes = 0
然后运行supervisord -n。
```

### 使用 Circus

使用 Circus 与 Supervisor 很类似。配置文件 circus.ini 如下：

```python
[watcher:web]
cmd = venv/bin/uvicorn --fd $(circus.sockets.web) example:App
use_sockets = True
numprocesses = 4

[socket:web]
host = 0.0.0.0
port = 8000
```

与 Nginx 部署
----------

Nginx 作为 Uvicorn 进程的代理并不是必须的，你可以使用 Nginx 做为负载均衡。推荐使用 Nginx 时配置请求头，如 `X-Forwarded-For`,`X-Forwarded-Proto`，以便 Uvicorn 识别出真正的客户端信息，如 IP 地址，scheme 等。这里有一个配置文件的样例：

```python
http {
  server {
    listen 80;
    client_max_body_size 4G;

    server_name example.com;

    location / {
      proxy_set_header Host $http_host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_redirect off;
      proxy_buffering off;
      proxy_pass http://uvicorn;
    }

    location /static {
      # path for static files
      root /path/to/app/static;
    }
  }

  upstream uvicorn {
    server unix:/tmp/uvicorn.sock;
  }

}
```

使用 HTTPS
--------

使用 HTTPS 证书是必须的，推荐使用免费的 Let's Encrypt。

```python
$ uvicorn example:app --port 5000 --ssl-keyfile=./key.pem --ssl-certfile=./cert.pem
```

使用 Gunicorn 也可以直接使用证书。

```python
$ gunicorn --keyfile=./key.pem --certfile=./cert.pem -k uvicorn.workers.UvicornWorker example:app
```

参考文档
----

1.  [官方文档-介绍](https://link.zhihu.com/?target=https%3A//www.uvicorn.org/)。
2.  [官方文档-部署](https://link.zhihu.com/?target=https%3A//www.uvicorn.org/deployment/)。

关注微信公众号：Python七号，和我一起学习 Python。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img/2022-9-21%2019-06-21/818ed1f2-10fc-4bfd-aaee-de2709aaeb05.png?raw=true)