# Go后端开发学习路线
鸽了很久的 Go 后端路线，这个来了！书籍、网站、项目推荐全都有！！ 星球 Go 的录友们可以学起来。

## 1. Go 语言基础

入门看这个：《Go 语言学习笔记》、 《Go 语言趣学指南》 、《Head First Go》任选一本都可以，跟着书籍多敲敲代码，go 语言相对 C++，java 来说，简单很多

![](https://code-thinking-1253855093.file.myqcloud.com/pics/20211230114758.png)

### 网课资源

网课不是很推荐，因为 go 的更新很快，B 站上尚硅谷和七米的视频质量不错，但也是两年前的，可以作为不懂知识点的参考

推荐一门较新的网课，可搭配入门书来看：[https://www.bilibili.com/video/BV1s341147US](https://www.bilibili.com/video/BV1s341147US)

#### 刚入门 go 可能会遇到 module 管理的疑惑，具体请看

-   [忘掉 GOPATH，即刻出发！从零开始搭建 Go 开发环境](https://www.bilibili.com/video/BV1bV41177KD)
-   [弃用 Go Path, Go Moudle 入门与精通](https://mp.weixin.qq.com/s/6gJkSyGAFR0v6kow2uVklA)
-   [Go Moudle 的历史与坑](https://mp.weixin.qq.com/s/jpp7vs3Fdg4m15P1SHt1yA)

### 学习基础网站资源

1. go 语言中文网：[https://studygolang.com/articles](https://studygolang.com/articles)

2. Go By Example 中文版 : [https://gobyexample-cn.github.io](https://gobyexample-cn.github.io), 使用代码示例来学习 Go 语言。

3. Go 面试题 : [http://www.topgoer.cn/docs/gomianshiti/mianshiti](http://www.topgoer.cn/docs/gomianshiti/mianshiti), 也是使用代码示例来讲解 Go，用来准备面试也是很好的。

4. 跟着单元测试学习 Go : [https://github.com/quii/learn-go-with-tests](https://github.com/quii/learn-go-with-tests), 有中文版。

### go 小项目

学完基础，知道你可能想试试手，这里推荐几个初级的项目

1. starcharts : [https://github.com/caarlos0/starcharts](https://github.com/caarlos0/starcharts), 项目的功能是生成 Github 上面的项目的 star 趋势图，核心代码不多，用来练手很合适。

2. gorched : [https://github.com/zladovan/gorched](https://github.com/zladovan/gorched), 使用 Go 写的一个小游戏。

3. pacgo : [https://github.com/danicat/pacgo](https://github.com/danicat/pacgo), 也是使用 Go 写的一个小游戏，每一步都有详细的描述和代码实现。

4. wechat-go : [https://github.com/songtianyi/wechat-go](https://github.com/songtianyi/wechat-go), 微信 web 版 API 的 Go 实现，模拟微信网页版的登录／联系人／消息收发等功能。

#### 项目视频推荐：

-   [李文周老师：日志收集项目](https://www.bilibili.com/video/BV1Df4y1C7o5)

：可以练习到 goroutine，channel 和 context 的操作，适合练手，其中涉及到的一些中间件技术大概知道能干嘛用就行，不用纠结

-   [基于 golang 协程实现流量统计系统](https://www.bilibili.com/video/BV1YE411G7BU?p=1)

: 这是慕课网的一个视频，网友上传到 B 站上，录制有一些卡顿，介意的话可以从其他渠道找一下慕课的这个视频

## 2. Web 开发

基础知识掌握之后，可以上手做一些 web 应用，进一步了解更多的 Go 语言相关框架以及生产环境中的常用中间件，推荐书籍《Go Web 编程》。

![](https://code-thinking-1253855093.file.myqcloud.com/pics/20211230115821.png)

[Go Web 编程 在线版](https://www.bookstack.cn/read/Go-Web/README.md)

第一个图片的书是一位印度工程师写的，基于原生的 go 标准库实现了一个聊天室，暂无电子版

了解了 web 基础知识后可以学习下 web 框架 Gin、beego：两个框架都比较的流行，选择其中一个其实就可以了，推荐 Gin。官方文档都有中文，照着 demo 敲一下，了解下怎么处理 HTTP 请求的。

[geektutu : gin 简明教程](https://geektutu.com/post/quick-go-gin.html)

另外还有 gorm 也是掌握的 ：

[gorm 指南](https://gorm.io/zh_CN/docs/index.html)

基本上看官方文档就可以了，不用去找其他的书籍，没有比官方文章更正宗的资料了。

### go web 项目

推荐几个使用 Go 构建的基础 web 项目：

1. gin-vue-admin : [https://github.com/flipped-aurora/gin-vue-admin](https://github.com/flipped-aurora/gin-vue-admin), 使用 Gin 框架构建的后台管理系统。

2. ferry : [https://github.com/lanyulei/ferry](https://github.com/lanyulei/ferry), 基于 Gin + Vue + Element UI 前后端分离的工单系统。

3. go-admin : [https://github.com/go-admin-team/go-admin](https://github.com/go-admin-team/go-admin), Gin + Vue + Element UI 的前后端分离权限管理系统。

对于 web 项目的学习，可能有同学觉得项目太庞杂，根本不知道怎么下手。我想建议的是，可以在本地把项目跑起来，然后断点调试一个 HTTP 请求的整体流程，搞懂了一个接口，其他的大同小异。

## 3. Go 语言进阶

基础知识拿捏后需要了解一下 go 的底层原理和高级用法啦（面试造火箭必备）

这里推荐书籍《Go 程序设计语言》（号称 Go 圣经）、《Go 专家编程》、《Go 语言高级编程》。

![](https://code-thinking-1253855093.file.myqcloud.com/pics/20211230115627.png)

### 进阶网站学习资源

1.《Go 语言圣经》: [https://books.studygolang.com/gopl-zh/](https://books.studygolang.com/gopl-zh/) . go 圣经是值得反复阅读的

2.《Go 语言高级编程》: [https://chai2010.cn/advanced-go-programming-book](https://chai2010.cn/advanced-go-programming-book) . 曹大的书，深入到了 go 的汇编和很多高级用法

3.《Go 专家编程》 : [https://books.studygolang.com/GoExpertProgramming/](https://books.studygolang.com/GoExpertProgramming/). 理解 go 中常见数据结构的底层，主要看 Channel，Mutex，Map

4.《Go 语言设计与实现》：[https://draveness.me/golang/](https://draveness.me/golang/) 了解 go 的编译机制，goroutine 调度机制，垃圾回收机制 (这本书写得很好，必须拿捏)

上面的撸完了，推荐看下极客时间郝林老师的专栏《Go 36 讲》，鸟叔的《Go 并发编程》继续精进

### 进阶 go 项目

想要进一步巩固所学知识，这里推荐几个比较进阶的项目

1. gochat : [https://github.com/LockGit/gochat](https://github.com/LockGit/gochat), 一个 Go 语言实现的轻量级 im 系统，对网络方面熟悉或者感兴趣的可以看看。

2. 7DaysGolang : [https://github.com/geektutu/7days-golang](https://github.com/geektutu/7days-golang), 7 天使用 Go 从零实现 web 框架、分布式缓存、ORM 框架、RPC 框架，代码量不多，但是质量挺不错

3. godis : [https://github.com/hdt3213/godis/blob/master/README\\\_CN.md](https://github.com/hdt3213/godis/blob/master/README\_CN.md), 用 golang 实现一个 redis 服务器

## 4. go 微服务

（可以选择性掌握）

目前 Go 在微服务中的应用也比较广泛，但说实话，微服务是一个太庞大的话题，你不可能把每一个核心的问题都能够搞清楚，而且也没条件，或许只能在公司的具体的微服务生产环境中，才能够对相关的概念有更加深刻的体会。

推荐一本微服务概述的基础书籍《微服务设计》、《微服务架构设计模式》，可以帮助你理解微服务的建模、集成、测试、部署和监控的一些基础知识。

![](https://code-thinking-1253855093.file.myqcloud.com/pics/20211230120046.png)

推荐 Go 语言的微服务框架 GoKit、GoMicro、go-zero、kratos，可以随便选择一个，理解其基本的用法、设计等等。其中 go-zero 和 kratos 是国内开源的，因此都有比较详细的中文文档。

建议学习 kratos，跟着快速上手和简单例子写一遍，不仅可以熟悉 CRUD，还能了解 Go 中优秀的框架，工程化

[Kratos](https://github.com/go-kratos/kratos)

这里推荐一个在线学习的资料：

### 微服务网站学习资源

[https://ewanvalentine.io/microservices-in-golang-part-1](https://ewanvalentine.io/microservices-in-golang-part-1)

手把手实现一个简单的 Go 微服务项目，你可以通过这个项目来学习微服务的相关知识，并且有中文版。

### 工作进阶

如果上面的都掌握的不错了，可以看极客时间毛剑的《go 进阶训练营》，注意一定要上面的都掌握了，不然理解不了毛剑老师的核心思路，看过就忘

曹大的《go 高级实战训练营》口碑也很好，相较于毛剑老师更贴近于程序优化，代码优化，底层逻辑

## 5. 计算机基础

关于计算基础（操作系统，数据库，网络）

[看这里](https://t.zsxq.com/vFAIqFy)

## 6. 算法

[看这里](https://t.zsxq.com/E6AEMRn)

推荐一个 go 语言实现的 leetcode 题解

[Cookbook](https://books.halfrost.com/leetcode/) 可以学习到很多 go 的函数调用和高级语法

## 7. 设计模式

设计模式的思想是相通的，但不要用面向对象语言中的写法来生套到 golang 中

推荐学习：

[Go 设计模式](https://lailin.xyz/post/singleton.html)

## 8. 优质 gopher 博客推荐：

-   [极客兔兔](https://geektutu.com/) 七天实现系列，面试题都很棒
-   [Go 语言充电站](https://lessisbetter.site/) go 的内存相关知识讲得很透彻
-   [Image's Blog](https://imageslr.com/)
-   [面向信仰编程](https://draveness.me/) go 语言设计与实现作者，`为什么这样设计`系列必看
-   [Mind Hacks](https://mindhacks.cn/) 不止于技术，分享了很多思维层面的知识
-   [煎鱼](https://eddycjy.com/) go 相关知识基本都有涉及

## 9. 找份实习需要学到什么程度？

```
对于学生来说，最重要的莫过于计算机基础和一个可以放简历上的项目，这里给出一个简明的路线，可根据个人情况进行扩展：     

```

-   **Go 语言基础** : 了解 golang 的`基本语法`和`语言特性`
-   **算法题** ：用 golang 来练习写算法题，熟练掌握 golang 的`基本语法`和`常用库函数`，同时刷好算法题也是进入中大厂的必要条件
-   **web 相关**：`《Go Web 编程》`看了，深入理解 golang 如何处理 request，标准库中的包如何应对各种`常见的 web 场景`； 紧接着看下 gin，先看极客兔兔的`gin 简明教程`，然后上手跟着极客兔兔的博客写 web 框架，写的过程要去看`gin 的官方文档`解决疑惑；该项目可以放简历上
-   **基础知识**：`网络`，`操作系统`，`数据库`，面试的重中之重，可以开始看，理解着背了
-   **Go 底层原理**：面试造火箭必备，具体看上面 Go 语言进阶部分。面试重点：`map`，`slice`，`channel`底层实现原理，`goroutine 调度`，`GMP`模型，`内存分配`，`垃圾回收`
-   **web 项目** ：选择一个 web 项目作为自己的项目（见上面 web 项目推荐），项目需要用到`数据库`，和至少一种`中间件`（如 redis，消息队列等）； 如果觉得难度大，可以找相关视频教程，不限于 B 站，某课等等

* * *

上面的学完就可以出去面试了，以下用于加强（根据自己方向选择）：

-   更多的中间件
-   微服务相关
-   docker, k8s 相关 
    [https://articles.zsxq.com/id_4dp0wr59orse.html](https://articles.zsxq.com/id_4dp0wr59orse.html)
