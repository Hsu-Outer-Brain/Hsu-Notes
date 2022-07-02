# QMT使用的汇总&参见问题 & 逆回购，申购新股代码
-   **第一步 先补充历史数据**

![](https://article-images.zsxq.com/Fr0DIe1VRLJ05T9t33qF7Fb4kPXC)

先下载 python 库安装后 重启一下软件！

-   回测 python 请求 1 分钟数据的话，有条数限制，最多拿回来 3w 多条

-   QMT 里面手续费显示不正确，以手机交割单为准。

-   开发不建议在 init 里取数据，会出现取不到的情况。

-   \_推荐使用 get\_\_market\_\_data\_\_ex, 取 k 线是 k 线时间戳，比如一分钟就是按一分钟来标记的，如果是取 tick 数据，就会有 tick 数据的时间戳。1 分钟最后一个 tick 触发的下单函数才会在下个周期的第一个 tick 发送出去

![](https://article-images.zsxq.com/Fou94o95MKyscNng9ehf7XnQnqzm)

-   **\_没成交回报 注意这个 set\_\_account 要搞一下**

![](https://article-images.zsxq.com/FqbOrtBZjSXQpuGurMB4FBAj3Kdi)

![](https://article-images.zsxq.com/FmiJrnBcu-C23iqZbyegawm3zjrd)

-   QQ 邮件的话 qq 的 smtp 服务是独立授权的 得去网页版邮箱获取一个授权码哈 不要用 QQ 密码呢

![](https://article-images.zsxq.com/Fnm3dTOVMrjxnMPtND3b-N_hbIb6)

-   逆回购代码 time='14:57' 自己设定时间即可


```
\# 逆回购代码

SH\_FLAG = True

SZ\_FLAG = False

def initialize(context):

	run\_daily(context, func, time='14:57')

def func(context):

	cash = context.portfolio.cash

	#上海逆回购

	if SH\_FLAG:

		amount = int((cash/100)/1000)\*1000

		order('204001.SS', -amount)

	#深圳逆回购

	if SZ\_FLAG:

		amount = int((cash/100)/10)\*10

		order('131810.SZ', -amount) 

	
def handle\_data(context, data): pass

```

-   新股新债申购的代码


```
passorder(23,1101,accont\_id,stock,5,1,volume,1,ContextInfo)

```

-   全部撤单的代码：


```
obj\_list = get\_trade\_detail\_data(ContextInfo.accid,'stock','order')
for obj in obj\_list:
    if obj.m\_nOrderStatus in \[48,49,50,55\]:
        cancel(obj.m\_strOrderSysID,ContextInfo.accid,'stock',ContextInfo)

```

 [https://articles.zsxq.com/id_7io5za0alcgf.html](https://articles.zsxq.com/id_7io5za0alcgf.html)
