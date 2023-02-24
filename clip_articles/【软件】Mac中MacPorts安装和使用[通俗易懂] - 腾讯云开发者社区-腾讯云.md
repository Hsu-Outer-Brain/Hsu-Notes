# 【软件】Mac中MacPorts安装和使用[通俗易懂] - 腾讯云开发者社区-腾讯云
[【软件】Mac 中 MacPorts 安装和使用 \[通俗易懂\] - 腾讯云开发者社区 - 腾讯云](https://cloud.tencent.com/developer/article/2097681) 

 大家好，又见面了，我是你们的朋友全栈君。

下载地址：[https://www.macports.org/install.php](https://www.macports.org/install.php) 选择自己的下载版本

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-24%2010-23-01/6b75c453-3486-4d33-b0e6-6391dcf1ab1e.png?raw=true)

步骤一：断开网络 步骤二：安装安装包

如果步骤一没断网成功会导致安装卡住，如果卡住了，需要强制退出软件 首先使用 option+command+esc 打开强制退出应用程序窗口，选择强制退出安装程序 然后执行 ps aux | grep install 找到 MacPorts 的安装程序，kill -9 直接删掉，最后再断开网络重新安装。

修改 目录：/opt/local/etc/macports/sources.conf 找到下面这条配置

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-24%2010-23-01/bae32549-479c-43d5-bff9-bd64d1ffe584.png?raw=true)

添加一条国内的镜像源 获取镜像源可通过这个网站查找：[https://trac.macports.org/wiki/Mirrors](https://trac.macports.org/wiki/Mirrors) 我现在用的就是下面这条

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-24%2010-23-01/8c820627-e3f4-4e65-8bfb-97288c32fa10.png?raw=true)

执行如下命令

添加如下环境变量

    export PATH=/opt/local/bin:/opt/local/sbin:$PATH

然后更新配置 source .bash_profile

执行 sudo port selfupdate 如果出现下面图片里面的内容，就表示安装成功了。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-24%2010-23-01/c8f84043-72d3-4210-8370-226d16ab6663.png?raw=true)

发布者：全栈程序员栈长，转载请注明出处：[https://javaforall.cn/155559.html 原文链接：https://javaforall.cn](https://javaforall.cn/155559.html原文链接：https://javaforall.cn)
