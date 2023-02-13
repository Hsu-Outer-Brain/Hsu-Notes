# (14条消息) Gerrit工作流程及使用手册_锐湃的博客-CSDN博客_gerrit
[(14 条消息) Gerrit 工作流程及使用手册\_锐湃的博客 - CSDN 博客\_gerrit](https://blog.csdn.net/chuyouyinghe/article/details/124024459) 

 [gerrit](https://so.csdn.net/so/search?q=gerrit&spm=1001.2101.3001.7020)的流程、权限控制其实对于初次接触的同学们来说，确实有点复杂。我希望这篇文章过后，我们能对 gerrit 的流程有一个大致的了解。  
这篇文章将用一个真实的例子，演示一下 gerrit 的管理员，普通项目成员是如何协同完成项目管理工作的。

这篇文章首先会大致讲解下 gerrit 的工作流程；然后介绍管理员的相关配置工作，包括设置[SSH](https://so.csdn.net/so/search?q=SSH&spm=1001.2101.3001.7020 "SSH")密钥验证，添加新成员；接下来会用一个示例演示普通成员 push 一个 commit 之后，代码审核员是如何进行审核的；最后介绍一下如何使用[sourceTree](https://so.csdn.net/so/search?q=sourceTree&spm=1001.2101.3001.7020)上传代码到 gerrit 服务器。

提醒：  
这篇文章需要一定的 git 基础，如果你还不熟悉 git，请先学习下关于 git 的相关知识。  
这篇文章中的所有操作不再需要登录到 gerrit 服务器上去了，因为所有的操作都是在管理员或者普通成员的电脑上完成的，至于管理员对组员的操作，也可以在管理员自己的电脑上通过 SSH 连接到 gerrit 服务器上完成。

\*\*

## gerrit 工作流程

\*\*  
好不容易在 google 上找了一篇相对简单明了的介绍 gerrit 工作流程的图：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/ddbad2e1-e117-4bbd-b38e-ee2bf9105209.png?raw=true)

使用过 git 的同学，都知道，当我们`git add --> git commit --> git push` 之后，你的代码会被直接提交到 repo，也就是代码仓库中，就是图中橘红色箭头指示的那样。

那么 gerrit 就是上图中的那只鸟，普通成员的代码是被先 push 到 gerrit 服务器上，然后由代码审核人员，就是左上角的 integrator 在 web 页面进行代码的审核 (review)，可以单人审核，也可以邀请其他成员一同审核，当代码审核通过(approve) 之后，这次代码才会被提交 (submit) 到代码仓库 (repo) 中去。

无论有新的代码提交待审核，代码审核通过或被拒绝，代码提交者 (Contributor) 和所有的相关代码审核人员 (Integrator) 都会收到邮件提醒。  
gerrit 还有自动测试的功能，和主线有冲突或者测试不通过的代码，是会被直接拒绝掉的，这个功能似乎就是右下角那个老头 (Jenkins) 的任务。

整个流程就是这样。 在使用过程中，有两点需要特别注意下：  
1、 当进行 commit 时，必须要生成一个 Change-Id，否则，push 到 gerrit 服务器时，会收到一个错误提醒。  
2、提交者不能直接把代码推到远程的 master 主线 (或者其他远程分支) 上去。这样就相当于越过了 gerrit 了。 gerrit 必须依赖于一个 refs/for/\* 的分支。  
假如我们远程只有一个 master 主线，那么只有当你的代码被提交到 refs/for/master 分支时，gerrit 才会知道，我收到了一个需要审核的代码推送，需要通知审核员来审核代码了。  
当审核通过之后，gerrit 会自动将这条分支合并到 master 主线上，然后邮件通知相关成员，master 分支有更新，需要的成员再去 pull 就好了。而且这条 refs/for/master 分支，是透明的，也就是说普通成员其实是不需要知道这条线的，如果你正确配置了 sourceTree，你也应该是看不到这条线的。

这两点很重要！！这两点很重要！！这两点很重要！！

这两点在稍后的示例中，我们会依次介绍。

## 管理员配置

在上一篇文章的最后，我们做为管理员，初次登录到了 gerrit 的 web 页面中，接下来还有很多内容需要设置一下。我们依次来看看吧。

## **profile**

**![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/8fdedf93-1e8b-443e-aa58-32410bd32736.png?raw=true)**

初次登录时，Full Name 和 Email Address 字段都是空的，待会我们会一一设置。没有设置之前，右上角显示的用户名称也是 admin

## **Preferences**

**![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/6ceee1f0-3279-4b45-9724-2c264eff64c4.png?raw=true)**

[Preferences](https://so.csdn.net/so/search?q=Preferences&spm=1001.2101.3001.7020)页面用于配置 gerrit 的 web 页面，我一般把时间格式改成习惯的 24 小时制，同时确保 Email Nofification 处于开启状态，这个默认就是开启的；然后在 Show Change Number In Changes Tables 打上勾，这样能清楚地看到每个审核的编号，邮件里面显示的也是这个编号。

## Watched Projects

这里有必要先说一下 gerrit 的两个默认项目：

我们点击左上方的菜单栏 Projects –> List，就能看到两个默认的项目 All-Projects 和 All-Users，这两个工程是两个基础的工程，我们新建的工程默认都是继承自 All-Projects 的权限。关于权限部分我们在后面的章节详细介绍。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/3ba17760-9610-4ddd-a749-fe3e1e912c3c.png?raw=true)

所以在 Watched Projects 菜单中，就是当前用户要监听的项目，当这些项目发生变化时，你会收到邮件提醒，如果你选择了 All-Projects，那么就意味着你要监听所有的工程，因为所有工程都会默认继承自 All-Projects。 

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/2421b539-6a10-434d-b977-19ffc29f40d7.png?raw=true)

注意，后面的那些选项，勾选了某一项，就表示仅仅给当前项发送邮件提醒，保险期间，我们就全部勾上就好了。

## Contact Information

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/ed447d01-845c-4e1c-9ef4-a0b6a008ff84.png?raw=true)

