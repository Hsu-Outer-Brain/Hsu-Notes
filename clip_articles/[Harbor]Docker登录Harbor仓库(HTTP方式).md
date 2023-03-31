# [Harbor]Docker登录Harbor仓库(HTTP方式)
[\[Harbor\]Docker 登录 Harbor 仓库 (HTTP 方式)](https://www.shuzhiduo.com/A/QW5Y4a8OJm/) 

  [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-31%2014-37-00/2077eb51-32d8-45aa-8c0d-e4e28c15b2a7.png?raw=true)
](https://www.niugongju.com/chatgpt-acount.html) 

Docker 登录到 Harbor 仓库时, 不管是使用 http 协议还是使用 https 协议, 都需要修改一些配置.

这篇文章来介绍一下, 在使用 http 协议时, 需要进行什么哪些配置.

首先, 确定自己的 Harbor 仓库使用的是 http 协议, 在 harbor.cfg 文件中就可以看到:

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-31%2014-37-00/6aecbbcd-d733-49b5-8e31-3a5d63a06f70.jpeg?raw=true)

查找 docker 的服务文件, 使用命令:

```


1.  `systemctl status docker  
    `


```

可以看到 docker 的服务文件在 / etc/systemd/system 目录下.

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-31%2014-37-00/3bcbd163-f5f9-4bbe-b57e-b51165a7f958.jpeg?raw=true)

接下来我们需要去编辑 docker.service 文件, 并进行一些修改, 在 ExecStart 处, 添加–insecure-registry 参数

```


1.  `--insecure-registry=reg.zll.com(Harbor地址,harbor.cfg文件中的hostname项)  
    `


```

修改完成如下图:

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-31%2014-37-00/3692e0b5-ae46-48cb-8286-5b2875377f96.jpeg?raw=true)

重新加载 service 文件, 重启 docker 服务:

```


1.  `systemctl daemon-reload  
    `
2.  `systemctl restart docker  
    `


```

在图中可以看到, Harbor 仓库我是使用的域名, 所以还需要在 hosts 文件中做一些配置, 如果使用的是 ip 地址, 则此步骤可以忽略

```


1.  `编辑hosts文件:vi /etc/hosts  
    `
2.  `将Harbor地址写入到hosts文件中:192.168.243.138 reg.zll.com  
    `
3.  `#以我这次的配置为例,具体可以灵活变动  
    `


```

此时, 相关步骤便结束了, 我们可以在 Docker 客户端使用命令进行登录

```


1.  `docker login [ip地址或域名](Harbor地址,harbor.cfg文件中的hostname项)  
    `
2.  `//根据提示分别输入用户名和密码  
    `


```

可以看到, 此时 Docker 可以登录到 Harbor 仓库上面了.

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-31%2014-37-00/acd3cbae-a99f-470a-8496-9ef75ed557b2.jpeg?raw=true)

因为使用的是 http 协议登陆的, 所以会有一个警告, 对于实验环境来说, 是可以忽略的.

