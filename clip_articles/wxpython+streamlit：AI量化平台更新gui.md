# wxpython+streamlit：AI量化平台更新gui
[wxpython+streamlit：AI 量化平台更新 gui](https://mp.weixin.qq.com/s?__biz=MzIwNTU2ODMwNg==&mid=2247487530&idx=1&sn=b2dabd31887def7726f2f25fbd9c1f38&chksm=972fb077a05839612506122d165ff0f5c9deb59c8f7394b07720e47f7422b1cf36e2e86c5205&token=265388920&lang=zh_CN#rd) 

 原创文章第 130 篇，专注 “个人成长与财富自由、世界运作的逻辑，AI 量化投资”。

继续 AI 量化投资系统建设，回测引擎是肯定是基础，我们基本完成了。

在 Analyzer 里，加一个回测结果保存，这样可以与其它结果进行对比分析。  

def save_result(self):  
    from engine.config import DATA_DIR_HDF5_BKT_RESULTS  
    with pd.HDFStore(DATA_DIR_HDF5_BKT_RESULTS.resolve()) as store:  
        store\[self.engine.name] = self.df_results

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-12%2018-50-49/796c4e0a-589f-4499-b01f-38646d7ca3c7.png?raw=true)

单策略回测基本就完成了。我们与基准对比，与其它策略对比，其实就是回测结果的对比，不必耦合到回测过程中。  

今天来搭建一个 gui，为了显示回测结果，对比分析。有 gui 会方便很多。  

结果分析，其实就是单时间序列或者多时间序列分析。  

使用机器学习与数据分析的可视化框架：streamlit。

直接使用 pip install streamlit 即可。  

from datetime import datetime

from engine.data_utils import DataUtils

import pandas as pd  
import numpy as np  
from engine.logic import Logic  
indexes = Logic.get_index_list()

import streamlit as st

title = '时间序列分析'  
st.set_page_config(page_title\\=title, page_icon\\=":bar_chart:", layout\\="wide")  
st.title(title)

date_start = st.sidebar.date_input('起始日期：', datetime(2010, 1, 1))  
date_start = date_start.strftime('%Y-%m-%d')  
date_end = st.sidebar.date_input('结束日期：', datetime.now().date())

st.sidebar.header("请选择:")  
symbols = st.sidebar.multiselect(  
    "选择指数:",  
  options\\=indexes,  
  default\\=indexes\[0]  
)

indexes = Logic.get_index_list()  
df_returns = DataUtils.load_returns2(symbols)  
df_equity = (1 + df_returns).cumprod()

\# df_equity = df_equity\[df_equity.index >= datetime(2010,1,1)]  
# print(df_equity)  

st.subheader('风险收益指标')  
st.table(DataUtils.calc_indicators(df_returns))

st.subheader('折线图')  
st.line_chart(df_equity)  
st.subheader('相关系数')  
st.table(df_returns.corr())

用 python+steamlit 开发这种数据驱动，可视化的界面特别简单，不需要传统 wxpython 这种要加事件回调等，无论是折线图，还是表格显示，都是 st.table(df)，st.line_chart(df) 这样的操作，着实帮我们省了不少事情。  

风险收益指标的计算，我们使用 quantopian 的 empyrical 的库，pyfolio 也是使用这个库进行计算的。

@staticmethod  
def calc_indicators(df_returns):  
    import empyrical  
    accu_returns = empyrical.cum_returns_final(df_returns)  
    accu_returns.name = '累计收益'  
  annu_returns = empyrical.annual_return(df_returns)  
    annu_returns.name = '年化收益'  
  max_drawdown = empyrical.max_drawdown(df_returns)  
    max_drawdown.name = '最大回撤'  
  sharpe = empyrical.sharpe_ratio(df_returns)  
    sharpe = pd.Series(sharpe)  
    sharpe.name = '夏普比'  
  max_drawdown.index = annu_returns.index

    sharpe = pd.Series(empyrical.sharpe\_ratio(df\_returns))  
    sharpe.index = accu\_returns.index  
    sharpe.name = '夏普比'  

  all = pd.concat(\[accu_returns, annu_returns, max_drawdown, sharpe], axis\\=1)

    return all

使用 wxpython 的浏览器做壳：

import wx  
from gui.mainframe import MainFrame  
import subprocess

p_restart = subprocess.Popen(\['streamlit', 'run', 'main_streamlit.py'])  
if \_\_name\_\_ == '\_\_main\_\_':  
    app = wx.App()  
    frm = MainFrame(None, title\\='AI 智能量化投研平台')  
    frm.Show()  
    app.MainLoop()  
    p_restart.kill()

代码与数据已经上传至星球——量化专栏。  

wxpython+streamlit，功能强大，很适合量化。后续因子分析，策略分析都会使用这个模式。  

[从零实现量化回测引擎：风险平价、动量策略对比](http://mp.weixin.qq.com/s?__biz=MzIwNTU2ODMwNg==&mid=2247487522&idx=1&sn=138abe3812673328a9b191967dab6a99&chksm=972fb07fa05839698869edb6f9343a9bd610e6fc860a524266b324af6ecd737d34254b61f4a3&scene=21#wechat_redirect)  

[大类资产配置——PCA 风险平价，年化 11%](http://mp.weixin.qq.com/s?__biz=MzIwNTU2ODMwNg==&mid=2247487488&idx=1&sn=f848083ff986860fea5033b63008cdc6&chksm=972fb05da058394bbd2eb6275034229a8af656c90fdc703a140163dfd74dd20b61a042563da4&scene=21#wechat_redirect)  

[七年财务自由之路的逻辑](http://mp.weixin.qq.com/s?__biz=MzIwNTU2ODMwNg==&mid=2247487494&idx=1&sn=dfaa07f7a3a5a7e0648573f5e95cbf1f&chksm=972fb05ba058394d82d5aeff8c1fdda3e81307a88bfcef428a13a35e900582e479183b5c0585&scene=21#wechat_redirect)