这里是当前账号的配置，你可以在这里把 full-name 填写完全，这样右上角也会同步更新你刚设置好的名称。

注意这里的 Preferred Email 有两种设置方法：  
1、如果你的邮件服务器配置完成了 (不清楚如何配置的同学请参见上一篇文章)，可以点击 Register New Email，你就会收到一封确认邮件，确认之后就能设置好了。  
2、通过 SSH 在命令行中进行配置，通过命令行进行配置是最方便快捷，也是最优先推荐的方法。

不过因为 SSH 命令行的方式需要配置 SSH 公钥，所以我们这里先留空，一会通过命令行来配置。

## SSH Public Keys

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/8d6f7d99-f3c9-4922-8882-5aec255e4c5f.png?raw=true)

这里需要把你的公钥内容拷贝出来，然后粘贴到对话框中。我简单演示下如何操作。

## 生成公私钥对

如果你之前有使用过 git，那你一定已经生成了公私钥对，可以直接跳过这个步骤。  
在命令行中输入下面的内容：

```null
$ ssh-keygen -t rsa

```

-   1

然后会提示你输入一个密码，用来访问公私钥，可以直接回车表示不加密码。接下来就会自动帮你创建好一对公私钥了。

## 找到公钥文件

默认公私钥的是放置在~/.ssh 目录下的，默认的名称是 id_rsa 和 id_rsa.pub。其中. pub 文件就是公钥，私钥你自己需要保存好。我们来看下我的. ssh 文件夹中的内容

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/d12ce235-9eb7-450d-9791-ff80ddac1e41.png?raw=true)

1、config 是 ssh 的配置文件，稍后我们会演示通过管理员 ssh 连接到 gerrit 服务器时，会在来看这个文件  
2、lipeng 和 lipeng.pub 分别是我的私钥和公钥

我们要做的，就是把公钥的内容拷贝出来，然后粘贴到页面上去。  
注意拷贝时，要从 RSA 开始拷贝，如果格式不对，页面上会有提示。  
还有一点要说明的是，这些都是在你想要作为管理员账号的机器上操作完成，对我来说，我的 macbook 就是管理员要使用的电脑。

## Groups

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/4156032c-ef13-4898-aafc-f775fe1be961.png?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/2a78e703-6e93-452d-8ce7-67161a0f027c.png?raw=true)

最后来看一下 gerrit 的分组。图片里面是 gerrit 默认的几个分组，我们需要知道的是 Administrator 就是管理员分组，Anonymous Users 指的是所有添加到 gerrit 数据库中的成员都默认加入的一个组。之后我们还可以建立新的分组，加入新的成员等等。

