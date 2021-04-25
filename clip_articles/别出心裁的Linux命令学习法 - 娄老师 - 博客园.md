# 别出心裁的Linux命令学习法 - 娄老师 - 博客园
操作系统操作系统为你完成所有 “硬件相关、应用无关” 的工作，以给你方便、效率、安全。操作系统的功能我总结为两点：**管家婆**和**服务生**：

-   管家婆：通过**进程**、**虚拟内存**和**文件**三个重要抽象管理计算机的 CPU、内存、I/O 设备。
-   服务生：为用户提供**shell**, 为程序员提供**系统调用**。

大家都比较熟悉 Windows 操作系统，Linux 也是一种操作系统。Linux 的架构如下图：

![](https://pic002.cnblogs.com/images/2012/413416/2012092023590167.jpg)

如果使用 GUI，Linux 系统和 Windows 操作系统的使用没有什么大的区别，用鼠标可以解决大部分问题。

Linux 学习应用的一个特点是通过命令行进行使用。学习使用 Linux，[实验楼](https://www.shiyanlou.com/)推荐的[学习路径](https://www.shiyanlou.com/paths/5)如下：

![](https://img2018.cnblogs.com/blog/741560/201904/741560-20190402102018018-410045507.png)

命令行的好处主要是可以批处理并自动化，还有些功能 GUI 无法完成，大家可以慢慢学习体会。

那么多命令先学什么，后学什么是一个大问题，本文期望找一种方式，通过解决 “Where” 的问题，通过几个核心命令的学习，让你可以举一反三通过实践学习其他命令，从而解决 Linux 命令的 “what” 问题。

我们使用的 Linux 发行版是 Ubuntu, 使用 Ubuntu 有几个快捷键要掌握一下，可以提高使用命令行的效率：

-   `CTRL+ALT+T`: 打开终端，天天使用终端，用鼠标打开太低效了；
-   `CTRL+SHIFT+T`：新建标签页，编程时有重要应用；
-   `ALT + 数字 N`：终端中切换到第 N 个标签页，编程时有重要应用；
-   `Tab`: 终端中命令补全，当输入某个命令的开头的一部分后，按下`Tab`键就可以得到提示或者帮助完成；
-   `上下键盘`：切换命令历史，刚输入一个很长的命令，按`上`键就可以恢复；
-   `CTRL+C`: 中断程序运行。

## 1 Linux 命令

登录 Linux 后，我们就可以在 #或 $ 符后面去输入命令，有的时候命令后面还会跟着`选项`（options）或`参数`（arguments）。即 Linux 中命令格式为：

```null
command [options] [arguments] 
```

其中`选项 (option)`是调整命令执行行为的开关，`选项`不同决定了命令的显示结果不同,`参数 (arugment)`是指命令的作用对象。

如 ls 命令，`ls`或`ls .`是两条等价的命令，显示是当前目录的内容，这里**“.”**就是参数，表示当前目录，这个参数缺省可以省略。我们可以用`ls -a .`显示当前目录中的所有内容，包括隐藏文件和目录。其中**“-a”** 就是选项，改变了显示的方式，如下图所示：

![](https://img2018.cnblogs.com/blog/741560/201904/741560-20190401081302114-1910653543.png)

以上简要说明了 Linux 命令以及选项和参数的区别，但具体 Linux 中哪条命令有哪些选项及参数，以及如何使用，需要我们靠经验积累或者查看 Linux 的帮助文档了。

## 2 man 命令

不论学习编程还是 Linux 命令，掌握帮助文档的使用都是很重要的，是举一反三的重要途径。

`man`是 manual 的缩写，我们可以通过 man man 来查看`man`的帮助，如下图：

![](https://img2018.cnblogs.com/blog/741560/201904/741560-20190401082118252-1644376891.png)

![](https://img2018.cnblogs.com/blog/741560/201904/741560-20190401082131848-489476442.png)

帮助文档包含：

```null
    1 Executable programs or shell commands（用户命令帮助） 
    2 System calls (系统调用帮助)  
    3 Library calls (库函数调用帮助)  
    4 Special files (usually found in /dev)  
    5 File formats and conventions eg /etc/passwd（配置文件帮助）  
    6 Games  
    7 Miscellaneous (including macro packages and conventions), e.g. man(7), groff(7)  
    8 System administration commands (usually only for root)  
    9 Kernel routines [Non standard]  
```

解释一下：

```null
1是普通的Linux命令  
2是系统调用,操作系统的提供的服务接口 
3是库函数,  C语言中的函数
5是指文件的格式,比如passwd, 就会说明这个文件中各个字段的含义  
6是给游戏留的,由各个游戏自己定义  
7是附件还有一些变量,比如向environ这种全局变量在这里就有说明  
8是系统管理用的命令,这些命令只能由root使用,如ifconfig 

```

其中 1，2，3 是初学时学习的重点，区别大家练习一下就知道了，比如 printf 是 C 语言的库函数，也是一个 Linux 命令，大家尝试一下`man printf`,`man 1 printf`,`man 3 printf`, 体会一下区别。

知道 printf 命令也好，printf 函数也好，查找帮助文档都很容易。`man`有一个`-k` 选项用起来非常好，这个选项让你学习命令，编程时有了一个搜索引擎，可以举一反三。

我们通过一个例子来说明，比如数据结构中学过排序（sort）, 我不知道 C 语言中有没有完成这个功能的函数，可以通过 “man -k sort” 来搜索，因为是找 C 库函数，我们关注带 3 的，qsort 好像是个好选项，如下图：

![](https://img2018.cnblogs.com/blog/741560/201904/741560-20190401082400082-491107507.png)

结合后面学习的 grep 命令和管道，可以多关键字查找：

```null
man -k key1 | grep key2 | grep key3 | ...
```

例如使用`man -k sort | grep 3`，可以更好的找到 qsort，你可以试试。

**`man -k`** 有个等价的命令`apropos`, 大家可以学习一下。

使用`man -k`找到命令后，可以用`man -f cmd`查看命令的基本功能。`man -f`等价于`whatis`.

## 3 cheat 命令

man 虽然很重要，但有些命令看了帮助还不会用，初学者需要例子，cheat 就是这个身边的小抄。  
cheat 命令不是 Linux 自带的，大家参考[这篇文章](http://linux.cn/article-3760-1.html)（[英文版](http://www.tecmint.com/cheat-command-line-cheat-sheet-for-linux-users/)）安装，实验楼课程实验系统中已经安装了。Ubuntu 中已经默认安装好了 python,cheat 安装过程如下：

```null
sudo apt-get install python-pip git
sudo pip install docopt pygments
git clone https://github.com/chrisallenlane/cheat.git
cd cheat
sudo python setup.py install

```

cheat 是作弊，小抄的意思。

cheat 命令是在 GNU 通用公共许可证下，为 Linux 命令行用户发行的交互式备忘单应用程序。它提供显示 Linux 命令使用案例，包括该命令所有的选项和简短但尚可理解的功能。比如强大的`find`命令，初学者看完帮助文档后可能还是一头雾水，那么我们`cheat find`一下：

```null

\*To find files by case-insensitive extension (ex: .jpg, .JPG, .jpG):

find . -iname "*.jpg"

\# To find directories:

find . -type d

\# To find files:

find . -type f

\# To find files by octal permission:

find . -type f -perm 777

\# To find files with setuid bit set:

find . -xdev \( -perm -4000 \) -type f -print0 | xargs -0 ls -l

\# To find files with extension '.txt' and remove them:

find ./path/ -name '*.txt' -exec rm '{}' \;

\# To find files with extension '.txt' and look for a string into them:

find ./path/ -name '*.txt' | xargs grep 'string'

\# To find files with size bigger than 5 Mb and sort them by size:

find . -size +5M -type f -print0 | xargs -0 ls -Ssh | sort -z

\# To find files bigger thank 2 MB and list them:

find . -type f -size +20000k -exec ls -lh {} \; | awk '{ print $9 ": " $5 }'

\# To find files modified more than 7 days ago and list file information

find . -type f -mtime +7d -ls

\# To find symlinks owned by a user and list file information

find . -type l 

\# To search for and delete empty directories

find . -type d -empty -exec rmdir {} \;

\# To search for directories named build at a max depth of 2 directories

find . -maxdepth 2 -name build -type d

\# To search all files who are not in .git directory

find . ! -iwholename '*.git*' -type f

\# Find all files that have the same node (hard link) as MY_FILE_HERE

find . -type f -samefile MY_FILE_HERE 2>/dev/null

```

通过实践结合 man 命令把上面的例子理解了，`find`基本上就用的很不错了。

使用 cheat 命令作弊是可以的。：）

## 4 其他核心命令

和查找相关的核心命令还有`find`,`locate`,`grep`,`whereis`,`which`等, 其中：

-   find 查找一个文件在系统中的什么位置，locate 是神速版本的 find（Windows 下有个神器[Everything](http://www.voidtools.com/)和 locate 功能类似）。可以通过`cheat find`学习`find`命令。
-   grep 可以对文件全文检索，比如你接手一个 C 语言项目，里面有上百个 C 源文件，想找找 main 函数在那个文件中，你可以通过`grep -n main *.c`, 快速找到 main 在哪个 C 文件中并指出在第几行。grep 支持[正则表达式](http://www.cnblogs.com/rocedu/p/4934924.html)，[正则表达式](http://www.cnblogs.com/rocedu/p/4934924.html)也是一个重要的元知识。可以通过`cheat grep`学习`grep`命令。上面还提到，
-   whereis,which 告诉你使用的命令工具装在什么地方。Linxu 初学者会不习惯 Linux 的文件系统，C 盘呢？D 盘呢？用`apt-get install`安装程序好象也不用我们选择安装位置，程序装在哪了？比如：我们在 Linux 下上网使用 firefox 浏览器，大家可以使用`whereis firefox`或更精确的使用`which firefox`来看看结果。
-   apt-cache 可以在使用 apt-get install 安装一个程序时先找找软件源的库里有没有这个程序，有才可以安装。比如老师推荐了一个调试工具`ddd`, 你可以用`apt-cache search ddd`查查有没有这个程序。

## 5 终端 “每日提示” 学习

主动学习是学习方法中非常重要的一点，我们可以 “问题驱动”，遇到问题用 "man -k key" 搜索相关 Linux 命令，然后用 "man cmd" 查看帮助文档结合“chea cmd” 来学习，开始学习时我们也可以每次打开终端就学习一个 Linux 命令，这样日积月累，就能更好的掌握 Linux 命令了。

我们可以让终端自动给我们个每日提示，让我们的学习自动化：

更娱乐化一点，我们先安装 cowsay。在 Ubuntu 下安装 cowsay：

```null
sudo apt-get install cowsay
```

安装 cowsay 后，在. bashrc 中增加下面一行：

```null
cowsay -f $(ls /usr/share/cowsay/cows | shuf -n 1 | cut -d. -f1) $(whatis $(ls /bin) 2> /dev/null | shuf -n 1)
```

每次打开终端，就会出现类似的界面，你就可以了解一个 Linux 命令了：  
![](https://img2018.cnblogs.com/blog/741560/201904/741560-20190401084951922-359271145.png)

![](https://img2018.cnblogs.com/blog/741560/201904/741560-20190401085008877-1037806106.png)

## 6 小结

上面的命令包括 man -k 有一个共同特点就是基于 “搜索” 解决了学习方法问题，这些命令学好用熟后可以举一反三，大家重点学习，掌握了他们，其他命令就可以自学了。特别是 man(man -k)和 cheat 对 Linux 命令学习非常重要。

我们课程涉及到的 Linux 命令在下面列出了，使用上面的**搜索命令**通过实践学习吧：

```null
ac,apt-get,apt-cache
bzip2,
cat,cd,chgrp,chmod,chown,clear,compress,cp,
dd,ddd,df,diff,du,dump,
env,
find,finger,free,
gcc,gdb,grep,gzip,
head,
kill,
less,ln,locate,l,ls,
make,man,mkdir,more,mount,mt,mv,
netstat,nslookup,
od,objdump
passwd,patch,ps,pstop,pwd,
rm,rmdir
shell,sort,ssh,stty,
tail,tar,telnet,touch,tree,
umask,uname,unzip,
vi,vim,
whereis,which,who,write
...
```

[命令行的艺术](https://github.com/jlevy/the-art-of-command-line/blob/master/README-zh.md)([The Art of Command Line](https://github.com/jlevy/the-art-of-command-line/blob/master/README.md)) 给出了一个由浅入深的学习路径。

学习 Linux 命令，[Linux 命令大全](http://man.linuxde.net/)是个很好参考，集合了 man 和 cheat 的功能，查阅起来很方便。

更多 Linux 命令的学习也可以参考[这里 (英文)](http://www.computerhope.com/unix.htm#04)还有[这里 (中文)](http://itlab.idcquan.com/linux/special/linuxcom/), 以及[O'Reilly Linux](http://archive.oreilly.com/linux/cmd/)、[The Linux Cookbook](http://www.dsl.org/cookbook/)和[LinuxCommand](http://www.linuxcommand.org/)。

除了这些文档，当遇到不懂的命令时，也可以到[Explain Shell](http://explainshell.com/)，让它给解释一下。

如果自己没有 Linux 环境，结合上面讲的搜索命令，可以去[实验楼](https://www.shiyanlou.com/)学习《[Linux 基础入门（新版）](https://www.shiyanlou.com/courses/1)》课程实践练习。

## 参考资料

1.  [Linux 命令速查手册](http://book.douban.com/subject/4046184/) ([电子版](http://www.duokan.com/book/11732),[Linux Phrasebook](http://book.douban.com/subject/1829759/))
2.  [Unix/Linux 编程实践教程](http://book.douban.com/subject/1219329/)
3.  [操作系统教程](http://book.douban.com/subject/3629787/)
4.  [Linux 系统架构](http://www.cnblogs.com/vamei/archive/2012/09/19/2692452.html)
5.  [打造高效的工作环境 – Shell 篇](https://coolshell.cn/articles/19219.html)
6.  [自学 Linux 命令的四种方法](https://www.cnblogs.com/linuxprobe/p/5580604.html)

* * *

欢迎关注**“rocedu”**微信公众号 (手机上长按二维码)

**做中教，做中学，实践中共同进步！**

![](https://images2015.cnblogs.com/blog/741560/201611/741560-20161128062214709-687903811.jpg)

* * *

-   原文地址：[http://www.cnblogs.com/rocedu/p/4902411.html](http://www.cnblogs.com/rocedu/p/4902411.html)
-   推荐网站：[博客园](http://www.cnblogs.com/rocedu/)、[新浪微博](http://weibo.com/rocedu)、[扇贝背单词](http://www.shanbay.com/referral/ref/c3c06/)、[DKY 背单词小组](http://www.shanbay.com/team/detail/18898/#p1)、[有道云笔记](http://note.youdao.com/web/setting?type=invite)、[豆瓣读书](http://book.douban.com/people/rocflytosky/collect)
-   版权声明：自由转载 - 非商用 - 非衍生 - 保持署名 | [Creative Commons BY-NC-ND 3.0](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)

* * *

如果你觉得本文对你有帮助，请点一下左下角的 “**好文要顶**” 和 “**收藏该文**”

* * *

 [https://www.cnblogs.com/rocedu/p/4902411.html](https://www.cnblogs.com/rocedu/p/4902411.html)
