# logseq要怎么使用？它可以代替Roam Research吗？ - 知乎
```text
本文是全网第一篇logseq测评文--2020年8月6日
---------------------------
本文导航：
本文分为三大区块26个知识小节
都是笔者通过大半个月实践筛捡而来的经验干货
可使用crtl+f快速跳转到你需要阅读的小节
---------------------------
什么是Roam Research它的优点在哪？
简单解释“小粒度双链块引用”是什么？
保护数据隐私
---------------------------
什么是LOGSEQ
为什么要用LOGEWQ
怎么使用它？直接进入正题!
第一招 换主题
第二招图谱
用它怎么编辑文字和最基本的使用它？
四个关键语法
如何同步？
那么下边我们开始实际操作
怎么配置一个密令？
如此一来你就集齐了三件宝物 账号、同步文件路径、密令 距离彻底通关又近了一步
并没有 ！
遇到bug怎么办？
最后
---------------------------
使用技巧补充
如何从github把笔记文件下载或上传20200801
如何快捷使用语法命令
遇到不能同步怎么办？20200806（这个目前一定要会）
遇到Go to diff怎么办？
对双向链笔记软件的看法
最后还是强烈建议选择对隐私有充分保障的笔记软件！
logseq首发桌面版评测-20210129
```

**什么是 Roam Research 它的优点在哪？**

