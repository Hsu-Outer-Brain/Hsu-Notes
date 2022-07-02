# 一条语句快速导入网页表格到 python dataframe
一条语句把网页中的数据导入 pandas.

不需要爬取数据, 不需要保存文件.

快速把网页中的类似表格的数据导入为 dataframe 格式

以宁稳网为例:

先选中你要获取的数据

![](https://article-images.zsxq.com/FkWk96MpgXFQay1SMMDRqEsOj6DJ)

以宁稳网为例,

然后我用鼠标选中前面 6 条, 然后按下 ctrl + c 复制

然后开一个 python 窗口, jupyter notebook 或者优矿, ipython, 交互式都可以.

运行一条命令

```
df = pd.read\_clipboard()

```

按下 enter 后, 然后 df.head() 查看下数据

![](https://article-images.zsxq.com/Fhm765H2p5A2LZu-E31WhlVburK2) 
 [https://articles.zsxq.com/id_r9wjw5846k29.html](https://articles.zsxq.com/id_r9wjw5846k29.html)
