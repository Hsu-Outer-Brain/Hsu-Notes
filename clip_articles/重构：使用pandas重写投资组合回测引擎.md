# 重构：使用pandas重写投资组合回测引擎
[重构：使用 pandas 重写投资组合回测引擎](https://mp.weixin.qq.com/s?__biz=MzIwNTU2ODMwNg==&mid=2247487501&idx=1&sn=f8f355f8cb2950c2056037fb360b9f5c&chksm=972fb050a0583946cde5969c49991515f407df59137461756f8cfa92adeb62413f5f3374b925&token=265388920&lang=zh_CN#rd) 

 原创文章第 128 篇，专注 “个人成长与财富自由、世界运作的逻辑，AI 量化投资”。

我们已经做了一个**“双十”，长期年化 10%，最大回撤率小于 10% 的模型**。现在我们的目标是做一个**“双二十”（长期年化 20%，最大回撤率控制在 20% 以内）**。——不要说这很容易，其实非常难，注意是长期年化 20%。  

**01 重构回测引擎**

当前我们使用的回测引擎是 backtrader，在引入了 algo，表达式引擎之后，我们没有使用 bt 的指标体系，也没有使用 bt 的可视化，使用的是 quantstats，这样的话，bt 在我们的 engine 里只起到一个事件驱动，下单。同时，我们仅使用了 order_target_pencent 这样函数对仓位重新进行分配。——而且对于 order_target_pencent 源码是比较不容易看清楚。

基于此，我们使用纯 pandas 的代码重新 “造” 一个轮子。  

从极简的角度，我们不考虑什么 order， broker，exchange 所，也不需要事件事驱，但走的是按 bar 特征——也不是向量化。

import pandas as pd  
from collections import defaultdict

class Account:  
    def \_\_init\_\_(self, init_cash=100000.0):  
        self.init_cash = init_cash  # 初始资金 self.curr_cash = self.init_cash  # 当前现金 self.curr_holding = defaultdict(float)  # 当前持仓 {symbol: 市值}self.cache_dates = \[]  # 日期序列 self.cache_portfolio_mv = \[]  # 每日市值序列

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-12%2016-35-07/ffef3639-733c-4114-84d2-57cd6739784e.png?raw=true)

**02 update_bar**

update_bar 根据每天持仓情况，更新市值。  

重点是这个 se_bar 的结构，多个 symbol 的同一个 date 的收益率。

\# 当日收盘后，要根据 se_bar 更新一次及市值，再进行交易——次日开盘交易（这里有滑点）。  
def update_bar(self, date, se_bar):

    \# 所有持仓的，按收益率更新mv total\_mv = 0.0 \# 当前已经持仓中标的，使用收盘后的收益率更新 for s, mv in self.curr\_holding.items():  
        rate = 0.0 \# 这里不同市场，比如海外市场，可能不存在的，不存在变化率就是0.0， 即不变 if s in se\_bar.index:  
            rate = se\_bar\[s\]  
        new\_mv = mv \* (1 \+ rate)  
        self.curr\_holding\[s\] = new\_mv  
        total\_mv += new\_mv

    self.cache\_portfolio\_mv.append(total\_mv + self.curr\_cash)  
    self.cache\_dates.append(date)

**03 调仓函数**

调仓的参数是日期，以及权重\[(symbol,w)...]。weights 的加和 &lt;=1。

如果 len(weights==0) 全部持仓，转为 cash。

\# weights 之和需要 &lt;=1，空仓就是 cash:1，只调整 curr_holding/cash 两个变量  
def adjust_weights(self, date, weights: dict): # weights:{symbol:weight} if len(weights.items()) &lt;= 0:  
        logger.info('权重长度为 0，清仓')  
        self.\_close_all()  
        return# 当前持仓总市值 + 现金部分 = 组合市值 total_mv = self.\_calc_total_holding_mv()  
    total_mv += self.curr_cash

    old\_pos \= self.curr\_holding.copy()  
    self.curr\_holding.clear()  
    for s, w in weights.items():  
        self.curr\_holding\[s\] = total\_mv \* w  
    self.curr\_cash = total\_mv - self.\_calc\_total\_holding\_mv()

\# 持仓市值，不包括 cash  
def \_calc_total_holding_mv(self):  
    total_mv = 0.0 for s, mv in self.curr_holding.items():  
        total_mv += mv  
    return total_mv

def \_close_all(self):  
    self.curr_cash += self.\_calc_total_holding_mv()  
    self.curr_holding.clear()

**04 加载收益率数据**  

