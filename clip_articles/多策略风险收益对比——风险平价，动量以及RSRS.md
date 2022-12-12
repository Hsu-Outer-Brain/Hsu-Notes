# 多策略风险收益对比——风险平价，动量以及RSRS
[多策略风险收益对比——风险平价，动量以及 RSRS](https://mp.weixin.qq.com/s?__biz=MzIwNTU2ODMwNg==&mid=2247487537&idx=1&sn=6f37beaf5eb686b48a79c974bdf69efd&chksm=972fb06ca058397afabd114d04c6ca5bdb12a90125ac81ae3169c7cd6f6e882e94ba391606b1&token=1863667599&lang=zh_CN#rd) 

 原创文章第 131 篇，专注 “个人成长与财富自由、世界运作的逻辑，AI 量化投资”。

昨天使用 streamlit 更新的 gui，效果确实不错，比起纯用 wxpython 开发要省了不是一星半点的工作量，而且功能强大，更别提 pyqt5 了——功能虽强，但要做好这个界面，更得花费不少心思——而我们核心是要做出真正可以实盘使用的策略，gui 只是方便我们做策略的工具，不可本末倒置。

关于疫情，无论是班级群，公司群，朋友圈，都陆续爆阳了，好像多多少少都有点症状，可能无症状的同学自己还不知道吧。现在核酸混管基本不可用的，到处都混，早上还去做了一个，明天能否得到结果未知，全凭运气了。  

都是早晚的事，面对这样一个群体事件——战略上藐视，战术上重视，而不是反过来。战略上，这就是一个流感，不用担心到无法正常生活与工作；战术上，戴好口罩，勤洗手，保持距离，如此而已。能不得就不得，能少得就少得，但心态要放松，积极乐观。

今天继续 streamlit，昨天是可以把多个指数进行风险收益分析，今天把回测结果纳入进来做对比分析。  

回测的结果保存在 data/hdf5/bkt_results.h5 里。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-12%2019-22-43/851550e2-22a9-4042-8486-800e0551569d.png?raw=true)

使用 pd 把 keys 读出来，就是各个策略的名字。

@staticmethod  
def get_bkt_results():  
    with pd.HDFStore(DATA_DIR_HDF5_BKT_RESULTS.resolve()) as store:  
        keys = \[k.replace('/', '') for k in store.keys()]  
    return keys

这样策略与多个基准可以搁在一起进行比较，风险，收益分析，这个很重要。但很多量化平台其实都做不到，通过只能指定一个 benchmark。  

正股债平衡策略的话，我即想跟股票指数比，与想跑债券指数比，这样才能看得出投资组合的优势嘛。

st.sidebar.header('请选择策略')  
stras = st.sidebar.multiselect('请选择策略：', options\\=Logic.get_bkt_results(), default\\=Logic.get_bkt_results())  

indexes = Logic.get_index_list()  
df_returns = DataUtils.load_returns2(symbols)  
df_stras = DataUtils.load_strategy_returns(stras)  
df_returns = pd.concat(\[df_returns, df_stras], axis\\=1)  
df_returns.dropna(inplace\\=True)  
df_returns = df_returns\[df_returns.index >= date_start]  
df_returns = df_returns\[df_returns.index &lt;= date_end]  
df_equity = (1 + df_returns).cumprod()  

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-12%2019-22-43/b64b228c-2f49-421d-93ac-e66d22a4b128.png?raw=true)

不得不多说一句，streamlit 真的很赞，面向数据驱动与机器学习类的应用，非常省事，而且功能强大。要知道无论是 wxpython 还是 web，要写一个时间日期控件，非常麻烦，更别提这个多选的下拉框——在 streamlit 里是内置的，而且不用绑定事件，streamlit 是流式执行。  

下一步的重点，就是把风险平价与因子模型要有机融合在一起，然后再看行业轮动增强收益。  

昨天还有朋友问我，有没有适合私募的，可以复利积累的模型，我现在就在探索和强化这个，昨天我们说了，10% 年化，回撤控制在 10% 以内的模型我们已经有了，现在要努力 “双 15” 或者 “双 20%” 的模型。

代码及数据请前往星球——量化专栏下载。

最近文章：  

[wxpython+streamlit：AI 量化平台更新 gui](http://mp.weixin.qq.com/s?__biz=MzIwNTU2ODMwNg==&mid=2247487530&idx=1&sn=b2dabd31887def7726f2f25fbd9c1f38&chksm=972fb077a05839612506122d165ff0f5c9deb59c8f7394b07720e47f7422b1cf36e2e86c5205&scene=21#wechat_redirect)  

[七年财务自由之路的逻辑](http://mp.weixin.qq.com/s?__biz=MzIwNTU2ODMwNg==&mid=2247487494&idx=1&sn=dfaa07f7a3a5a7e0648573f5e95cbf1f&chksm=972fb05ba058394d82d5aeff8c1fdda3e81307a88bfcef428a13a35e900582e479183b5c0585&scene=21#wechat_redirect)  

[大类资产配置——PCA 风险平价，年化 11%](http://mp.weixin.qq.com/s?__biz=MzIwNTU2ODMwNg==&mid=2247487488&idx=1&sn=f848083ff986860fea5033b63008cdc6&chksm=972fb05da058394bbd2eb6275034229a8af656c90fdc703a140163dfd74dd20b61a042563da4&scene=21#wechat_redirect)  

[ETF 轮动 + RSRS 择时，加上卡曼滤波：年化 48.41%，夏普比 1.89](http://mp.weixin.qq.com/s?__biz=MzIwNTU2ODMwNg==&mid=2247487385&idx=1&sn=d050e6d0a03147ebc57850fdc76ec026&chksm=972fafc4a05826d239985c5297df5906c77e37519e3b33663ef31b9eac0d3f25799be68673b2&scene=21#wechat_redirect)