## 示例

接下来我们来做一个演示，看看一个新的成员是如何被添加到 gerrit 服务器中，然后他们又是如何协同工作的。  
这里一共涉及到两个角色，一个是管理员，一个是普通成员。

## 管理员设置 SSH

在之前的文章中我们提到过，gerrit 自带的 H2 数据库就完全够用了，对成员的管理，邮件添加等操作，均可以通过 SSH 来完成。那第一步我们就来看一下管理员如何才能远程 SSH 到 gerrit 服务器。

首先确保在之前，已经成功把你的公钥添加到了 web 页面账户中。  
接下来，需要修改之前~/.ssh / 文件夹下面的 config 文件，我们拿我的 config 文件做为示例，做个讲解。

我们还是先进入到~/.ssh / 文件夹中

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/ebc52213-08bd-4ad4-bff3-048171857ded.png?raw=true)

然后查看一下 config 文件： vim config 

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/470ab0ff-9a76-4c67-93be-aa15a6fb84de.png?raw=true)

我们看到这里面有两个 Host 部分，我们重点来看第 2 个 Host 部分，这个是我们新建的，用于连接到 gerrit 服务器的配置。

照猫画虎，对于我们之前建立在 192.168.1.100 的 gerrit 服务器来说，你的 Host 配置可能如下：

User 要和我们在 gerrit 服务器上注册的名称保持一致 (不是 full-name)，认证文件注意要和公钥对应的私钥文件，端口要填写 gerrit 服务的端口号，这里是默认的 29418。

配置完 config 文件，我们就可以 SSH 到 gerrit 了，我们来尝试一下吧：

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/f8abc334-f58f-49ca-a17f-b41dac63ee35.png?raw=true)

我们在管理员的机器上，输入 ssh gerrit -l admin 命令，就可以得到 gerrit 服务器的响应，只不过因为我们禁用了 shell，所以连接很快断开了，没有关系，这样证明做为管理员，已经可以通过命令行对 gerrit 服务器进行一系列的操作了。

## 添加普通成员

在管理员添加新的组员之前，我们需要先在普通成员的机器上生成 ssh 的公私钥，这里方便描述，我们把这个普通成员命名为 test3。

在 test3 的电脑命令行中，生成利用 ssh-keygen 命令，生成公私钥。

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/2b2bb04a-c8ee-43d4-990b-098e36289cfd.png?raw=true)

我们就使用默认的 id_rsa 命名好了。

接下来，test3 成员需要把 id_rsa.pub 公钥发送给管理员，这样管理员才能正常把 test3 添加到 gerrit 用户组中。  
我们假设管理员将 test3 的 pub 公钥放到了 / home 目录下，也就是说，在管理员的电脑上，test3 的公钥存放在 / home/id_rsa.pub 文件，当然我们也可以重新把它命名为 test3.pub，方便演示我这里就不做更名处理了，

接下来，管理员在命令行中输入如下的命令来完成添加普通成员的操作。注意： 这个命令很强大很方便，可以一步到位地把成员的的名称，全名，邮箱以及 ssh 公钥认证全部设置好。

```null
$ cat ~/home/id_rsa.pub | ssh gerrit gerrit create-account --full-name test3 --email test3@microwu.com --ssh-key - test3
```

接下来我们来详细看一下这个命令：

> | 符号把这个命令分成了两部分，第一部分的 cat ~/home/id_rsa.pub 表示把 test3 的公钥内容读入到输入流中  
> ssh gerrit 是我们之前在~/.ssh/config 中配置好的 gerrit 服务器地址  
> 又接着一个 gerrit 表示通过 ssh 中输入 gerrit 命令来进行相关操作  
> create-account 表示要新建用户。注意，新建的用户名写在最后面，中间是其他参数  
> full-name 就如同页面中的全名，我们这里命名为 test3  
> email 表示该用户的 email 地址，我们填入 test3@microwu.com  
> ssh-key - 注意，最后的 - 表示从输入流中读取 ssh 的公钥内容，也就是 | 符号之前我们读入的 test3 用户公钥内容  
> 最后面加上我们要 create 的用户名称