> Roam Research 是什么：  
> Roam Research 是一种新型的**双向链**笔记的典型代表，最近年非常火热（2020 年中旬）大量的软件都把它当作典型进行学习，比如 RoamEdit、wolai、obsidian、marginote、notion、workflowy 、印象笔记、Athens Research 等都在推出自己的的**双向链功能，**笔者对这些软件体验后发现目前学习的最好的就是 RoamEdit 和 logseq 这两款软件**。**  
> **分别网址：**  
> [roamedit.com](https://link.zhihu.com/?target=http%3A//roamedit.com/)  
> [http://logseq.com](https://link.zhihu.com/?target=http%3A//logseq.com)
>
> 优点：  
> 这类软件的优点在于它们能帮助你 “**便捷的最大化的让知识产生连接**” 从而帮助自己对知识产生较佳的思考、洞见、权衡等，其它方式或软件并不能很好达到类似效果，能达到也是比较吃力的。要注意这种优点是基于很重要的两个特性也就是 “**小粒度双链块引用” 和 “知识网络图谱”。** 
>
> 如果只是链接大小标题的双向链接在非常多年前的很多 Zettelkasten 笔记应用中已经有的功能，例如开源免费工具 [http://zettelkasten.danielluedecke.de/en/](https://link.zhihu.com/?target=http%3A//zettelkasten.danielluedecke.de/en/) ，如果只是网络图谱 TheBrain 这款软件也已经早已实现，这两种软件一直都不温不火。
>
> 有的朋友会说这种新兴笔记对我而言就是 “然并卵”，的确对一部分人来说这些工具就是毫无卵用的渣渣，但一定要知道每个人的人生其实很容易就会被自己的观点所左右，你想什么你的生活就会向什么方面发展，抱持开放眼光和抱持封闭眼观的人，生活一定不相同的。
>
> 拓展阅读：  
> [Roam Reserach 到底好在哪儿？](https://zhuanlan.zhihu.com/p/145384101)  
> [Hao Xu：Roam Research：将写作作为思考的倍增器](https://zhuanlan.zhihu.com/p/147717407?utm_source=wechat_timeline&utm_medium=social&s_s_i=aco9%2BxormidS1MpRgrFW86wo2Fj2y9sOdKNhr4lBo%2Fc%3D&s_r=1&from=timeline)

**简单解释 “小粒度双链块引用” 是什么？（**使用 Roam Research 来做解释**）**

![](https://pic4.zhimg.com/v2-5be3a2c5581e3b87fae67ea34f9ad0d7_b.jpg)

> 以上红色箭头所指的两个功能就是块引用功能，在别的笔记里写的内容可以通过这两个功能**引用**到现在文章内直接展现, 这种引用同时产生了现在文章与源文章见的 “**双向链 "（**现文章和源文章可以互相跳转），而且在当下笔记就能全局修改被引用的内容。

![](https://pic2.zhimg.com/v2-5e8fe750d389283cf78cd7e300634309_b.jpg)

> 注意 “这是一段文字” 左边有个小黑点，这个小黑点就是一个 “块”。Block Reference 只能引“一个小黑点” 也就是软件内一个小黑点后边的内容。下边的 Block Embed 可以引用到一个小黑点下缩进后的所有内容。
>
> 再简单一些可以很直观的把它理解为 “**多功能的复制粘贴**”  
> **“块引用 Block Reference”**  
> 可以理解为是复制 一个小点和其后边内容粘贴，这个被粘贴的内容可编辑、可全局修改、还可以返回源文。
>
> “**块嵌入 Block Embed”**  
> 就是 这个小点下缩进的小点 也可以一起被复制粘贴，其它功能和

**保护数据隐私**

> 能替代 Roam Research 的软件有其实有很多款，只要能实现**块引用、块嵌入、并且有带知识图谱功能**的软件都能很好的替代 Roam Research，很多数据需要上传软件商服务器的双向链笔记个人数据隐私都得不到保障，也不能保存到更可靠的自选网盘服务器内，_味着你数据是有一定安全风险的，对数据没有充分的自主权，软件商只要通过后台就可以查看你的私人笔记，甚至倒卖都是合法的_。_另外的数据风险比如：网站倒闭、封号、被网络攻击、服务器故障、bug 故障、获数据任意获倒卖、突然高额收费、等都是有可能发生的。最近就发生了一起比特币勒索某象用户的事件：《_[求助！账号被黑！](https://link.zhihu.com/?target=https%3A//www.douban.com/group/topic/188262008/%3Fdt_dapp%3D1%26dt_platform%3Dmobile_qq)》
>
> 本人千挑万选选得一款既是 Roam Research 类型又能数据不保存在软件商服务器的软件 也就是 logseq，下边介绍一下这款软件。

* * *

**什么是 LOGSEQ**

> [http://roamedit.com](https://link.zhihu.com/?target=http%3A//roamedit.com)和[http://logseq.com](https://link.zhihu.com/?target=http%3A//logseq.com) 几乎都和 Roam Research 差不多，也都是网页程序，因为 logseq 未来开源且数据不上传软件商服务器，隐私声明也是特别有保障的，所以本文侧重介绍 logseq。
>
> （特别注意 1：**roamedit 开发的更为成熟**，如果你不在乎隐私可以先用，因为开发周期更长有先发优势软件细节上做得更充分，roamedit 也无需在文章中多做说明它的各方面都很易用，roamedit 注册账号极为简单，其 [Roam Edit 使用手册](https://link.zhihu.com/?target=http%3A//roamedit.com/jx/home.php%3Fplugin%3Doutline/sandbox%26to%3Dzoom/inTopic/uF-topic-rY3ydAi) 对各种功能的介绍也是非常晰明了，20200820）
>
> （特别注意 2：logseq 目前使用的是 github 做 “云盘” 进行数据储存，因为 logseq 不能直接访问 github 的资源，所以软件作者在 logseq 内搭建了一个 cors-proxy 服务器作为公用服务器，笔记数据会**经过**这个服务器才能到达 github 上，通过这个 cors-proxy 服务器你的信息有暴露的可能性，所以有数据隐私要求的话最好自己搭建一个 cors-proxy，这可以让你的数据直接从本地到 github，下边有搭建教程，通过此教程的设置你的数据就会不经过官方 cors-proxy 服务器，数据就由你的设备直接通向 github 了，另外在手机上可用 termux 搭建。**作者知晓这个问题**所以 logseq 官方也有 cors-proxy 服务器的搭建教程，但是晦涩难懂而且操作后提示 npm WARN 失败无法搭建成功。这个问题 logseq 几个月后会推出桌面帮和 html 独立网页应用版彻底解决）
>
> [cors-proxy 搭建教程由 花降らし提供](https://link.zhihu.com/?target=https%3A//www.notion.so/miku233/Windows-Logseq-cors-proxy-3816821bb7974aad8accd8742aa82a72%3Fv%3Dcb7f5a6fcd12408d8cd3b129ff38fa50)。（如果你信任官方服务器可以不做这个操作）
>
> 从 logseq 作者处得知之后 2020 年 11 月份左右会有桌面端和可独立于网络的 html，所以数据隐私上未来你不会搭建 cors-proxy 也不是问题。目前由于网页版的特性只要有网络的地方就可以打开浏览器都开始使用，数据可以离线或同步到自选的大公司网盘、服务器或者本地保存，用户对自己数据有拥有充分的自主权**！**它支持块引用，也可以展现图谱。
>
> 暂时的缺陷：  
> 1、用户需要熟悉较为陌生的[http://github.com](https://link.zhihu.com/?target=http%3A//github.com)使用它做为网盘，也是该软件最劝退的地方。  
> 2、目前使用的是亚马逊服务器，网络很有可能会链接不上。  
> 3、目前没有桌面端、html 独立网页应用端，如果是长期离线状态下数据就不方便导入导出。  
> 4、在 2020 年 7 月才出测试版所以开发还不是很完善目前很多功能还比较初期，目前还是一个比较好的雏形状态（20200806）。现在就选出推荐给大家的原因是，这款软件的发展方向较好，未来开发完善后应该会是一款比较好的类 Roam Research 软件，如果你想要一款类 Roam Research 有没有数据隐私担忧的软件可以先试用它关注它，不过目前基本功能使用是问题不大的。

![](https://pic2.zhimg.com/v2-094f9a094def512629abc42bb57de231_b.png)

> （上图为作者对桌面端的描述，logseq 未来是可期的）  
> 从发展方向来看，**logseq 的使用体验更接近网状笔记更为好用**，看得出来它注重用户是否能很好的去进行双向链网状笔记的记录，**抓住了使用需求的核心痛点，这点非常重要。** 虽然目前只是测试版该有的功能也基本都有了，官方对中英文 bug 和功能需求响应较为及时。

![](https://pic1.zhimg.com/v2-48c1a0433388945188abda49ec3bac0c_b.jpg)

> 如果你是 ipad 用户可以直接使用[http://logseq.com](https://link.zhihu.com/?target=http%3A//logseq.com)通过 safari 在桌面上添加一个 logseq，后直接进入 logseq。

下边是款款软件更详细的入门教学

**为什么要用 LOGEWQ**

> 因为其功能和 Roam Research 差不大多也有自己的特色，目前开发阶段还是免费，未来收费应该也不会太离谱，数据也是以以离线储存或自选大厂云端同步三方靠谱服务器同步为主，数据安全个人数据隐私有保证，其官网上也写道：

![](https://pic4.zhimg.com/v2-fbaae5c3e4910b0bc34c192f326ac5a7_b.jpg)

**怎么使用它？直接进入正题!**

![](https://pic2.zhimg.com/v2-71fde74c7cd0aba305c19b1a8f6c0e0d_b.jpg)

> 最简单的办法就是直接进入官网直接使用，但是进入官网后你会发现网页黑成了一坨感觉什么不知道怎么用怎么办？
>
> **使用电脑或手机版谷歌浏览器**谷歌浏览器下载地址：  
> [谷歌浏览器 (Google Chrome)](https://link.zhihu.com/?target=https%3A//dl.pconline.com.cn/download/51614.html)

**第一招 换主题**

![](https://pic2.zhimg.com/v2-51cd8101835d81c6105ffc8930bc38a5_b.jpg)

> （嘿嘿嘿\~~ 见上图你是不是发现 Roam Research 几乎和 Roam Research 界面几乎一模一样，在此感谢 logseq 开发者开发如此的软件拯救大家的知识管理）
>
> 开发者应该是为了界面简洁的缘故很多按键都隐藏在右上角的 “三” 图标内和用户头像内。  
> 只需要点击 “三” 再点击黑暗主题即可换成人类普遍适应的浅色界面。
>
> 换完这个主题界面，暗色的阴暗色调带来的**压迫感**就会消失，各种选项按钮也凸显出来，你就能感觉到 “意~ 好像可以用了”，这种暗色在暗色环境下使用起来才会比较舒服。
>
> 然后你只要理解了下文几乎就可以较好的理解并使用这个 “浏览器数据版” 的 logseq，缺点是数据在浏览器你现在用的内部，“浏览器数据版”只建议试用千万不能当作生产力工具开始用有丢失数据风险，下文有提到如何把数据保持下来也就是保存到 github 的方法，把 github 理解成一个网盘就行。

[https://zhuanlan.zhihu.com/p/164299598​zhuanlan.zhihu.com![](https://zhstatic.zhihu.com/assets/zhihu/editor/zhihu-card-default.svg)
](https://zhuanlan.zhihu.com/p/164299598)

**第二招图谱**

![](https://pic4.zhimg.com/v2-bba1a51570e5f8d9079627a3a3dac1ff_b.jpg)

> 这个就很简单了点击 “三” 再点击上图位置，只要你在某笔记中建立过\[\[]]链接就可以出现图谱，非常简单。

* * *

**用它怎么编辑文字和最基本的使用它？**

![](https://pic4.zhimg.com/v2-dca50af0ec11c8d8a583796b3a83a70f_b.jpg)

> 可以把它默认语言设置为 org 格式（因为 org 格式的功能拓展性更强，如果你习惯使用 mardown 语法你可以选择 mardown 格式使用），在 logseq 里使用 org 的语法就可以对 org 模式下文字进行各种花样改变，比如\*B\*，中间的 B 字就会加粗。具体使用某种语法就要对应某种输入才会产生文字效果 help 里面有简单的**语法说明**，如下图。

![](https://pic2.zhimg.com/v2-22ef200cda0cb217d80f93da33700b35_b.jpg)

> 更详细的 Markdown 语法改变文字样式请参考  
> [Markdown 教程 | 菜鸟教程](https://link.zhihu.com/?target=https%3A//www.runoob.com/markdown/md-tutorial.html)  
> [Markdown 在线编辑器 - MdEditor](https://link.zhihu.com/?target=http%3A//www.mdeditor.com/)
>
> 更详细的 Org-mode 语法改变文字样式等功能请参考  
> [Org-mode 简明手册 - open source - 博客园](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/Open_Source/archive/2011/07/17/2108747.html%23sec-5-1)

![](https://pic3.zhimg.com/v2-33b87b7c75120ad0d990996d3c11d31e_b.jpg)

> （注意：Org-mode 你只需要了解部分处理文字的项目即可，比如 “**TODO** \[#A] ”就是一个 TODO 加一个标签 A，点击标签 A 能进入所有和标签 A 有光的内容，点击 TODO 右边的 T 还可以把 TODO 丢入 “现在就要进行” 的列表 NOW。再比如“表格”，要注意的是 logseq 并非完全支持所有 org-mode 语法，有的前边\*\*\*号在 logseq 内是不需要的）
>
> （Markdown 和 org mode 模式，我个人更推荐 org 模式，这种模式的拓展性和兼容性更强，可原生实现**块引用**！不过你平常使用 Markdown 格式已经比较喜欢，就可以选择 Markdown 模式）

**四个关键语法**

> 1、\[\[]] 这个你可以理解为搜索标题并引用  
> 2、\[\[#]] 这个你可以理解为搜索小标题并引用  
> 3、(()) 这个你可以理解为搜索关键词并引用段落也就是块引用  
> 4、 # 这个是不打空格是标签，打了空格是标题标题级别越小 #愈多。
>
> 这些语法怎么用呢？  
> 例如：在你打出 (( 后再继续输入关键字，然后选取你需要的就可以引用成功！也就创造了链接！有个前提就是你已经输入了一些内容在文档内也就是 logseq 内，关键词和你已经输入的内容有一样的词语或句子。

**如何同步？**

（强烈建议同步, 这步是使用这款软件的一个门槛，实际更多是 github 网站的使用方法，只要你能越过这个门槛就能很好的使用这个软件，logseq 的其他更方便同步方式开发者也在制作中，未来也许这种庞杂的方式就会优化掉）

> **官方这点上没说太明白, 还有很多人不知道 github 是什么导致大量的朋友就在这点上直接劝退，无法继续使用下去。现在还处于开发测试阶段 20200801，其它同步方法也没出来，所以我觉得有必要来分享一下到底 github 同步怎么用，会了就很简单，不过你不知道怎么搞那就太难了。** 
>
> 简单说就是注册一个 [http://github.com](https://link.zhihu.com/?target=http%3A//github.com/) 账户 ，让后在这网站建立一个同步文件夹，再进入设置项在开发人员选项（Developer settings）下配置一个**密令**出来，把文件夹地址和密令填到 logseq 软件内，退出账号清理浏览器数据再登陆即可完成 logseq 数据到 github 的同步工作，同步后文件为. md 格式可以下载，意味着做好笔记后数据可以掌握在自己手中。
>
> github 是什么？  
> 在此你只需把它理解为一个多功能的 “云盘” 就行了，仅次足以。  
> （它主要对于程序员很实用，可以保管代码等资源，就是所谓的代码仓库，实现程序代码的迭代更新或返回，软件编写的协作讨论等等，这些乱七八糟的完全无需了解）

* * *

**那么下边我们开始实际操作**

**(注册账号、建立同步文件、获得密令)**

> **怎么注册账号？**  
> 在下边地址注册即可，建议浏览器安装翻译插件。

[github 注册账号​github.com](https://link.zhihu.com/?target=https%3A//github.com/join%3Fref_cta%3DSign%2Bup%26ref_loc%3Dheader%2Blogged%2Bout%26ref_page%3D%252F%26source%3Dheader-home)

![](https://pic4.zhimg.com/v2-df5aa46d81e79b95f6466123b1d3a193_b.jpg)

> 注册后什么都不用选，进入你自己的邮箱验证一下。验证后什么都不用选我们开始创建这个 “比较难用云盘” 的同步文件。

![](https://pic3.zhimg.com/v2-02dace788758c9693f14d51cefc047ae_b.jpg)

> 点击左上角的猫图标选择这里即可。

![](https://pic4.zhimg.com/v2-fa0c4c68bbc26d9731c00792902a9507_b.jpg)
![](https://pic4.zhimg.com/v2-5e288234f48ad7a7c35071fa6d7f3e57_b.jpg)

> 给你的文件夹取个名字，和输入描述，还有你是否公开它。其它的不用管。此时你的网盘同步文件夹就建立好了。他的路径会在浏览器上，**记住这个路径之后用得上**。

**怎么配置一个密令？**

> 这个密令很多人第一次接触不知道是个什么东西，也是比较厉害的劝退内容。可以理解为它就是一个取东西的口令，就像**快递取货码一样**，你输入在[https://logseq.com/](https://link.zhihu.com/?target=https%3A//logseq.com/) 内 [github.com](https://link.zhihu.com/?target=http%3A//github.com/) 就可以取走 logseq 的文件到 github 上，放到 github 的指定文件中。

![](https://pic1.zhimg.com/v2-6511bf8a6a3787daa7afd3498e5ebbd8_b.jpg)

> 如图上所示进入开发人员设定（Developer settings）

![](https://pic1.zhimg.com/v2-48f13d6863c7c4d54e8bf0dd8c041dc8_b.jpg)
![](https://pic3.zhimg.com/v2-6816bfd294488d91f1f8c649a6cb7c66_b.jpg)

> 按图上所示设置，最后拉倒底部点击给 “我取件码”（Generate token）  
> 注意密（我取件码）令不要给别人，不在乎资料和账号安全的无所谓。

**如此一来你就集齐了三件宝物 账号、同步文件路径、密令 距离彻底通关又近了一步。** 

![](https://pic1.zhimg.com/v2-5092df74f6322f1865a7f233798ced80_b.jpg)

> 接下来进入 [https://logseq.com/](https://link.zhihu.com/?target=https%3A//logseq.com/) 用 github 账号登陆

![](https://pic4.zhimg.com/v2-86cd2eada65e0f05f134ee5d0b576aab_b.jpg)

> 执行以下内容：  
> 点击授权（Authorize tiensonqin）。  
> Markdown 和 org mode 模式，【【**强烈推荐 org 模式】】，这种模式下可以不出乱码完美块引用！不过你平常使用**Markdown 文本编辑器比较多就可以选择 Markdown 模式。  
> 填入你的电子邮件。  
> 填入三件宝物之一 “**同步文件路径**“**。**  
> （**此时有提示要输入的别内容对应输入即可，比如密令，有时不提示**）

![](https://pic3.zhimg.com/v2-28c2306e7ed04bcacc7c474cb6aa3972_b.jpg)

> 最后点击头像，选择退出，然后再登陆，你是不是想着现在已经通关了？

![](https://pic4.zhimg.com/v2-35bdb419d3ac21d1b2b0fc4242c6d773_b.jpg)

> 登录后如上图设置输入三宝之一 密令，**这个密令不输入或输入不正确，或者网络不行都会导致不同步！**  
> 登录账号后点击头像没有 settings 选项，你可以先执行下边的步骤，再反回来做这步，这步可能需要返回来做也有可能之前就提示你输入了，也有可能下边步骤后你再登录账号才提示你输入，因为开发初期软件还太年轻多少有些问题，但只要知道这个是需要填的就能解决这步的问题。

**并没有 ！**

![](https://pic1.zhimg.com/v2-a8c6a50ddfcf1cb2e53db901aaabb650_b.jpg)

> 清除你浏览器的所有缓存信息后再进入 [https://logseq.com/](https://link.zhihu.com/?target=https%3A//logseq.com/)

![](https://pic1.zhimg.com/v2-21fc04c7bc86087306530e4a6021b1ac_b.jpg)

> 进入账号后出现 “日期” 标题则说明已经成功登入！右边点为绿色说明同步成功！  
> （**此时如果有提示要输入的内容对应输入即可，比如密令，有时不提示**）  
> 若不成功可多重复清理浏览器后登陆几次，或者重新安装浏览器，建议使用谷歌浏览器。

![](https://pic4.zhimg.com/v2-429e15a988d8fb3cf05bae5a698974d3_b.jpg)

> 功能提示点击绿点后即可和网盘通讯，绿点显色橘色就是未同步，显示绿色就是同步成功，它是自动的，也可手动，点击绿点右边名字可以直接进入 github ” 网盘 “。  
> 一般在 github 后点击左上角猫的图像，下边就会出现你建立过的文件路径，对应点进去就是储存位置，github 的其它功能无需再过多实用和了解。

**遇到 bug 怎么办？**

> 正如可以在[Changelog](https://link.zhihu.com/?target=https%3A//logseq.com/blog/changelog) （版本更新日志） 看到的最早的一条是 2020 年 7 月 1 日，所以这个软件还正在成长当中，有很多的地方还没有完善，所以使用上会遇到不少问题，

![](https://pic3.zhimg.com/v2-620da8e2a52039442568c858c761eb6a_b.jpg)

> 如果你遇到问题可以，用**三宝之一**账号去这里反馈，开发者一般都能看到，重要的是开发者懂中文！我本人也反馈过几条问题，开发者效率目前还是很高的。

[BUG 反馈​github.com](https://link.zhihu.com/?target=https%3A//github.com/logseq/logseq/issues)

**如此这般，你就可以愉快的用上**[logseq](https://link.zhihu.com/?target=https%3A//logseq.com/) **了。** 

* * *

**最后**

![](https://pic2.zhimg.com/v2-95192e80b382df8144f4d1b14783e475_b.jpg)

[吉祥三宝​music.163.com![](https://pic2.zhimg.com/v2-fe10c2001d35eac7ea05b1c5ba8c3ccd_ipico.jpg)
](https://link.zhihu.com/?target=http%3A//music.163.com/song%3Fid%3D62373%26userid%3D253650087)

**送上一首音乐，祝知友们生活、学习、工作愉快！**

本人和此软件利益无关，纯粹兴趣分享。

* * *

## **使用技巧补充：**

**如何从 github 把笔记文件下载或上传 20200801**

> 打开你的 github，到对应的 “网页路径” 也就是三宝之一。

![](https://pic2.zhimg.com/v2-4a4a2c975234ff26141910217226651d_b.jpg)

> 找到 code 点击，再点击下载为 zip 就可以保存到本地。

![](https://pic4.zhimg.com/v2-25d9fa61054ab3f9265a66acdf7cbe5f_b.jpg)

> 如果需要上传就按上图操作即可。

![](https://pic2.zhimg.com/v2-4501c459fb582eb759a1f78d8111818d_b.jpg)

> 上传后到[http://logseq.com](https://link.zhihu.com/?target=http%3A//logseq.com)后点击绿点，再点击拉取即可更新到[http://logseq.com](https://link.zhihu.com/?target=http%3A//logseq.com)中。

**如何快捷使用语法命令**

> 如果你平常觉得输入语法麻烦可以使用快捷命令 /

**遇到不能同步怎么办？20200806（这个目前一定要会）**

![](https://pic4.zhimg.com/v2-261bf9d9dd2c7eb69329ee5d643d21d3_b.jpg)

> 由于现在软件还处于开发初期，同步很不稳定，如果你文件不能同步时可以点击头像，如上图找到你的对应笔记进行本地保存，保存格式为 .md 格式，这样保存后就不怕丢失可以继续使用了。  
> 登出后处于安全考虑笔记都会被清空，所以无法同步的情况下你可以用这个方法进行备份再上传到 github 上，自行完成同步。

![](https://pic2.zhimg.com/v2-d232959d6ad69639d01650d31dcdce9d_b.jpg)

**遇到**Go to diff**怎么办？**

> Diff 其实就是你现在浏览器缓存和 github 云盘的数据有差异你要选择保留那边的内容。
>
> 在多个客户端同时使用[http://logseq.com](https://link.zhihu.com/?target=http%3A//logseq.com)时, 某端添加笔记或修改内容后笔记后，自动同步时会提示 Go to diff。  
> 很多人不理解 Go to diff、Diff 是什么（大量中文用户都不是 github 用户从未接触过，也不了解同步是什么东西），即使会英文也不一定理解 github 这些词语意义，同步问题云里雾里劝退很多人，只要对应以下内容你就可以轻松设置。
>
> Diff（差异同步选择）  
> Diff 下出现（红字为本地新内容云端没有，绿字为云端内容本地没有）。  
> keep local（本地覆盖云端）  
> use remote（云端覆盖本地）  
> commit and force pushing (本地强制覆盖云端)  
> edit （内容编辑）  
> Go to diff（去往差异同步选择）

* * *

**对双向链笔记软件的看法**

> 选择双向链笔记的用户更多都是冲着 **双向链、块引用、网状笔记** 这些特性是否能很好的使用**帮助自己阅读、思考、链接知识**而来，而不是为了是否能保持 Markdown 格式给其它 Markdown 软件打阅读而来，**是否保持了 Markdown（.md）其实没必要作为双向链笔记软件开发的第一考量,**Markdown 格式一般人根本不知道是东西也根本不在乎也不想了解它是什么。
>
> 选择双向链笔记的用户更多都是冲着 **双向链、块引用、网状笔记** 这些特性是否能很好的使用**帮助自己阅读、思考、链接知识**而来，而不是为了是否能保持 Markdown 格式给其它 Markdown 软件打阅读而来，**是否保持了 Markdown（.md）其实没必要作为双向链笔记软件开发的第一考量,**Markdown 格式一般人根本不知道是东西也根本不在乎也不想了解它是什么。
>
> 需要分享或用别的软件编辑时考虑到的也会是一些主流格式比如 pdf、doc、jpg 等，主流格式能在任何使用场景下方便的阅读、分享、备份。是否是 .md 这种文本格式对于大部分人而言并不重要。如果以 Markdown 格式捆绑软件在双向链开发方向上会有很多限制，意味着这也的软件功能潜质上顶到天花板的概率更大。

* * *

**最后还是强烈建议选择对隐私有充分保障的笔记软件！**

> 很多在线笔记软件的隐私声明几乎都是让你再也没有隐私，它们的隐私声明让你看到怀疑人生，此处就不列举了。
>
> 以下是 logseq 的隐私声明链接  
> [https://logseq.com/blog/privacy-policy](https://link.zhihu.com/?target=https%3A//logseq.com/blog/privacy-policy)

![](https://pic3.zhimg.com/v2-ebe6b9df27b7e7d5ea4a42a148c7a532_b.jpg)

* * *

## **logseq 的更多相关文章和网页**

推荐教程：[Hsun Cheung：Logseq 小白系列教程入门篇一](https://zhuanlan.zhihu.com/p/343854552)

桌面下载地址：[https://github.com/logseq/logseq/tags](https://link.zhihu.com/?target=https%3A//github.com/logseq/logseq/tags)

logseq 中文论坛：[Logseq 中文社区](https://link.zhihu.com/?target=https%3A//cn.logseq.com/)

logseq 外文论坛：[Logseq](https://link.zhihu.com/?target=https%3A//discuss.logseq.com/)

logseq 开发线路图：[Trello](https://link.zhihu.com/?target=https%3A//trello.com/b/8txSM12G/logseq-roadmap)

笔者之前的测评 1 [威廉：logseq 要怎么使用？它可以代替 Roam Research 吗？](https://zhuanlan.zhihu.com/p/165657050)

笔者之前的测评 2 [重磅 logseq 桌面版 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/348008490) 】

笔者的测评 3 [威廉：双向链笔记综合体验威廉排名榜](https://zhuanlan.zhihu.com/p/267451435)

笔者的测评 4 [威廉：重磅！Logseq 即将重构成功，将成为最好的 Markdown 大纲笔记软件兼容 Obsidian 用户！支持 SQL、正则式搜索！](https://zhuanlan.zhihu.com/p/364013092)

## **拓展阅读**

[威廉的《RemNote》专栏](https://www.zhihu.com/column/c_1343929082308993024)

[威廉的《Obsidian》专栏](https://www.zhihu.com/column/c_1327704371744546816)

## **Logseq 软件交流**

[开黑啦 - Logseq 聊天服务器](https://link.zhihu.com/?target=https%3A//kaihei.co/9CkRiI) 
 [https://zhuanlan.zhihu.com/p/165657050](https://zhuanlan.zhihu.com/p/165657050)
