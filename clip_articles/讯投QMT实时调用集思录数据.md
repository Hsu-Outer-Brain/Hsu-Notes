# 讯投QMT实时调用集思录数据
在 QMT 自带的文档里面，实在找不到任何的溢价率的数据，连转股价也没有，只有光秃秃的一个价格。所以在可转债多因子量化交易里面实在无法进行下去。

不过好在 QMT 支持第三方库，并且也可以连通外部数据，不像 Ptrade 那样封闭（Ptrade 里面 os 这个内置库都被阉割了，更别说访问外部数据），所以笔者就写了一个实时访问集思录数据的接口，供 QMT 访问。

使用 flask 做接口是最简单，可是 flask 性能非常低下，故使用异步框架 uvicorn +asgi。

![](https://article-images.zsxq.com/Fs24JX-JY6aP5vrG_8b7o3yqlGiK)

返回了 383 个转债数据，只要集思录上有的，都可以获取到 QMT 里面。

在 QMT 里面的调用函数就 8 行：

![](https://article-images.zsxq.com/Fszc0mmmYOuABQQY4ZMggeiUTkoN)

其主要核心是之前的文章里面登录并获取集思录数据。然后套一个 web 接口调用即可。

而这里也把之前集思录密码加密部分改为自己使用 AES 加密，省去了 JS 执行的流程，简化了运行流程，提升了效率。

![](https://article-images.zsxq.com/FpgB93stxb1825IaPAVOKtl4azQo)

每次请求大约耗时需要 0.8~0.9 秒左右。

![](https://article-images.zsxq.com/FqwpqWCuMxrpIOGvmXsoaGO20BWu)

运行方式：

先把依赖安装好， pip install -r requirements.txt

在文件 userinfo.py 里面填入你的集思录用户名和密码。

然后 python app.py

就在后台运行了，不要关闭。

然后用浏览器打开 [http://127.0.0.1:8080/jisilu](http://127.0.0.1:8080/jisilu) 如果有数据就说明成功了。

QMT 部分的代码：

```
def get\_jisilu\_data():
	try:
		r = requests.get('http://127.0.0.1:8080/jisilu')
	except Exception as e:
		print(e)
		return \[\]
	else:
		return r.json()

```

调用上面本地的接口就可以获取数据了。

PS：提升速度 TIP

第一次运行的时候 cache=False

![](https://article-images.zsxq.com/FrPU2pn5JDIGvbggqJF_UK1Od0Bp)

会保存你的用户名密码加密数据，然后后续可以关闭上面的 python 程序，把上面代码的 cache=False 改为 cache=True, 重新运行，这样速度会得到提升。因为不用每次都做 AES 计算了，因为每次对用户名密码做 AES 运算的结果第一次已经保存下来。

附录：

返回的字段是根据集思录来命名的，如果需要更多字段，可以自行修改：(注意，最新的集思录的字段做了修改，可以参照： [https://docs.qq.com/sheet/DY3FlR05BdlJ2bllX](https://docs.qq.com/sheet/DY3FlR05BdlJ2bllX) 这个文件来取你需要的字段 )

![](https://article-images.zsxq.com/FkTiER6hnhFFq-0GgPN8P2_ZnK2f)

![](https://article-images.zsxq.com/FszcaMsUCs3NypNBuakkvHMi8Gr-) 
 [https://articles.zsxq.com/id_le9gxibsm2w4.html](https://articles.zsxq.com/id_le9gxibsm2w4.html)
