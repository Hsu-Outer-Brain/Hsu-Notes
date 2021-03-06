# (1条消息) 实现一个域名对应多个IP地址和DNS优缺点详解！_魏波-CSDN博主-CSDN博客_一个域名绑定多个ip
### 1.DNS 定义：

DNS（Domain Name System）是因特网的一项服务，它作为域名和 IP 地址相互映射的一个分布式数据库，能够使人更方便的访问互联网。

### 2.DNS 作用：

（1）解析域名

人们在通过浏览器访问网站时只需要记住网站的域名即可，而不需要记住那些不太容易理解的 IP 地址。在 DNS 系统中有一个比较重要的的资源类型叫做主机记录也称为 A 记录，A 记录是用于名称解析的重要记录，它将特定的主机名映射到对应主机的 IP 地址上。如果你有一个自己的域名，那么要想别人能访问到你的网站，你需要到特定的 DNS 解析服务商的服务器上填写 A 记录，过一段时间后，别人就能通过你的域名访问你的网站了。

（2）负载均衡

DNS 除了能解析域名之外还具有负载均衡的功能，下面是利用 DNS 工作原理处理负载均衡的工作原理图：

![](https://img-blog.csdn.net/20181005190122337?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaWJvMTIzMDEyMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

由上图可以看出，在 DNS 服务器中应该配置了多个 A 记录，如：

````null
www.apusapp.com IN A 114.100.20.201;www.apusapp.com IN A 114.100.20.202;www.apusapp.com IN A 114.100.20.203;```

  
因此，每次域名解析请求都会根据对应的负载均衡算法计算出一个不同的IP地址并返回，这样A记录中配置多个服务器就可以构成一个集群，并可以实现负载均衡。上图中，用户请求www.apusapp.com，DNS根据A记录和负载均衡算法计算得到一个IP地址114.100.20.203，并返回给浏览器，浏览器根据该IP地址，访问真实的物理服务器114.100.20.203。所有这些操作对用户来说都是透明的，用户可能只知道www.apusapp.com这个域名。

###   
3.DNS域名解析负载均衡有如下优点：

1\. 将负载均衡的工作交给DNS，省去了网站管理维护负载均衡服务器的麻烦。

2\. 技术实现比较灵活、方便，简单易行，成本低，使用于大多数TCP/IP应用。

3\. 对于部署在服务器上的应用来说不需要进行任何的代码修改即可实现不同机器上的应用访问。

4\. 服务器可以位于互联网的任意位置。  
5\. 同时许多DNS还支持基于地理位置的域名解析，即会将域名解析成距离用户地理最近的一个服务器地址，这样就可以加速用户访问，改善性能。

###   
4.DNS域名解析也存在如下缺点：

  
1\. 目前的DNS是多级解析的，每一级DNS都可能缓存A记录，当某台服务器下线之后，即使修改了A记录，要使其生效也需要较长的时间，这段时间，DNS任然会将域名解析到已下线的服务器上，最终导致用户访问失败。

2\. 不能够按服务器的处理能力来分配负载。DNS负载均衡采用的是简单的轮询算法，不能区分服务器之间的差异，不能反映服务器当前运行状态，所以其的负载均衡效果并不是太好。

3\. 可能会造成额外的网络问题。为了使本DNS服务器和其他DNS服务器及时交互，保证DNS数据及时更新，使地址能随机分配，  
一般都要将DNS的刷新时间设置的较小，但太小将会使DNS流量大增造成额外的网络问题。

  
事实上，大型网站总是部分使用DNS域名解析，利用域名解析作为第一级负载均衡手段，即域名解析得到的一组服务器并不是实际提供服务的物理服务器，而是同样提供负载均衡服务器的内部服务器，这组内部负载均衡服务器再进行负载均衡，请请求发到真实的服务器上，最终完成请求。 
 [https://blog.csdn.net/weibo1230123/article/details/82946179](https://blog.csdn.net/weibo1230123/article/details/82946179)
````