这个命令执行完之后，管理员就把 test3 用户加入到了 gerrit 用户组中，并且设置了他的全称，邮件以及公钥文件，是不是一步到位，非常方便？？

这里我们回过头来，在管理员首次登陆 web 页面进行修改配置的时候，我们说过，管理员的邮箱可以通过命令行来设置，是的，同样通过 ssh 命令行：

```null
$ ssh gerrit gerrit set-account --add-email admin@microwu.com admin

```

这个命令就表示为我们的 admin 用户添加 emailadmin@microwu.com。执行完这个命令，再回到 web 界面上的用户设置界面，看看是不是管理员的 email 已经被设置好了？？

## 修改用户所在组

接下来我们看一下怎样修改 test3 用户所在的组吧。我们知道他已经出在 Anonymous Users 组中了，那我们想要新建一个组，就叫 test_user 吧，我们来看一下

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/c4fe2623-bafd-4b4a-9c03-f8ff5dbf7701.png?raw=true)

我们在 gerrit 页面的顶部，点击 People –> list， 看一下默认的两个分组，Administrator 和 Non-interactive Users，这两个分组我们都能从字面上理解是什么意思。我们注意到 Anonymous Users 这个分组并没有显示在页面，因为它是匿名的嘛，所有的用户自动添加到这个分组中了。 

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/1ba590b9-ff7d-4f9b-959a-40258097e3a3.png?raw=true)

选择 Create New Group，输入我们要添加的新的分组 test_user 

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/a11722e4-5d25-459d-861a-e0fcff9527ff.png?raw=true)

新的分组中，我们看到管理员的账号被自动添加了进来 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/fecb7dbf-4877-460c-bf8f-ec91545900a5.png?raw=true)

我们在 Add 搜索栏中输入 test，就会自动显示出来管理员之前在命令行中创建的 test3 用户，看到 full-name 和 email 了吧，都已经添加完成了！  
test3 用户已经添加到了 test_user 分组中了。

## 新建和修改项目

用同样的方法，我们来新建一个项目吧

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/d5359c4e-25b0-4c50-ad6b-cac9267ea094.png?raw=true)

点击 Project，然后 Create New Project， 创建一个名为 test2 的项目吧。 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/46c1eb06-5e09-4981-a2b6-b1fa36ef96d0.png?raw=true)

可以通过点击 project 名称，进入到工程的详细设置界面。 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/84d44b39-f70e-4799-a6ae-251ce6ed1065.png?raw=true)

点击顶部菜单栏中的 Access，来设置这个项目的权限吧。  
我们可以看到这个项目已经有了个默认的继承自 All-Projects 的权限，关于默认的权限这里不做多的介绍，想要深入学习的同学可以点击进去看一下。

修改权限的时候慎重，不要直接修改 All-Projects 组的权限，因为这个是所有项目的依赖权限组，修改了以后，所有的项目权限都会跟着发生变化。

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/fb116e2e-b155-4318-a799-af771703cf72.png?raw=true)

我们点击 Edit 按钮来修改这个项目的权限 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/7c8a8af1-72b0-43de-9f4a-a647fa347e00.png?raw=true)

我们把 Reference 改成 refs/\*，表示所有的 refs 下的分支  
然后选择 read 项目

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/442f9cd4-e021-47ff-aa7c-d2d617b5e3e1.png?raw=true)

接着筛选不同的分组，并且赋予不同的权限。

```null
我们把Anonymous Users分组中的用户设置为DENY把我们刚建立好的，并且添加了test3用户的test_user分组权限设置为ALLOW
```

这样，当前的工程就只能被 test_user 组内的用户所访问，其他组的用户均无法访问了！

## test3 用户 clone 工程

接下来我们来到 test3 用户的电脑上，下拉刚刚创建的 test2 工程

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/bfcc6622-4bc5-4ffb-ac40-45582e80d2ca.png?raw=true)

在命令行执行下面的命令，就可以把 test2 工程给 clone 下来了

```null
$ git clone ssh://test3@192.168.1.100:29418/test2.git

```

对比下，会发现我们的目录中多了一个名为 test2 的文件夹，这个就是我们的工程了！

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/e79372a3-bd3b-4d87-9f9d-9d2d5038fdc5.png?raw=true)

