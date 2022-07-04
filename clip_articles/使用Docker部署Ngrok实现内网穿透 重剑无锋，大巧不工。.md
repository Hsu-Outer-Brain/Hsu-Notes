# 使用Docker部署Ngrok实现内网穿透 | 重剑无锋，大巧不工。
以前写过一篇文章， [ubuntu 安装 ngrok 并使用 nginx 代理](https://www.lylinux.org/ubuntu%E5%AE%89%E8%A3%85ngrok%E5%B9%B6%E4%BD%BF%E7%94%A8nginx%E4%BB%A3%E7%90%86.html)，介绍了在 ubuntu 上安装 ngrok，但是是需要在系统中安装 gvm 等依赖，而且过程还是挺麻烦的，正好今天有时间，于是试着使用 Docker 来部署下，下面介绍下如何部署。

## 准备工作

### 域名解析

首先，需要在你的域名提供商处增加两条 A 记录解析到你的服务器，比如我的是 `ngrok.lylinux.net`和`*.ngrok.lylinux.net`。这样你可以使用`subdomain`的方式，来实现穿透。

## 配置

目录结构如下图所示， ![](https://resource.lylinux.net/image/2018/06/16/Jietu20180616-020109@2x.jpg)

可以看到，有必须的`Dockerfile`文件，`build.sh`是编译 ngrok 的脚本，`config.yml`是客户端使用的配置文件，下面分别介绍下。

### Dockerfile

`FROM  golang:1.7.1-alpine
ADD  build.sh /
RUN  apk add --no-cache git make openssl
RUN  git clone https://github.com/inconshreveable/ngrok.git --depth=1 /ngrok
RUN  sh /build.sh
EXPOSE  8081
VOLUME  ["/ngrok"]
CMD  [  "/ngrok/bin/ngrokd"]` 

可以看到，我们是基于`golang:1.7.1-alpine`这个镜像来构建。

### build.sh

\`export NGROK_DOMAIN="ngrok.lylinux.net"
cd /ngrok/
openssl genrsa -out rootCA.key 2048
openssl req -x509 -new -nodes -key rootCA.key -subj "/CN=$NGROK_DOMAIN" -days 5000 -out rootCA.pem
openssl genrsa -out device.key 2048
openssl req -new -key device.key -subj "/CN=$NGROK_DOMAIN" -out device.csr
openssl x509 -req -in device.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out device.crt -days 5000
cp rootCA.pem assets/client/tls/ngrokroot.crt
cp device.crt assets/server/tls/snakeoil.crt
cp device.key assets/server/tls/snakeoil.key

make release-server
GOOS=linux GOARCH=386 make release-client
GOOS=linux GOARCH=amd64 make release-client
GOOS=windows GOARCH=386 make release-client
GOOS=windows GOARCH=amd64 make release-client
GOOS=darwin GOARCH=386 make release-client
GOOS=darwin GOARCH=amd64 make release-client
GOOS=linux GOARCH=arm make release-client\` 

注意，需要把上面的`NGROK_DOMAIN`修改为你自己的域名。

### 构建镜像

有了这两个文件之后，可以执行:  
`docker build -t ngrok .` 来构建镜像，如下图： ![](https://resource.lylinux.net/image/2018/06/16/Jietu20180616-020717@2x.jpg)
 等执行完成之后，可以使用如下命令来启动:

`docker run -it  -p 8081:8081 -p 4443:4443 -v /root/docker/ngrok/bin:/root/ngrok/bin/ -d ngrok /ngrok/bin/ngrokd -domain="ngrok.lylinux.net" -httpAddr=":8081"` 

同样的，修改上述的`domain`为你自己的域名。

### 客户端部分

上述命令执行完成之后，会输出容器 id，比如我的是`447314d83bfd`。可以使用:`docker inspect 447314d83bfd`命令来查看其详细信息，另外在`Mounts`节点可以看到挂载信息，如下图： ![](https://resource.lylinux.net/image/2018/06/16/Jietu20180616-021822@2x.jpg)
 在 Source 目录中的`bin`目录中可以找到编译出来的二进制客户端文件，找到我们需要执行的客户端对应平台就可以在客户端连接了。  
config.yml：

`server_addr: "ngrok.lylinux.net:4443"
trust_host_root_certs: false
tunnels:
  webapp:
    proto:
      http: 80
    subdomain: test` 

然后执行：

`./ngrok  -config=config.yml start-all` 

看到如下输出，则说明成功了。 ![](https://resource.lylinux.net/image/2018/06/16/Jietu20180616-022538@2x.jpg)
 最后一步，需要配置下 nginx，使用 nginx 反向代理，这样我们就可以使用 80 端口了。 下面只介绍下 nginx 配置，具体可以参考本文开始的文章。

### nginx 配置

`server  {server_name  ~^(?<subdomain>\w+)\.ngrok\.lylinux\.org$;
  listen  80;
  keepalive_timeout  70;
  proxy_set_header  "Host"  $host:8081;
  location  /  {
  proxy_pass_header  Server;
  proxy_redirect  off;
  proxy_pass  http://127.0.0.1:8081;
  }
  access_log  off;
  log_not_found  off;
}` 
 [https://www.lylinux.net/article/2018/9/18/51.html](https://www.lylinux.net/article/2018/9/18/51.html)
