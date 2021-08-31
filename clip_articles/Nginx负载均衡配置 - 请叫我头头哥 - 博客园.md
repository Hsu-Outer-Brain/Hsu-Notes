# Nginx负载均衡配置 - 请叫我头头哥 - 博客园
v 博客前言

> Nginx 是一款轻量级的 Web 服务器 / 反向代理服务器及电子邮件（IMAP/POP3）代理服务器，在 BSD-like 协议下发行。其特点是占有内存少，并发能力强，事实上 nginx 的并发能力在同类型的网页服务器中表现较好。
>
> 本篇文章主要介绍 nginx 复杂均衡配置，以及配置过程中遇到的一些问题。负载用的实例以 docker 为主。

## v 安装 nginx

`yum -y install nginx`

注意：若在这一步，遇到错误提示 `没有可用软件包 nginx。` ，可以看看文章[《yum install nginx - 没有可用软件包 nginx。》](https://www.cnblogs.com/toutou/p/yum_install_nginx.html "Nginx 负载均衡配置 ")提供了详细的解决办法。

安装完毕查一下 nginx 版本号 `nginx -v`

顺便通过 `rpm -ql nginx` 查看 nginx 的安装目录。/etc/nginx 主要都是配置文件，/var/log/nginx 主要都是 nginx 相关日志。

![](https://img2020.cnblogs.com/blog/506684/202011/506684-20201127165739934-338724917.png)

图片来源于网络，侵删。

启动 Nginx ： `systemctl start nginx`

若开启了防火墙，可以先关闭防火墙。 `systemctl stop firewalld`

设置 nginx 自动启动： `systemctl enable nginx`

启动 nginx: `systemctl start nginx`

关闭 nginx： `systemctl stop nginx`

重启 nginx： `systemctl restart nginx`

查看 nginx： `systemctl status nginx`

查看 nginx 失败原因： `systemctl status nginx.service` 或者 `journalctl -xe` 。这个很有用，在配置 nginx.conf 时配错了，启动 nginx 起不来，可以通过这个命令查看。

## v 详解 nginx.conf

其实我们常说的 Nginx 负载均衡配置，Nginx 负载均衡配置。 一般主要就是配置 nginx.conf。

2.0 nginx.conf 默认配置

![](https://images.cnblogs.com/OutliningIndicators/ContractedBlock.gif)

\# For more information on configuration, see:

# \* Official English Documentation: [http://nginx.org/en/docs/](http://nginx.org/en/docs/)

# \* Official Russian Documentation: [http://nginx.org/ru/docs/](http://nginx.org/ru/docs/)

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.

include /usr/share/nginx/modules/\*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user \[$time_local] "$request" ' '$status $body_bytes_sent "$http_referer"''"$http_user_agent" "$http_x_forwarded_for"';

    access\_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp\_nopush          on;
    tcp\_nodelay         on;
    keepalive\_timeout   65;
    types\_hash\_max\_size 2048;

    include             /etc/nginx/mime.types;
    default\_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx\_core\_module.html#include
    # for more information.
    include /etc/nginx/conf.d/\*.conf;

    server {
        listen       80 default\_server;
        listen       \[::\]:80 default\_server;
        server\_name  \_;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/\*.conf;

        location / {
        }

        error\_page 404 /404.html;
        location = /404.html {
        }

        error\_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }

# Settings for a TLS enabled server.

# 

# server {

# listen       443 ssl http2 default_server;

# listen       \[::]:443 ssl http2 default_server;

# server_name  \_;

# root         /usr/share/nginx/html;

# 

# ssl_certificate "/etc/pki/nginx/server.crt";

# ssl_certificate_key "/etc/pki/nginx/private/server.key";

# ssl_session_cache shared:SSL:1m;

# ssl_session_timeout  10m;

# ssl_ciphers HIGH:!aNULL:!MD5;

# ssl_prefer_server_ciphers on;

# 

# # Load configuration files for the default server block.

# include /etc/nginx/default.d/\*.conf;

# 

# location / {

# }

# 

# error_page 404 /404.html;

# location = /404.html {

# }

# 

# error_page 500 502 503 504 /50x.html;

# location = /50x.html {

# }

# }

}

View Code

2.1 nginx.conf 文件目录

![](https://common.cnblogs.com/images/copycode.gif)

...              #全局块

events {         #events 块
   ...
}

http      #http 块
{
    ...   #http 全局块
    server        #server 块
    { 
        ...       #server 全局块
        location \[PATTERN]   #location 块
        {...}
        location \[PATTERN] 
        {...}
    }
    server
    {
      ...
    }
    ...     #http 全局块
}

![](https://common.cnblogs.com/images/copycode.gif)

2.2 nginx.conf 详解：

![](https://common.cnblogs.com/images/copycode.gif)

\######Nginx 配置文件 nginx.conf 中文详解 #####

\#定义 Nginx 运行的用户和用户组
user www www;

\#nginx 进程数，建议设置为等于 CPU 总核心数。
worker_processes 8;

\#全局错误日志定义类型，\[ debug | info | notice | warn | error | crit ]
error_log /usr/local/nginx/logs/error.log info;

\#进程 pid 文件
pid /usr/local/nginx/logs/nginx.pid;

\#指定进程可以打开的最大描述符：数目
\# 工作模式与连接数上限
\# 这个指令是指当一个 nginx 进程打开的最多文件描述符数目，理论值应该是最多打开文件数（ulimit -n）与 nginx 进程数相除，但是 nginx 分配请求并不是那么均匀，所以最好与 ulimit -n 的值保持一致。
\# 现在在 linux 2.6 内核下开启文件打开数为 65535，worker_rlimit_nofile 就相应应该填写 65535。
\# 这是因为 nginx 调度时分配请求到进程并不是那么的均衡，所以假如填写 10240，总并发量达到 3-4 万时就有进程可能超过 10240 了，这时会返回 502 错误。
worker_rlimit_nofile 65535;

events
{
    \#参考事件模型，use \[ kqueue | rtsig | epoll | /dev/poll | select | poll ]; epoll 模型
    \#是 Linux 2.6 以上版本内核中的高性能网络 I/O 模型，linux 建议 epoll，如果跑在 FreeBSD 上面，就用 kqueue 模型。
    \#补充说明：
    \#与 apache 相类，nginx 针对不同的操作系统，有不同的事件模型
    \#A）标准事件模型
    \#Select、poll 属于标准事件模型，如果当前系统不存在更有效的方法，nginx 会选择 select 或 poll
    \#B）高效事件模型
    \#Kqueue：使用于 FreeBSD 4.1+, OpenBSD 2.9+, NetBSD 2.0 和 MacOS X. 使用双处理器的 MacOS X 系统使用 kqueue 可能会造成内核崩溃。
    \#Epoll：使用于 Linux 内核 2.6 版本及以后的系统。
    \#/dev/poll：使用于 Solaris 7 11/99+，HP/UX 11.22+ (eventport)，IRIX 6.5.15+ 和 Tru64 UNIX 5.1A+。
    \#Eventport：使用于 Solaris 10。 为了防止出现内核崩溃的问题， 有必要安装安全补丁。
    use epoll;

    #单个进程最大连接数（最大连接数=连接数\*进程数）
    #根据硬件调整，和前面工作进程配合起来用，尽量大，但是别把cpu跑到100%就行。每个进程允许的最多连接数，理论上每台nginx服务器的最大连接数为。
    worker\_connections 65535;

    #keepalive超时时间。
    keepalive\_timeout 60;

    #客户端请求头部的缓冲区大小。这个可以根据你的系统分页大小来设置，一般一个请求头的大小不会超过1k，不过由于一般系统分页都要大于1k，所以这里设置为分页大小。
    #分页大小可以用命令getconf PAGESIZE 取得。
    #\[root@web001 ~\]# getconf PAGESIZE
    #4096
    #但也有client\_header\_buffer\_size超过4k的情况，但是client\_header\_buffer\_size该值必须设置为“系统分页大小”的整倍数。
    client\_header\_buffer\_size 4k;

    #这个将为打开文件指定缓存，默认是没有启用的，max指定缓存数量，建议和打开文件数一致，inactive是指经过多长时间文件没被请求后删除缓存。
    open\_file\_cache max=65535 inactive=60s;

    #这个是指多长时间检查一次缓存的有效信息。
    #语法:open\_file\_cache\_valid time 默认值:open\_file\_cache\_valid 60 使用字段:http, server, location 这个指令指定了何时需要检查open\_file\_cache中缓存项目的有效信息.
    open\_file\_cache\_valid 80s;

    #open\_file\_cache指令中的inactive参数时间内文件的最少使用次数，如果超过这个数字，文件描述符一直是在缓存中打开的，如上例，如果有一个文件在inactive时间内一次没被使用，它将被移除。
    #语法:open\_file\_cache\_min\_uses number 默认值:open\_file\_cache\_min\_uses 1 使用字段:http, server, location  这个指令指定了在open\_file\_cache指令无效的参数中一定的时间范围内可以使用的最小文件数,如果使用更大的值,文件描述符在cache中总是打开状态.
    open\_file\_cache\_min\_uses 1;

    #语法:open\_file\_cache\_errors on | off 默认值:open\_file\_cache\_errors off 使用字段:http, server, location 这个指令指定是否在搜索一个文件是记录cache错误.
    open\_file\_cache\_errors on;

}

\#设定 http 服务器，利用它的反向代理功能提供负载均衡支持
http
{
    \#文件扩展名与文件类型映射表
    include mime.types;

    #默认文件类型
    default\_type application/octet-stream;

    #默认编码
    #charset utf-8;

    #服务器名字的hash表大小
    #保存服务器名字的hash表是由指令server\_names\_hash\_max\_size 和server\_names\_hash\_bucket\_size所控制的。参数hash bucket size总是等于hash表的大小，并且是一路处理器缓存大小的倍数。在减少了在内存中的存取次数后，使在处理器中加速查找hash表键值成为可能。如果hash bucket size等于一路处理器缓存的大小，那么在查找键的时候，最坏的情况下在内存中查找的次数为2。第一次是确定存储单元的地址，第二次是在存储单元中查找键 值。因此，如果Nginx给出需要增大hash max size 或 hash bucket size的提示，那么首要的是增大前一个参数的大小.
    server\_names\_hash\_bucket\_size 128;

    #客户端请求头部的缓冲区大小。这个可以根据你的系统分页大小来设置，一般一个请求的头部大小不会超过1k，不过由于一般系统分页都要大于1k，所以这里设置为分页大小。分页大小可以用命令getconf PAGESIZE取得。
    client\_header\_buffer\_size 32k;

    #客户请求头缓冲大小。nginx默认会用client\_header\_buffer\_size这个buffer来读取header值，如果header过大，它会使用large\_client\_header\_buffers来读取。
    large\_client\_header\_buffers 4 64k;

    #设定通过nginx上传文件的大小
    client\_max\_body\_size 8m;

    #开启高效文件传输模式，sendfile指令指定nginx是否调用sendfile函数来输出文件，对于普通应用设为 on，如果用来进行下载等应用磁盘IO重负载应用，可设置为off，以平衡磁盘与网络I/O处理速度，降低系统的负载。注意：如果图片显示不正常把这个改成off。
    #sendfile指令指定 nginx 是否调用sendfile 函数（zero copy 方式）来输出文件，对于普通应用，必须设为on。如果用来进行下载等应用磁盘IO重负载应用，可设置为off，以平衡磁盘与网络IO处理速度，降低系统uptime。
    sendfile on;

    #开启目录列表访问，合适下载服务器，默认关闭。
    autoindex on;

    #此选项允许或禁止使用socke的TCP\_CORK的选项，此选项仅在使用sendfile的时候使用
    tcp\_nopush on;
     
    tcp\_nodelay on;

    #长连接超时时间，单位是秒
    keepalive\_timeout 120;

    #FastCGI相关参数是为了改善网站的性能：减少资源占用，提高访问速度。下面参数看字面意思都能理解。
    fastcgi\_connect\_timeout 300;
    fastcgi\_send\_timeout 300;
    fastcgi\_read\_timeout 300;
    fastcgi\_buffer\_size 64k;
    fastcgi\_buffers 4 64k;
    fastcgi\_busy\_buffers\_size 128k;
    fastcgi\_temp\_file\_write\_size 128k;

    #gzip模块设置
    gzip on; #开启gzip压缩输出
    gzip\_min\_length 1k;    #最小压缩文件大小
    gzip\_buffers 4 16k;    #压缩缓冲区
    gzip\_http\_version 1.0;    #压缩版本（默认1.1，前端如果是squid2.5请使用1.0）
    gzip\_comp\_level 2;    #压缩等级
    gzip\_types text/plain application/x-javascript text/css application/xml;    #压缩类型，默认就已经包含textml，所以下面就不用再写了，写上去也不会有问题，但是会有一个warn。
    gzip\_vary on;

    #开启限制IP连接数的时候需要使用
    #limit\_zone crawler $binary\_remote\_addr 10m;



    #负载均衡配置
    upstream jh.w3cschool.cn {
     
        #upstream的负载均衡，weight是权重，可以根据机器配置定义权重。weigth参数表示权值，权值越高被分配到的几率越大。
        server 192.168.80.121:80 weight=3;
        server 192.168.80.122:80 weight=2;
        server 192.168.80.123:80 weight=3;

        #nginx的upstream目前支持4种方式的分配
        #1、轮询（默认）
        #每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。
        #2、weight
        #指定轮询几率，weight和访问比率成正比，用于后端服务器性能不均的情况。
        #例如：
        #upstream bakend {
        #    server 192.168.0.14 weight=10;
        #    server 192.168.0.15 weight=10;
        #}
        #2、ip\_hash
        #每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，可以解决session的问题。
        #例如：
        #upstream bakend {
        #    ip\_hash;
        #    server 192.168.0.14:88;
        #    server 192.168.0.15:80;
        #}
        #3、fair（第三方）
        #按后端服务器的响应时间来分配请求，响应时间短的优先分配。
        #upstream backend {
        #    server server1;
        #    server server2;
        #    fair;
        #}
        #4、url\_hash（第三方）
        #按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，后端服务器为缓存时比较有效。
        #例：在upstream中加入hash语句，server语句中不能写入weight等其他的参数，hash\_method是使用的hash算法
        #upstream backend {
        #    server squid1:3128;
        #    server squid2:3128;
        #    hash $request\_uri;
        #    hash\_method crc32;
        #}

        #tips:
        #upstream bakend{#定义负载均衡设备的Ip及设备状态}{
        #    ip\_hash;
        #    server 127.0.0.1:9090 down;
        #    server 127.0.0.1:8080 weight=2;
        #    server 127.0.0.1:6060;
        #    server 127.0.0.1:7070 backup;
        #}
        #在需要使用负载均衡的server中增加 proxy\_pass http://bakend/;

        #每个设备的状态设置为:
        #1.down表示单前的server暂时不参与负载
        #2.weight为weight越大，负载的权重就越大。
        #3.max\_fails：允许请求失败的次数默认为1.当超过最大次数时，返回proxy\_next\_upstream模块定义的错误
        #4.fail\_timeout:max\_fails次失败后，暂停的时间。
        #5.backup： 其它所有的非backup机器down或者忙的时候，请求backup机器。所以这台机器压力会最轻。

        #nginx支持同时设置多组的负载均衡，用来给不用的server来使用。
        #client\_body\_in\_file\_only设置为On 可以讲client post过来的数据记录到文件中用来做debug
        #client\_body\_temp\_path设置记录文件的目录 可以设置最多3层目录
        #location对URL进行匹配.可以进行重定向或者进行新的代理 负载均衡
    }
     
     
     
    #虚拟主机的配置
    server
    {
        #监听端口
        listen 80;

        #域名可以有多个，用空格隔开
        server\_name www.w3cschool.cn w3cschool.cn;
        index index.html index.htm index.php;
        root /data/www/w3cschool;

        #对\*\*\*\*\*\*进行负载均衡
        location ~ .\*.(php|php5)?$
        {
            fastcgi\_pass 127.0.0.1:9000;
            fastcgi\_index index.php;
            include fastcgi.conf;
        }
         
        #图片缓存时间设置
        location ~ .\*.(gif|jpg|jpeg|png|bmp|swf)$
        {
            expires 10d;
        }
         
        #JS和CSS缓存时间设置
        location ~ .\*.(js|css)?$
        {
            expires 1h;
        }
         
        #日志格式设定
        #$remote\_addr与$http\_x\_forwarded\_for用以记录客户端的ip地址；
        #$remote\_user：用来记录客户端用户名称；
        #$time\_local： 用来记录访问时间与时区；
        #$request： 用来记录请求的url与http协议；
        #$status： 用来记录请求状态；成功是200，
        #$body\_bytes\_sent ：记录发送给客户端文件主体内容大小；
        #$http\_referer：用来记录从那个页面链接访问过来的；
        #$http\_user\_agent：记录客户浏览器的相关信息；
        #通常web服务器放在反向代理的后面，这样就不能获取到客户的IP地址了，通过$remote\_add拿到的IP地址是反向代理服务器的iP地址。反向代理服务器在转发请求的http头信息中，可以增加x\_forwarded\_for信息，用以记录原有客户端的IP地址和原来客户端的请求的服务器地址。
        log\_format access '$remote\_addr - $remote\_user \[$time\_local\] "$request" '
        '$status $body\_bytes\_sent "$http\_referer" '
        '"$http\_user\_agent" $http\_x\_forwarded\_for';
         
        #定义本虚拟主机的访问日志
        access\_log  /usr/local/nginx/logs/host.access.log  main;
        access\_log  /usr/local/nginx/logs/host.access.404.log  log404;
         
        #对 "/" 启用反向代理
        location / {
            proxy\_pass http://127.0.0.1:88;
            proxy\_redirect off;
            proxy\_set\_header X-Real-IP $remote\_addr;
             
            #后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
            proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for;
             
            #以下是一些反向代理的配置，可选。
            proxy\_set\_header Host $host;

            #允许客户端请求的最大单文件字节数
            client\_max\_body\_size 10m;

            #缓冲区代理缓冲用户端请求的最大字节数，
            #如果把它设置为比较大的数值，例如256k，那么，无论使用firefox还是IE浏览器，来提交任意小于256k的图片，都很正常。如果注释该指令，使用默认的client\_body\_buffer\_size设置，也就是操作系统页面大小的两倍，8k或者16k，问题就出现了。
            #无论使用firefox4.0还是IE8.0，提交一个比较大，200k左右的图片，都返回500 Internal Server Error错误
            client\_body\_buffer\_size 128k;

            #表示使nginx阻止HTTP应答代码为400或者更高的应答。
            proxy\_intercept\_errors on;

            #后端服务器连接的超时时间\_发起握手等候响应超时时间
            #nginx跟后端服务器连接超时时间(代理连接超时)
            proxy\_connect\_timeout 90;

            #后端服务器数据回传时间(代理发送超时)
            #后端服务器数据回传时间\_就是在规定时间之内后端服务器必须传完所有的数据
            proxy\_send\_timeout 90;

            #连接成功后，后端服务器响应时间(代理接收超时)
            #连接成功后\_等候后端服务器响应时间\_其实已经进入后端的排队之中等候处理（也可以说是后端服务器处理请求的时间）
            proxy\_read\_timeout 90;

            #设置代理服务器（nginx）保存用户头信息的缓冲区大小
            #设置从被代理服务器读取的第一部分应答的缓冲区大小，通常情况下这部分应答中包含一个小的应答头，默认情况下这个值的大小为指令proxy\_buffers中指定的一个缓冲区的大小，不过可以将其设置为更小
            proxy\_buffer\_size 4k;

            #proxy\_buffers缓冲区，网页平均在32k以下的设置
            #设置用于读取应答（来自被代理服务器）的缓冲区数目和大小，默认情况也为分页大小，根据操作系统的不同可能是4k或者8k
            proxy\_buffers 4 32k;

            #高负荷下缓冲大小（proxy\_buffers\*2）
            proxy\_busy\_buffers\_size 64k;

            #设置在写入proxy\_temp\_path时数据的大小，预防一个工作进程在传递文件时阻塞太长
            #设定缓存文件夹大小，大于这个值，将从upstream服务器传
            proxy\_temp\_file\_write\_size 64k;
        }
         
         
        #设定查看Nginx状态的地址
        location /NginxStatus {
            stub\_status on;
            access\_log on;
            auth\_basic "NginxStatus";
            auth\_basic\_user\_file confpasswd;
            #htpasswd文件的内容可以用apache提供的htpasswd工具来产生。
        }
         
        #本地动静分离反向代理配置
        #所有jsp的页面均交由tomcat或resin处理
        location ~ .(jsp|jspx|do)?$ {
            proxy\_set\_header Host $host;
            proxy\_set\_header X-Real-IP $remote\_addr;
            proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for;
            proxy\_pass http://127.0.0.1:8080;
        }
         
        #所有静态文件由nginx直接读取不经过tomcat或resin
        location ~ .\*.(htm|html|gif|jpg|jpeg|png|bmp|swf|ioc|rar|zip|txt|flv|mid|doc|ppt|
        pdf|xls|mp3|wma)$
        {
            expires 15d; 
        }
         
        location ~ .\*.(js|css)?$
        {
            expires 1h;
        }
    }

}

![](https://common.cnblogs.com/images/copycode.gif)

## v 配置 nginx.conf

3.1 创建 springboot 工程，添加测试用的接口

![](https://common.cnblogs.com/images/copycode.gif)

/\*\* \* @author toutou
 \* @date by 2020/11
 \* @des \*/ @Slf4j
@RestController public class IndexController {@GetMapping("/index") public Result index(HttpServletRequest req) {
        String ip \\= ""; try {
            InetAddress localHost \\= InetAddress.getLocalHost(); if (localHost != null && localHost.getHostAddress() != null) {
                String\[] split \\= localHost.getHostAddress().split("\\\\."); if (split.length> 1) {
                    ip \\= split\[split.length - 1];
                }
            }
        } catch (Exception ipe) {
            log.warn("Caught Exception when get ip", ipe);
        }

        String message \= String.format("Server ip:%s; Server Port:%d;Local Port:%d", ip, req.getServerPort(), req.getLocalPort()); return Result.setSuccessResult(message);
    }

}

![](https://common.cnblogs.com/images/copycode.gif)

如果不会创建 springboot 工程的也没有关系，可以看看[SpringBoot 入门教程 (一) 详解 intellij idea 搭建 SpringBoot](https://www.cnblogs.com/toutou/p/9650939.html "SpringBoot 入门教程 (一) 详解 intellij idea 搭建 SpringBoot")，手把手教程。

3.2 如下图，先创建 6 个 docker springboot 实例

![](https://img2020.cnblogs.com/blog/506684/202011/506684-20201127170133083-96444643.png)

如果不会 docker 部署 springboot 的话，可以看看[详解 docker 部署 SpringBoot 及如何替换 jar 包](https://www.cnblogs.com/toutou/p/docker_springboot.html "详解 docker 部署 SpringBoot 及如何替换 jar 包")，手把手教程。

3.3 设置 nginx.conf 配置

![](https://common.cnblogs.com/images/copycode.gif)

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.

include /usr/share/nginx/modules/\*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user \[$time_local] "$request" ' '$status $body_bytes_sent "$http_referer"''"$http_user_agent" "$http_x_forwarded_for"';

    access\_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp\_nopush          on;
    tcp\_nodelay         on;
    keepalive\_timeout   65;
    types\_hash\_max\_size 2048;

    include             /etc/nginx/mime.types;
    default\_type        application/octet-stream;

    include /etc/nginx/conf.d/\*.conf;

    server {
        listen       80 default\_server;
        listen       \[::\]:80 default\_server;
        server\_name  \_;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/\*.conf;

        location / {
        }

        error\_page 404 /404.html;
        location = /404.html {
        }

        error\_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }

    upstream hellolearn1 {
        server 127.0.0.1:8301;
        server 127.0.0.1:8302;
        server 127.0.0.1:8303;
    }

    server {
        listen       8080;
        server\_name  www.toutou.com;

        location / {
            proxy\_pass http://hellolearn1;
            proxy\_set\_header Host $host;
            proxy\_set\_header X-Real-IP $remote\_addr;
            proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for;
        }
    }

}

![](https://common.cnblogs.com/images/copycode.gif)

主要是在原有的 nginx.conf 配置文件中追加 upstream 块和 server 块。

客户端 host 配置

ip www.toutou.com
ip www.nginxconfig.com
ip api.nginxconfig.com

3.4 测试效果

![](https://img2020.cnblogs.com/blog/506684/202011/506684-20201127170315996-1341187933.png)

3 次请求分别命中了 3 个 docker 实例，这是因为上面配置文件 upstream 块中没有配置 nginx 负载均衡的分配方式，默认是轮询方式。

3.5 nginx 负载分配方式：

3.5.1 轮询

默认方式，每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器 down 掉，能自动剔除。虽然这种方式简便、成本低廉。但缺点是：可靠性低和负载分配不均衡。适用于图片服务器集群和纯静态页面服务器集群。

3.5.2 weight(权重分配)

指定轮询几率，权重越高，在被访问的概率越大，用于后端服务器性能不均的情况。

例子：

upstream hellolearn{ 
    server 127.0.0.1:8001 weight=1; 
    server 127.0.0.1:8002 weight=2; 
    server 127.0.0.1:8003 weight=3;
}

当 Nginx 每收到 6 个请求，会把其中的 1 个转发给 8001，把其中的 2 个转发给 8002，把其中的 3 个转发给 8003。

3.5.3 ip_hash

每个请求按访问 ip 的 hash 结果分配，这样每个访客固定访问一个后端服务器，可以解决 session 的问题。

例子：

upstream hellolearn{ 
    ip_hash; 
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
}

3.5.4 fair（第三方）

按后端服务器的响应时间分配，响应时间短的优先分配，依赖第三方插件 nginx-upstream-fair，需要先安装。

例子：

upstream hellolearn{ 
    fair; 
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
}

3.5.5 url_hash（第三方）

按访问 url 的 hash 结果来分配请求，使每个 url 定向到同一个后端服务器，后端服务器为缓存时比较有效。在 upstream 中加入 hash 语句，hash_method 是使用的 hash 算法。

例子：

![](https://common.cnblogs.com/images/copycode.gif)

upstream hellolearn{ 
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
    hash $request_uri; 
    hash_method crc32; 
}

![](https://common.cnblogs.com/images/copycode.gif)

nginx 1.7.2 版本后，已经集成了 url hash 功能，可直接使用，不要安装三方模块。 该信息来自网络，未亲测，感兴趣的朋友可以试试。

3.6 负载均衡调度中的状态

![](https://img2020.cnblogs.com/blog/506684/202011/506684-20201127170935152-1286820953.png)

例子：

upstream hellolearn {
    server 127.0.0.1:8301 down;
    server 127.0.0.1:8302 backup;
    server 127.0.0.1:8303 max_fails=3 fail_timeout=30s;
}

注意：若访问 nginx 提示 502 的话，可以查看一下日志 `less /var/log/nginx/error.log` 。若发现提示 Permission denied，输入命令 `setsebool -P httpd_can_network_connect 1` 即可解决。

## v 进阶配置

http 块中可以有多个 upstream 块和 server 块，那么我们就来配置多个 upstream 块和 server 块看看什么效果。

4.1 nginx.conf 中追加 upstream 块和 server

![](https://common.cnblogs.com/images/copycode.gif)

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/\*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user \[$time_local] "$request" ' '$status $body_bytes_sent "$http_referer"''"$http_user_agent" "$http_x_forwarded_for"';

    access\_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp\_nopush          on;
    tcp\_nodelay         on;
    keepalive\_timeout   65;
    types\_hash\_max\_size 2048;

    include             /etc/nginx/mime.types;
    default\_type        application/octet-stream;

    include /etc/nginx/conf.d/\*.conf;

    server {
        listen       80 default\_server;
        listen       \[::\]:80 default\_server;
        server\_name  \_;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/\*.conf;

        location / {
        }

        error\_page 404 /404.html;
        location = /404.html {
        }

        error\_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }

    upstream hellolearn1 {
        server 127.0.0.1:8301;
        server 127.0.0.1:8302;
        server 127.0.0.1:8303;
    }

    server {
        listen       8080;
        server\_name  www.toutou.com;

        location / {
            proxy\_pass http://hellolearn1;
            proxy\_set\_header Host $host;
            proxy\_set\_header X-Real-IP $remote\_addr;
            proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for;
        }
    }

    upstream hellolearn2 {
        server 127.0.0.1:8304;
        server 127.0.0.1:8305;
        server 127.0.0.1:8306;
    }

    server {
        listen       8090;
        server\_name  www.nginxconfig.com;

        #负载到upstream hellolearn2
        location / {
            proxy\_pass http://hellolearn2;
            proxy\_set\_header Host $host;
            proxy\_set\_header X-Real-IP $remote\_addr;
            proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for;
        }
        
        #匹配以/images/开头的任何查询并停止搜索，表示匹配URL时忽略字母大小写问题
        location ^~ /sayhello/ {
            default\_type 'text/plain';
            return 200 "say hello.";
        }
    }

    server {
        listen       8099;
        server\_name  ~.\*nginxconfig.com;

        #可以匹配所有请求
        location / {
            default\_type "text/html;charset=utf-8";
            return 200 "<p style\='color:red;'\>我可以匹配所有请求</p\>";
        }
        
        #只有当用户请求是/时，才会使用该location下的配置
        location = / {
            default\_type "text/html;charset=utf-8";
            return 404 "<p style\='color:blue;font-weight: bold;'\>只有根目录才能发现我</p\>";
        }
        
        #匹配任何以.red、.blue或者.black结尾的请求。
        location ~\* \\.(red|blue|black)$ {
            default\_type "text/html;charset=utf-8";
            return 200 "<p\>red or blue or black</p\>";
        }
    }

}

![](https://common.cnblogs.com/images/copycode.gif)

4.2 运行效果图

![](https://img2020.cnblogs.com/blog/506684/202011/506684-20201130143613030-605898951.png)

在 nginx.conf 末尾处中追加了一组 upstream 块和两组 server 块，其中两组 server 块共计包含 5 组 location，上图中的 5 个演示效果，正是对应上末尾追加的 5 个 location 块的匹配规则的。

4.3 server_name 的作用

server name 为虚拟服务器的识别路径。因此不同的域名会通过请求头中的 HOST 字段，匹配到特定的 server 块，转发到对应的应用服务器中去。

server_name 与 Host 的匹配优先级：

首先选择所有字符串完全匹配的 server_name。如：www.toutou.com

其次选择通配符在前面的 server_name。如：\*.toutou.com

其次选择通配符在后面的 server_name。如：www.toutou.\*

最后选择使用正在表达式才匹配的 server_name。如：~^\\.toutou\\.com$

如果上面这些都不匹配

1、优先选择 listen 配置项后有 default 或 default_server 的

2、找到匹配 listen 端口的第一个 server 块

4.4 location 匹配规则

location 后面的这些 `=` 或者 `~` 具体是干嘛用的呢，分别介绍一下。

语法： `location [=|~|~*|^~] /uri/ { … }`

4.4.1 `=` 精确匹配路径，用于不含正则表达式的 uri 前，如果匹配成功，不再进行后续的查找。

4.4.2 `^~` 用于不含正则表达式的 uri； 前，表示如果该符号后面的字符是最佳匹配，采用该规则，不再进行后续的查找。

4.4.3 `~` 表示用该符号后面的正则去匹配路径，区分大小写。

4.4.4 `~*` 表示用该符号后面的正则去匹配路径，不区分大小写。跟 ~ 优先级都比较低，如有多个 location 的正则能匹配的话，则使用正则表达式最长的那个。

## v 博客总结

> Nginx 能成为最受欢迎的 web 服务器之一，功能十分强大，还有很多功能值我们去学习。

其他参考 / 学习资料：

-   [Configuration Guide](https://docs.nginx.com/nginx-app-protect/configuration/ "Nginx 负载均衡配置")
-   [Nginx 从入门到实践，万字详解！](https://juejin.cn/post/6844904144235413512 "Nginx 负载均衡配置")
-   [Understanding the Nginx Configuration File Structure and Configuration Contexts](https://www.digitalocean.com/community/tutorials/understanding-the-nginx-configuration-file-structure-and-configuration-contexts "Nginx 负载均衡配置")
-   [Nginx location priority](https://stackoverflow.com/questions/5238377/nginx-location-priority "Nginx 负载均衡配置")
-   [nginx.conf 文件内容详解](https://www.cnblogs.com/paulwhw/articles/11116363.html "Nginx 负载均衡配置")
-   [Nginx —— nginx 服务的基本配置（nginx.conf 文件的详解）](https://blog.csdn.net/weixin_42167759/article/details/85049546 "Nginx 负载均衡配置")

## v 源码地址

[https://github.com/toutouge/javademosecond/tree/master/hellolearn](https://github.com/toutouge/javademosecond/tree/master/hellolearn "Nginx 负载均衡配置")

作　　者：**[请叫我头头哥](http://www.cnblogs.com/toutou/)**  
出　　处：[http://www.cnblogs.com/toutou/](http://www.cnblogs.com/toutou/)  
关于作者：专注于基础平台的项目开发。如有问题或建议，请多多赐教！  
版权声明：本文版权归作者和博客园共有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文链接。  
特此声明：所有评论和私信都会在第一时间回复。也欢迎园子的大大们指正错误，共同进步。或者[直接私信](http://msg.cnblogs.com/msg/send/%E8%AF%B7%E5%8F%AB%E6%88%91%E5%A4%B4%E5%A4%B4%E5%93%A5)我  
声援博主：如果您觉得文章对您有帮助，可以点击文章右下角**【推荐】** 一下。您的鼓励是作者坚持原创和持续写作的最大动力！ 
 [https://www.cnblogs.com/toutou/p/nginx_conf.html](https://www.cnblogs.com/toutou/p/nginx_conf.html)