注意如果从 ssh 方式 clone 下来的工程，里面是自带了 hooks 文件夹的，这个文件夹很重要！！如果不是用 ssh:// 方式克隆下来的，还没有这个文件夹，需要我们自己 mkdir

我们直接新建一个 test.md 文件，来尝试着往远程提交。  
注意： 前方会出现很多错误，耐心一步一步来

## 提交

首先我们通过下面两个命令首先 commit 到本地仓库：

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/d9e8d313-b7e3-4357-9c37-e22b203eca5e.png?raw=true)

在 commit 的时候，发现提交者的名称和 email 都是错误的，我们需要先配置成我们当前的 test3 用户，以及对应的 email

```null
$ git config user.name test3$ git config user.email test3@microwu.com
```

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/429764fd-2d24-46d0-a405-d043e99aeac3.png?raw=true)

通过 git config --list 来查看一下当前 git 仓库的配置，发现已经把用户名和密码正确设置了。

接下来我们继续 git commit 来提交到本地

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/10e6316d-7d1e-45ad-a8bc-d8cc23737ea2.png?raw=true)

我们把这次提交命名为 commit_1，然后我们通过 git push 命令来推送

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/23a71931-3185-4973-9e07-9f54c43ad3a5.png?raw=true)

发现推送失败了，提示的错误是：

```null
e not allowed to perform this operation[remote rejected] master -> master (prohibited by Gerrit)
```

Gerrit 拒绝了我们直接提交到 master 的推送！

这就是我们在文章开头提到的问题，我们需要 push 到 refs/for/master 那条线上！！

那怎么办呢？？  
我们在命令行写入下面的命令：

```null
$ git config remote.origin.push refs/heads/*:refs/for/*
```

这行命令的意思是，当执行 push 命令时，将会推送到 refs/for / 当前 head 所在的分支上。

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/1055fdc1-5acd-46d1-9eb9-12fb74a5cde7.png?raw=true)

我们设置了 push 命令之后，重新进行 push，结果又报错了。。  
这次的错误是：

```null
g Change-Id in commit message footer

```

这个是提到的第 2 个问题，commit 一定要有 Change-Id  
然后我们看到了命令行中给了我们提示，我们可以从 hooks 文件中拷贝 commit-msg 文件下来，这样 commit 时，会自动帮我们生成 Change-Id.

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/bcc311d1-4cbb-40f9-9930-35c2dd8e6dd9.png?raw=true)

我们可以看一下 git rev-parse --git-dir 就是指向的当前 git 配置的文件，就是. git 文件夹  
所以我们直接用 scp 命令从 gerrit 服务器上拉取当前用户的 hooks 文件。

```null
$ scp -p -P 29418 test3@192.168.1.100:hooks/commit-msg .git/hooks/

```

然后我们重新 push 发现一样的错误，因为我们还停留在上次 commit，上次的 commit 是没有生成 Change-Id 的！

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/fab5b5f3-1037-472c-b3d4-7ab5f1275343.png?raw=true)

没有关系，我们回退一下，然后重新提交。  
回退命令是先用 git log 找到上一次的 commit id， 然后用 git reset --hard 找到的 id 命令回退

这次我们终于提交成功了，可以看到提交到的分支是 refs/for/master  
管理员审查

接下来我们回到管理员的 web 页面，会发现，test3 用户刚提交的那条已经在页面中了  
点击顶部的 My --> Watched Changes

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/1dfb6432-558a-48f0-8ac0-45dfd442a451.png?raw=true)

因为之前我们已经监听了 All-Projects，而 test2 工程又是默认继承自 All-Projects 的，所以我们就可以收到了，如果当前管理员没有监听 All-Projects，就需要手动把 test2 项目加进来，否则是不会在 watched 页面看到这条推送提醒的。

与此同时，我们会收到来自 gerrit@microwu.com 发送来的邮件了：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/7efbbc5f-4fff-4c51-8439-09abe74bfc20.png?raw=true)

事件处理

我们通过点击 commit_1 可以进入到当前事件的详情页面：

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/e2e737a0-0fcd-4fe2-a88c-f0d3133c2a99.png?raw=true)

