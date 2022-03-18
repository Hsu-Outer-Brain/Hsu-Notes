# CloudCanal测试记录_lhw - 技术部 - 真融宝WIKI
## 方式一：基于 macos 的本地测试

安装本地环境的 cloudcanal 需要有 docker 的支持，所以需要首先配置 docker

docker 下载及文档地址：[https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/)

正常安装即可

打开软件：

![](http://wiki.zhenrongbao.com/download/attachments/46270653/image2022-2-22%2017%3A5%3A5.png?version=1&modificationDate=1645523098000&api=v2)

根据自己电脑配置进行修改即可

理论上我们在 linux 上需要额外安装 docker compose  连接地址：[https://docs.docker.com/compose/install/#alternative-install-options](https://docs.docker.com/compose/install/#alternative-install-options)

但是在 macos 和 windows 上桌面应用已经集成好了，所以不需要额外安装

下载 cloudcanal 地址：[https://cloudcanal-community.oss-cn-hangzhou.aliyuncs.com/latest/cloudcanal.7z](https://cloudcanal-community.oss-cn-hangzhou.aliyuncs.com/latest/cloudcanal.7z)

这是个 7z 压缩文件 我们需要手动解压 需要 7z 插件 我们在终端安装 homebrew 

直接在终端输入：/bin/bash -c "$(curl -fsSL [https://raw.githubusercontent.com/Homebrew/install/master/install.sh](https://raw.githubusercontent.com/Homebrew/install/master/install.sh))"

即可实现镜像安装，速度较快

安装完 homebrew 再安装 7z 工具 

执行 brew install p7zip

安装完成后 进入 cloudcanal 的下载路径，建立一个想要解压的目录（解压完了直接散开了，没有文件夹所以需要自己建立一个，再把压缩包移动过去）

进入建立好的文件夹执行 7z e cloudcanal.7z  解压到当前目录 

启动 cloudcanal 前要记得建立好 docker：

![](http://wiki.zhenrongbao.com/download/attachments/46270653/image2022-2-22%2017%3A17%3A59.png?version=1&modificationDate=1645523098000&api=v2)

`复制图中的连接粘贴到终端中，等待执行成功回到 docker 页面（变成这样）`

![](http://wiki.zhenrongbao.com/download/attachments/46270653/image2022-2-22%2017%3A19%3A33.png?version=1&modificationDate=1645523098000&api=v2)

这里解释一下：因为我们在本地配置，那么我们的 macos 就相当于我们 docker 的宿主机，cloudcanal 运行于 docker 中

接下来我们就可以启动 cloudcanal 啦

终端来到安装 cloudcanal 的目录，sh startup.sh

等待执行完成，出现

![](https://www.askcug.com/assets/uploads/files/1620979032983-7b00e562-cd45-4905-a626-1356503d8213-image-resized.png)

本机浏览器：localhost:8111 进入页面 输入预先配置的账号密码

用户名：test@clougence.com   密码：clougence2021

默认添加的测试 MySQL 数据库 cloudcanal_test_a(源端) 和 cloudcanal_test_b(目标端) 这两个库中已准备用于测试的表和数据默认已添加了一台运行机器，

用于执行具体的数据同步任务如遇到需要发送短信的场景，先点击获取验证码，然后输入短信验证码 777777 即可

来到这个界面 ：

![](http://wiki.zhenrongbao.com/download/attachments/46270653/image2022-2-22%2017%3A29%3A49.png?version=1&modificationDate=1645523098000&api=v2)

新建任务

![](http://wiki.zhenrongbao.com/download/attachments/46270653/image2022-2-22%2017%3A31%3A48.png?version=1&modificationDate=1645523098000&api=v2)

如图所示配置，下一步

![](http://wiki.zhenrongbao.com/download/attachments/46270653/image2022-2-22%2017%3A32%3A43.png?version=1&modificationDate=1645523098000&api=v2)

根据 docker 配置来，

![](http://wiki.zhenrongbao.com/download/attachments/46270653/image2022-2-22%2017%3A35%3A16.png?version=1&modificationDate=1645523098000&api=v2)

全选，下一步，无脑下一步到创建成功

![](http://wiki.zhenrongbao.com/download/attachments/46270653/image2022-2-22%2017%3A36%3A49.png?version=1&modificationDate=1645523098000&api=v2)

在终端输入 mysql -uclougence -h127.0.0.1 -P25000 -p123456 进入 cloudcanal 内置 mysql

在里面进行测试即可

我们是使用测试 mysql 的 cloudcanal_test_a 库 同步到 cloudcanal_test_b 库 我们可以测试新增表，新增字段，修改字段，已经对表的 ddl 操作

在新增表的时候，我们需要去

![](http://wiki.zhenrongbao.com/download/attachments/46270653/image2022-2-22%2017%3A40%3A57.png?version=1&modificationDate=1645523098000&api=v2)

![](http://wiki.zhenrongbao.com/download/attachments/46270653/image2022-2-22%2017%3A41%3A41.png?version=1&modificationDate=1645523098000&api=v2)

修改订阅，将新插入的表勾选上，无脑下一步，等待他重启过后就好了。

## 方式二：服务器测试

![](http://wiki.zhenrongbao.com/download/attachments/46270653/image2022-2-22%2017%3A43%3A50.png?version=1&modificationDate=1645523098000&api=v2)

这台机器上已经全部部署，使用他的 ip 加端口号 即可访问 cloudcanal 操作和上述一致 
 [http://wiki.zhenrongbao.com/pages/viewpage.action?pageId=46270653](http://wiki.zhenrongbao.com/pages/viewpage.action?pageId=46270653)
