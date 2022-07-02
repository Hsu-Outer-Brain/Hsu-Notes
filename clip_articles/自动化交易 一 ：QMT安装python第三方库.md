# 自动化交易<一>：QMT安装python第三方库
根据之前规划，接下来会写一系列的 QMT 相关的自动化交易的教程。

考虑到星球的搜索功能太弱，所以把教程分得细一些，便于星友根据标题查找。

引用 QMT 帮助文档里面：

> 对于有经验的 Python 开发者来说，平台提供了自行安装第三方库的方式。为

> 了引入额外的第三方库，用户需要做如下一些操作：

>

> 安装前注意事项：

> 三方库的安装有可能会引起系统错误， 建议有经验的用户进行尝试，已经内置

> 的库，如果没有特殊需要请勿随意升级，特别是平台内置的 pandas 库请务

> 必不要升级，其他库请自行尝试；

>

> 安装三方库前，请备份 QMT 安装目录\\bin.x64\\ 目录下的 DLLs 和 Lib 这两个

> 文件夹，以便在安装三方库引起系统错误后，替换回来，恢复系统默认的库所

> 用

安装前，需要先下载 python 库。

![](https://article-images.zsxq.com/Fl-de54AbwdDEm9NyN0A0III8gZa)

这个是 QMT 内置的 python 库。路径用默认的即可。

如果不下载这个库，系统内置的一些示例都是无法跑的。

比如

```
import numpy as np

```

这样的语句都会报错，提示的错误信息是，无法找到这个库。

这个下载过程可能会比较坎坷，官方客服说，在开盘期间下载速度会限速，所以建议同学们最好在收盘期间下载。

星主自己的亲身经历就是白天基本下不下来，一直提示下载失败。 晚上 10 点的时候下载，一路畅通，几台服务器一下子就一起下好了。

下好后可以到设置的目录看一下。

![](https://article-images.zsxq.com/Fgit4T9-aM0undIaEHXRWsyoPjoD)

文件挺多的，大概有 1.5GB。 而第三方的库文件安装在这个目录的 Lib\\site-packages 下面。

#### Lib\\site-packages 目录

![](https://article-images.zsxq.com/FsExcHxf7SRHgj84C7LCpU7Y7ryY)

QMT 内置的库里面包含了挺多实用库，还有和数据库连接的库，pymongo，pymysql，redis，这样就可以直接在 QMT 里面调用自己的数据库数据，并保存 QMT 的数据到自己的数据库。 后面教程会专门介绍数据访问。

还内置了 requests 库，意味着可以实时去爬一些网站的数据，比如集思录，宁稳等。所以在 QMT 的自带数据不满足的情况下，还可以使用外面的数据来补充。

当然即使不内置 requests，也可以自己安装一个。

#### pip 安装第三方库

为了兼容性，需要本地使用 python3.6.8 版本，因为刚刚下载的 QMT python 版本就是 3.6.8.

如果你本地有其他 python 版本的话，建议使用虚拟环境操作。这样不会影响你原有的 python 版本以及已经安装的第三方库。因为一机装多版本的 python 是很正常的操作。

这里建议使用安装 anaconda 来管理你的 python 版本。

如果你的电脑不是经常使用 python 的话，那就直接安装一个 python3.6.8 版本的即可。可以跳过下面这一步骤。

1. 创建一个 python3.6.8 虚拟环境

```
conda create --name qmt python=3.6.8

```

![](https://article-images.zsxq.com/FvXkEBPyryYnZ1eeeclOk8cISwHL)

![](https://article-images.zsxq.com/FlWSMQoVURQWMJmv2CrHpDUAIPBo)

选择 Y

装好后启动虚拟环境的 python

```
activate qmt

```

假设我的安装目录在 D:\\Tool\\QMT\\

那么第三方库需要安装到这里

```
D:\\Tool\\QMT\\bin.x64\\Lib\\site-packages

```

安装命令：以安装 xcsc_tushare 这个库为例

```
pip install xcsc\_tushare --target=D:\\Tool\\QMT\\bin.x64\\Lib\\site-packages

```

等一下很快就安装好了。这里注意，不要加参数 --upgrade ，因为加了会可能把内置的 pandas，numpy 都帮你更新到最新版本。会导致与系统 QMT 内置版本出现兼容性问题，而且，加了 --upgrade 可能会让你安装失败。

![](https://article-images.zsxq.com/Fj_F1xJh0U0D4rCX3RXnehwhvJAi)

#### 测试是否安装成功

使用下面的代码，获取可转债的基本数据。

```
#encoding:gbk
import xcsc\_tushare as xc

xc\_server = 'http://116.128.206.39:7172'
xc\_token\_pro = '' # 注册tushare的token，可使用之前星球分享的token，查看之前的文章或者私聊获取

xc.set\_token(xc\_token\_pro)
pro = xc.pro\_api(env='prd', server=xc\_server)


def init(ContextInfo):
	pass
	
def handlebar(ContextInfo):
	df = pro.cb\_basic(fields="ts\_code,bond\_short\_name,stk\_code,stk\_short\_name,list\_date")
	print(df)

```

勾选本地 python

![](https://article-images.zsxq.com/FhNCUiHVx8DV7VcPv1OqH3XlJ86x)

点击编译，回测。

如果看到输出很多的转债基础数据，那么说明你安装第三方库成功了。

![](https://article-images.zsxq.com/FrQECuhJsH7r7OX7_2WYJz1iJL6z)

可以看到这个接口的数据是所有历史转债的数据，后面可以根据日期做筛选。

因为 QMT 内置的转债数据太少了，所以只能转向第三方的库，甚至集思录这样的站点获取。

### 当前运行的目录

查看内置的 python 运行的目录：

```
def init(ContextInfo):
	print("当前目录",os.getcwd())

```

输出：

```
D:\\Tool\\QMT\\bin.x64

```

所以如果你想让 QMT 读取你本地的文件，比如你准备好的转债代码，你可以准备好一个文件，放在这个目录下，然后 python 就可以直接读取，而不用添加路径。

如：

```
df = pd.read\_excel("可转债代码.xlsx")

```

当然，如果你添加一个全路径，那么你可以读取你电脑任意的文件，

```
df = pd.read\_excel(r"C:\\code\\可转债代码.xlsx")

```

或者你想导出数据：

```
df.to\_excel("持仓数据.xlsx")

```

结合上面的代码，你可以每天收盘把数据导出到本地。

如果有任何问题，欢迎下面评论留言或者群里咨询。如果有开 QMT 的需求，也可以咨询。提供低费率（万一免五, 转债百万分之五，免五）的渠道。 
 [https://articles.zsxq.com/id_cx1axbemf0co.html](https://articles.zsxq.com/id_cx1axbemf0co.html)