dataloader 之前已经介绍过，使用 datafeed_hdf5，这两块代码没有太大变化。

from engine.datafeed.dataloader import Dataloader

class DataUtils:  
    @staticmethod def load_returns(symbols):  
        names = \['return']  
        fields = \['$close/Ref($close,1)-1']  
        df_returns = Dataloader().load_one_df(symbols, names, fields)  
        df_returns = df_returns\[\['return', 'code']]  
        df_returns.dropna(inplace\\=True)  
        return df_returns

05  引擎主体  

这里考虑到要兼容强化学习环境，把 step 独立出来——要知道 backtrader 要实现这个有多么麻烦。

import pandas as pd

from engine.account import Account  
from engine.data_utils import DataUtils

class Engine:  
    def \_\_init\_\_(self, symbols, init_cash=100000):  
        self.symbols = symbols  
        self.df_returns = DataUtils.load_returns(symbols)  
        self.acc = Account(init_cash\\=init_cash)

        \# 从feed得到df，并从index取出日期列表  

  self.dates = self.df_returns.index.unique()

    def run(self, algo\_list):  
        self.algo\_list = algo\_list  
        for index, date in enumerate(self.dates):  
            self.step(index, date)

    def step(self, index, date):  
        return\_se = self.\_get\_curr\_return\_se(date)  
        self.acc.update\_bar(date, return\_se)  
        self.algo\_processor()

    def algo\_processor(self):  
        context = {'engine': self, 'acc': self.acc}  
        for algo in self.algo\_list:  
            if algo(context) is True:  \# 如果algo返回True,直接不运行，本次不调仓  

  return None  

 def \_get_curr_return_se(self, date):  
        df_bar = self.df_returns.loc\[date]  
        if type(df_bar) is pd.Series:  
            df_bar = df_bar.to_frame().T  
        df_bar.set_index('code', inplace\\=True)  
        return df_bar\['return']

我们的引擎不需要单独的 strategy，使用使用 “算子”，这个是我们的引擎最有优势的地方。  

if \_\_name\_\_ == '\_\_main\_\_':  
    symbols = \['SPX', '000300.SH']  
    from engine.algo.algos import RunOnce, SelectAll, WeightEqually  
    e = Engine(symbols)  
    se = e.\_get_curr_return_se('2022-12-02')  
    print(se)  
    print('初始总市值', e.acc.get_total_mv())  
    e.run(algo_list\\=\[  
        RunOnce(),  
  SelectAll(),  
  WeightEqually()  
    ])  
    print('最终总市值', e.acc.get_total_mv())

一个**核心功能齐全的投资组合管理的回测引擎**就完成了。  

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-12%2016-35-07/2fc09a3c-c644-437a-ac2b-695cb517bdf4.png?raw=true)

**代码请前往星球下载。** 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-12%2016-35-07/6043f3d3-8f9d-4106-83af-aa37b7ceb19b.png?raw=true)

其他：

关于防疫优化，确实进展很快，“走小步，不停步”。尽管我做了两天核酸，依然没有拿到结果，今天做了 3 回，若还没结果就算了。现在倒也基本不需要使用，堂食可能是一个点，但也没有那么严格的执行，还有外卖这个替代方案。

今天与一家银行聊量化投资的课程，我比较喜欢这样的模式，大家商量着来。我比较 “反感” 合同制的，比较不喜欢看合同，各种条款，事情还没有做，把所有的东西都限制了。也许很多条款会有人真去计较，但总感觉限制发挥。做一件事，本身兴趣很重要。去年婉拒一家线上培训机构就是在看了合同之后——嫌麻烦。本身就是一个授权与合作，但要聊如此多版权的细节，就有点不自在。——不知道这算不算一种 “舒适区” 呢？要不要走出舒适区？

代码和数据请前往星球下载。

[七年财务自由之路的逻辑](http://mp.weixin.qq.com/s?__biz=MzIwNTU2ODMwNg==&mid=2247487494&idx=1&sn=dfaa07f7a3a5a7e0648573f5e95cbf1f&chksm=972fb05ba058394d82d5aeff8c1fdda3e81307a88bfcef428a13a35e900582e479183b5c0585&scene=21#wechat_redirect)  

[大类资产配置——PCA 风险平价，年化 11%](http://mp.weixin.qq.com/s?__biz=MzIwNTU2ODMwNg==&mid=2247487488&idx=1&sn=f848083ff986860fea5033b63008cdc6&chksm=972fb05da058394bbd2eb6275034229a8af656c90fdc703a140163dfd74dd20b61a042563da4&scene=21#wechat_redirect)