可以在页面中看到我们生成的 Change-Id，以及我们新添加的文件 test.md，点击以后可以看到每个文件的详情。  
注意屏幕中间的两个按钮：  
Reply 表示对这次事件的回应，里面可以有 5 个选项，表示当前审查人员对这个事件的打分：-2，-1，0，+1，+2， +2 表示直接同意，1 表示我同意了，需要别的人员来一起审核  
Code-Review+2 相当于直接打 + 2 分 

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/74025fa1-ec62-4a64-a4db-e77f35c5674a.png?raw=true)

我们还可以通过点击右边的小人，来添加新的审核人员

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/1ea28f01-6194-460d-9c65-0f09ec7ac321.png?raw=true)

reply 之后，如果分数够了 2 分，就可以直接 submit 到主线上去了 

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/b001bb7d-ddcb-4a16-beaa-8fd6cbe40266.png?raw=true)

合并了之后，可以在 My --> Changes 中看到我们的审核历史 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/bfa5b659-7c68-40f0-accc-aef861b7e9f8.png?raw=true)

到这里，一个完整的从普通项目成员提交，到代码审核人员检查的全过程，就结束了。  
SourceTree

SourceTree 是 git 的可视化工具，也是比较流行的 git 工具，通过 sourceTree 我们一样可以 commit，push 等，而且更方便直观地看到项目的历史等等。

在上面的示例中，我们都是通过命令行来完成 git 的相关操作的，那通过 soureTree，我们同样需要注意那两点：

这里还需要额外注意一点：  
如果我们是通过 soureTree 工具 clone 项目，而不是通过 ssh:// 方式来 clone，那么工程中的. git 文件夹中是没有 hooks 文件夹的，需要我们手动去文件夹中创建

我们在管理员的视角从 gerrit 服务器上拉取之前的 test2 工程吧：

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/3b355eeb-ddd0-43a7-9cd7-7f2f611c7e67.png?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/5a384794-9073-4da6-91b5-1f1b67871614.png?raw=true)

Source URL 中填入 gerrit:test2.git。 因为我们之前已经在. ssh/config 文件中设置好了名为 gerrit 的 HOST 所以这里就可以简写了 

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/f3b6b52c-54a0-4f54-a567-c11bd73fba00.png?raw=true)

可以看到 test3 用户提交的 commit_1，因为已经通过审核了，所以，就合并到 master 中了

我们到当前的目录中，看一下. git 文件夹，确实是没有 hooks 文件夹的

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/0a753361-bab3-4269-b8f9-cd1421f9e824.png?raw=true)

我们通过 scp gerrit:hooks/commit-msg hooks / 命令来拉取 commit-msg 文件 

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/e647ba02-f75c-4b8b-b1e2-68e817839325.png?raw=true)

同时通过 git config remote.origin.push refs/heads/_:refs/for/_命令来设置 push 命令  
设置自定义 push

虽然我们设置好了 push 命令到远程的 refs/for/_目录，但是如果我们直接用 SourceTree 中的 push 功能，我们会发现直接给我们在远程新建了一个 refs/for/_分支，而且 gerrit 也没有审核事件触发，这是因为 sourceTree 的 push 应该是有它自己的一些配置，所以这里我们需要自定义 push 事件，来完成将代码推送到正确的分支上。

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/63984c7a-03d9-45f3-bc85-240eedbb3f86.png?raw=true)

我们进入 SourceTree 的配置页面

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/b9b68c36-9c1a-4583-9f7f-5ffed6dfd0e6.png?raw=true)

点击 Custom Actions，然后输入命令的名字： push to gerrit  
Script to run： 指的是要执行的文件，我们这里把 git 的可执行文件目录放进来，如果是 windows 请自行找到该目录  
Parameters 就直接写入 push，表示执行的是 push 命令 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2010-14-00/ec026d46-e705-4534-9fee-d8bc4c32a582.png?raw=true)

最后当我们要通过推送到 gerrit 服务器时，在当前的分支上，右键，然后点击 Custom Actions，再选择我们刚创建的 push to gerrit 动作，就实现了推送到 gerrit 服务器的功能！！

好了，到这里，关于 gerrit 的所有内容都介绍完了！！！

转自：

【Gerrit】Gerrit 工作流程及使用手册\_程序员小明丶的博客 - CSDN 博客\_gerrit 使用教程  
[https://blog.csdn.net/weixin_45372436/article/details/105736466](https://blog.csdn.net/weixin_45372436/article/details/105736466)
