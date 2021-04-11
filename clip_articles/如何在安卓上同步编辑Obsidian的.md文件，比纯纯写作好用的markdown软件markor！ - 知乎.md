# 如何在安卓上同步编辑Obsidian的.md文件，比纯纯写作好用的markdown软件markor！ - 知乎
## 笔记安全与便捷可以同时兼顾

用 obsidian 的朋友大多都是为了数据不走三方服务器，从而避免服务商窥探自己数据保护自己隐私为出发点，也避免了数据在线的不安全泄露问题。

自己设置搭建不同设备的同步可以大大的保证自己的数据隐私的同时，拥有在多设备间的方便使用的便利性，比如你是律师、作家等安全肯定就是必要的。

同步和加密的问题其实在下边两篇文章中已经提到解决, 这篇文章就是对 “安卓手机” 怎么用的补充。

【[Obsidian 手机平板使用之二 1Writer 和 IA writer 等及一定要注意的图片和笔记问题！ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/220081353) 】

【[用 Obsidian 必备的 Cryptomator 和微力 Sync 同步（加密本地与网盘笔记方法） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/339981965) 】

* * *

![](https://pic1.zhimg.com/v2-9391f3a5663589c4e364c846d722e0d8_b.jpg)

（上图为手机端使用的一种方式）

## **直接讲办法 - P2P 传输 +**markor

电脑上安装 Obsidian、Resilio Sync 或微力同步。

安卓手机上安装【[点击下载 markor](https://link.zhihu.com/?target=https%3A//github.com/gsantner/markor/tags) 】、Resilio Sync 或微力同步即可。

markor 是重点所在，全网中文文章提到 markor 这款的很少，它是很不错的安卓手机 markdown 软件可以和 obsidian 的连用。（ 这篇博文也有关于 markor 的用法感兴趣的朋友可以去看 【[分享自己的离线笔记](https://link.zhihu.com/?target=https%3A//unee.wang/post/20200215note) 】 ）

markor 怎么下载呢，只要进入【[点击下载 markor](https://link.zhihu.com/?target=https%3A//github.com/gsantner/markor/tags) 】选择版本号后下载 “.apk” 即可。**markor 这款软件实测比需要付费百元的纯纯写作好很多，功能丰富完善，是笔者用过安卓最好的无需登录、注册、联网的安卓 markdown 软件**，无需账号、无需任何登录即可离线使用，只要手机的读取本地文件的限权。

只要选择安装完上边提到的软件，百度一下 Resilio Sync 或微力同步的用法（实际就是两边安装后设置同步文件夹，获取 A 设备的一个码在另外的 B 设备输入即可同步），就可把电脑文件轻松同步到手机，手机文件轻松同步到电脑。Resilio Sync 或微力同步不同于网盘，只要你在设备上安装，各设备互为服务器（p2p 传输），手机和电脑同时在线时软件会以最新编辑的版本在设备间进行更新。

![](https://pic1.zhimg.com/v2-a30588f6346e6027db2b21297a618540_b.jpg)

特别注意的是 Resilio Sync 设置里边选择性同步要关闭，这样才能完整的自动进行同步。更多的 Resilio Sync 或微力同步用法百度上已经很多，再此就不累述了。

* * *

## **更安全的办法 -**Cryptomator

如果你是律师或作家之类的职业，对保密需求就会要求更强，就可以采取电脑上 Cryptomator、obsidian+Resilio Sync 或微力同步，安卓手机用 Cryptomator+Resilio Sync 或微力同步进行，这个方法原理就是用 Resilio Sync 或微力同步，同步 obsidian 的文件，用 Cryptomator 做各设备上的加密和自动解密。

Cryptomator 安卓端折合人民币差不多 90 元（pc 端免费）自带比较原始的. md 编辑功能。你有数据更安全的需求安卓上就无需使用 markor 直接使用 Cryptomator+Resilio Sync 或微力同步即可，当然你用 Cryptomator 时也能把文件用 markor、file manager 之类的软件打开编辑。

另外也可以用 Cryptomator 配合网盘使用，网盘服务商无法看到你的内容的前提下在不同端同步也是可以的，Cryptomator 厉害在你设备上是没加密一样的存在方式，就中间的网盘部分看不到内容，这就是不用 Resilio Sync 或微力同步的方式。

* * *

## 方法拓展

下边这篇文章讲了如何在非局域网、跨城市的情况下也使用 Resilio Sync、微力的办法。

[威廉：用蒲公英和 Resilio Sync 免费同步 Obsidian 笔记的方法（稳定同步不限流量跨城市可用）](https://zhuanlan.zhihu.com/p/352128221)

* * *

## **一定需要注意的问题，通用的图片语法、手机版一定要注意的问题**

这是在不同 markdown 软件间使用一定要注意的问题：

[威廉：用 Obsidian 一定要注意的图片和笔记问题！](https://zhuanlan.zhihu.com/p/268309654)

[威廉：使用 Obsidian 手机版一定要注意的问题！](https://zhuanlan.zhihu.com/p/351859263)

* * *

![](https://pic3.zhimg.com/v2-6879d036a7c6f25a2f31c2a7ef23144e_b.jpg)

## **总结**

按照上文的办法，使用网盘也可以用 Cryptomator 配合进行配合，如果是微力或 Resilio Sync 再加 Cryptomator 那就是点对点的**双重加密传输**（微力或 Resilio Sync 加密了一次，Cryptomator 又加密了一次），数据安全极大的得到了保障，性价比和安全度都是远超官方 4 美金一月的同步服务（约 30 元人民币，当然真正有什么公司机密之类的文件还是用 U 盘吧，**保密数据不上网**）。

因为微力同步是非开源软件传输过程安全度是没 Resilio Sync 高的这点值得注意到，如果你有网速需求微力同步可以是首选的，要求数据安全度那 Resilio Syn 就是首选的（某些情况下 Resilio Syn 需要配合贝锐科技的 “花生壳” 使用才能传输数据），网盘方面一款叫[nextcloud](https://link.zhihu.com/?target=https%3A//nextcloud.com/)的安全程度是很多笔友认为比较高的。

Obsidian 官方的同步服务意味着你的数据还要经过他们的服务器停留，停留的时候是否被偷窥也是看服务器管理者的人品，所以笔者认为，这方面还是自己动手丰衣足食。

**只是手机到电脑同步需求的话微力或 Resilio Sync+**[markor](https://link.zhihu.com/?target=https%3A//github.com/gsantner/markor/tags)**就完美了**，无需使用 Cryptomator，笔者自己已经入了 Cryptomator 全家桶，安卓 90、ios 60 元、PC 免费，其实也是很少使用。

* * *

## **拓展阅读**

[威廉 Obsidian 使用心得系列文章总汇（干货满满） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/336860525/)

## 软件交流

![](https://pic1.zhimg.com/v2-3a247dd22b793c02e67e4bcbbf0e29a4_b.jpg)

-   以下两个方式都是为笔记爱好者们提供的纯净中文交流环境无任何商业用途。推荐优先加入 "开黑啦" 聊天室，因为 QQ 群有满员的问题，开 "黑啦" 没有人数限制。
-   【[欢迎点击加入 Obsidian and RemNote 聊天室（开黑啦服务器号 ID 38993595）](https://link.zhihu.com/?target=https%3A//kaihei.co/Yz3ciI) 】
-   【[欢迎点击加入 Obsidian and RemNote QQ 群（群号 1026788769）](https://link.zhihu.com/?target=https%3A//jq.qq.com/%3F_wv%3D1027%26k%3D98OOipKU) 】 
    [https://zhuanlan.zhihu.com/p/347288495](https://zhuanlan.zhihu.com/p/347288495)