可能遇到的问题: Error response from daemon: Get [http://reg.zll.com/v2/:](http://reg.zll.com/v2/:) dial tcp 192.168.243.138:80: connect: connection refused

原因是因为在修改了 hosts 文件之后, 没有重新载入 docker, 再运行一下命令即可:

```


1.  `systemctl daemon-reload  
    `
2.  `systemctl restart docker  
    `


```

关于 Docker 登录 Harbor 仓库 (HTTP 方式) 到此便结束了, 感谢您的阅读~

 [![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-31%2014-37-00/0b2889ca-0270-4043-a6d4-39f7173112ce.png?raw=true)
](https://www.niugongju.com/chatgpt-acount.html) 

## [\[Harbor\]Docker 登录 Harbor 仓库 (HTTP 方式) 的更多相关文章](https://www.shuzhiduo.com/R/QW5Y4a8OJm/)

1.  [企业级 Docker Registry —— Harbor 搭建和使用](https://www.shuzhiduo.com/A/o75NM07DJW/)

    本节内容: Harbor 介绍 安装部署 Harbor 环境要求 环境信息 安装部署 harbor 配置 harbor 配置存储 完成安装和启动 harbor 访问 Harbor 修改管理员密码 启动后相关容器 ...
2.  [用其他主机 docker login 登录 Harbor 仓库报错](https://www.shuzhiduo.com/A/gGdXmmlGz4/)

    做微服务的时候, 我准备把编译好的 jar 包, 部署到我的 Harbor 仓库上, 却登录不上去, 出现以下报错: \[root@k8s-master ~]# docker login 192.168.30.24Us ...
3.  [Docker: 企业级镜像仓库 Harbor 的使用](https://www.shuzhiduo.com/A/WpdK472rzV/)

    上一节, 演示了 Harbor 的安装部署 这次我们来讲解 Harbor 的使用. 我们需要了解到: 1. 如何推镜像到镜像仓库 2. 如何从镜像仓库拉取镜像 3. 如何运行从私有仓库拉取的镜像 # 查看 h ...
4.  [docker 企业级镜像仓库 Harbor 管理](https://www.shuzhiduo.com/A/RnJWyP7vdq/)

    Harbor 概述 Harbor 是由 VMWare 公司开源的容器镜像仓库. 事实上, Harbor 是在 Docker Registry 上进行了相应的企业级扩展, 从而获得了更加广泛的应用, 这些新的企业级特性包括: ...
5.  [docker 的企业级仓库 - harbor](https://www.shuzhiduo.com/A/kjdwarDAJN/)

    Harbor 一. 背景 Docker 中要使用镜像, 我们一般都会从本地. Docker Hub 公共仓库或者其它第三方的公共仓库中下载镜像, 但是出于安全和一些内外网的原因考虑, 企业级上不会轻易使用. 普通的 D ...
6.  [Docker 私有镜像仓库 Harbor](https://www.shuzhiduo.com/A/A2dml1QO5e/)

    一. 安装 Harbor(离线安装包的方式安装) 1. 解压离线包 2. 进入 harbor 目录中编辑 harbor.yml 3. 安装 docker-compose yum -y install docker-co ...
7.  [Docker 企业级镜像仓库 Harbor 的搭建与维护](https://www.shuzhiduo.com/A/amd0GkZ15g/)

    目录 一. 什么是 Harbor 二. Harbor 安装 2.1.Harbor 安装环境 2.2.Harbor 安装 2.3 配置 HTTPS 三. Harbor 的使用 3.1. 登录 Harbor 并使用 3. ...
8.  [docker(三)：Harbor 1.8.0 仓库的安装和使用](https://www.shuzhiduo.com/A/q4zVADnKdK/)

    回顾: docker(一):docker 是什么? docker(二):CentOS 安装 docker docker(部署常见应用):docker 部署 mysql 安装的先决条件 硬件环境 1.CPU    ...
9.  [菜鸟系列 docker——搭建私有仓库 harbor(6)](https://www.shuzhiduo.com/A/lk5ajPQad1/)

    docker 搭建私有仓库 harbor 1. 准备条件 安装 docker sudo yum update sudo yum install -y yum-utils device-mapper-per ...

## 随机推荐

1.  [lucene 之中文分词及其高亮显示 (五)](https://www.shuzhiduo.com/A/WpdKwlpXdV/)

    中文分词: 即换个分词器 Analyzer analyzer = new StandardAnalyzer();// 标准分词器     换成  SmartChineseAnalyzer analyze ...
2.  [python 自动化开发 -\[第二天\]- 基础数据类型与编码 (续)](https://www.shuzhiduo.com/A/gVdnPpO8JW/)

    今日简介: - 编码 - 进制转换 - 初识对象 - 基本的数据类型 - 整数 - 布尔值 - 字符串 - 列表 - 元祖 - 字典 - 集合 - range/enumcate 一. 编码 encode ...
3.  [sklearn - 数据预处理 scale](https://www.shuzhiduo.com/A/QV5ZDPr2zy/)

    sklearn 实战 - 乳腺癌细胞数据挖掘 (博主亲自录制视频) [https://study.163.com/course/introduction.htm?courseId=1005269003&](https://study.163.com/course/introduction.htm?courseId=1005269003&) ...
4.  [mybatis\_异常](https://www.shuzhiduo.com/A/WpdKwWq1dV/)

    1.HTTP Status 500 - Request processing failed; nested exception is org.apache.ibatis.binding.Binding ...
5.  [flask flash 消息](https://www.shuzhiduo.com/A/LPdoqvrBJ3/)

    请求完成, 让用户知道状态发生了变化, 可以使用 flash 确认消息 示例: xx.py from flask import Flask,render_template,request,redirect,u ...
6.  [python 变量 if](https://www.shuzhiduo.com/A/1O5EOZRrz7/)

    \######################### 总结 ###################### 1. 初识 python python 是一门弱类型的解释型高级编程语言 解释器: CPython 官方 ...
7.  [python 中\\r 的意义及用法](https://www.shuzhiduo.com/A/B0zqQrbQzv/)

    \\r 的意义 \\r 表示将光标的位置回退到本行的开头位置 \\b 表示将光标的位置回退一位 在 python 里 print 会默认进行换行, 可以通过修改参数让其不换行 (1) python2 中可以在 print 语句 ...
8.  [Git 配置信息相关命令](https://www.shuzhiduo.com/A/A2dm9llgde/)

    查看 git 所有配置项 $ git config -l or $ git config --list 全局配置用户名邮箱 $ git config --global user.name "yo ...
9.  [IBM X3650 M5 服务器 RAID 阵列设置](https://www.shuzhiduo.com/A/qVde4oKQ5P/)

    生产环境中的 raid 配置说明: 一. 开机后, 注意引导界面, 按 F1 键进入 BIOS 进行设置 二. 进入 BIOS 后, 选择 system setting--storage , 进入磁盘阵列配置界面, 可以看到 M5 ...
10. [day 9 - 1 函数](https://www.shuzhiduo.com/A/D854LAy65E/)

    函数 函数: 定义了之后, 可以在任何需要它的地方调用 函数模拟 len() 方法 #len() 方法的使用 s="世界上唯一不变的就是变化" print(len(s)) #下面是我们 ...
