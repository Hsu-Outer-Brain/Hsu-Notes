# streamlit量化选择etf：可视化界面实现
[streamlit 量化选择 etf：可视化界面实现](https://mp.weixin.qq.com/s?__biz=MzIwNTU2ODMwNg==&mid=2247487548&idx=1&sn=f75694153381ee09735416a58d615474&chksm=972fb061a0583977295af4dd6b691c09c4f902abe0cbdcffefb3a0b2a02ab3df57c2277927c7&token=1863667599&lang=zh_CN#rd) 

 原创文章第 132 篇，专注 “个人成长与财富自由、世界运作的逻辑，AI 量化投资”。

本以为本周的早高峰人会变多，因为有大量的公司要开始复工，结果似乎没有发生。所以，“大家做自己健康的第一责任人”是没有问题的，而且很多店铺自己就把店关了，多数人都自觉戴上了 N95，自觉保持社交距离。——也许这就是 “市场经济” 的力量吧，你越不管，“自觉”的力量就觉醒了。

今天把 etf 的列表可视化出来，人是视觉动物，可视化非常重要。  

看下效果：  

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-12%2019-59-43/1e982c03-5d92-4f07-9dea-219f6ddbb8b1.png?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-12%2019-59-43/358e00b3-4133-43bd-96e9-f9096d2f1488.png?raw=true)

01 从 tushare 下载场内基金基本信息  

A 股场内基金一共 1700 多支（比场外基金少多了），退市的 500 多支，**可交易的 1205 支**。  

\# ================ basic =========================  
def download_etfs_basic():  
    # 拉取数据  
  df = pro.fund_basic(\*\*{  
        "ts_code":"",  
  "market": "E",  
  "update_flag":"",  
  "offset": "",  
  "limit": "",  
  "status": "L",  
  "name": ""  
  }, fields\\=\[  
        "ts_code",  
  "name",  
  "management",  
  "custodian",  
  "fund_type",  
  "found_date",  
  "due_date",  
  "list_date",  
  "issue_date",  
  "delist_date",  
  "issue_amount",  
  "m_fee",  
  "c_fee",  
  "duration_year",  
  "p_value",  
  "min_amount",  
  "exp_return",  
  "benchmark",  
  "status",  
  "invest_type",  
  "type",  
  "trustee",  
  "purc_startdate",  
  "redm_startdate",  
  "market"  
  ])  
    df.rename(columns\\={'ts_code':'code'}, inplace\\=True)  
    df.sort_values(by\\='list_date', inplace\\=True)  
    df.set_index('code', inplace\\=True)  
    return df  

调用的代码，写入 hdf5:

from engine.config import DATA_DIR_HDF5_ALL, DATA_DIR_HDF5_BASIC  
basic = download_etfs_basic()  
print(basic)

with pd.HDFStore(DATA_DIR_HDF5_BASIC.resolve()) as store:  
    store\['etfs'] = download_etfs_basic()

结果显示：（最早的 etf，上市时间是 2004 年底）

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-12%2019-59-43/c2949b72-9655-49d8-ad1c-ad7f6b669e63.png?raw=true)

**02 streamlit 读出并查询 etfs**

@staticmethod  
def load_etfs_basic():  
    with pd.HDFStore(DATA_DIR_HDF5_BASIC.resolve()) as store:  
        return store\['etfs']

**03 界面布局**  

col_indicators, col_corr = st.columns(2)  
col_indicators.subheader('风险收益指标')  
col_indicators.table(DataUtils.calc_indicators(df_returns))

col_corr.subheader('相关系数')  
col_corr.table(df_returns.corr())

核心代码就是 st.columns(N)，即可以横向布局：  

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-12%2019-59-43/8e7f3fab-d6f3-48d4-8ff5-0455061fae69.png?raw=true)

04 etf 筛选  

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-12%2019-59-43/59135e10-1352-43a0-b162-362ecfdce4d5.png?raw=true)

这里使用了 streamlit 的多页面：  

只需要在 main.py 的同级目录下，创建一个 pages，然后里面的 xx.py 都会自动生成一个页面：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-12-12%2019-59-43/550e71e7-767c-4003-8637-df065ef0ab4a.png?raw=true)

多级联动也非常简单，主要使用 pandas 的 query 查询语法：

import streamlit as st  
from engine.data_utils import DataUtils  
from engine.logic import Logic

st.header('etf 分析')

@st.cache  
def load_etfs():  
    return Logic.load_etfs_basic()

col_fund, col2_invest = st.columns(2)  
# 基金类型  
df = load_etfs()  
total = len(df)  
fund_type = col_fund.selectbox('基金类型：', options\\=df\['fund_type'].unique())

df = df.query("fund_type == '{}'".format(fund_type))  
invest_type = col2_invest.selectbox('投资类型：', options\\=df\['invest_type'].unique())  
df = df.query("invest_type == '{}'".format(invest_type, fund_type))

col1, col2 = st.columns(2)  
col1.metric("场内基金总数", total)  
col2.metric("当前数量", len(df))  
st.table(df)

streamlit 特别适合数据驱动型，机器学习型的项目，尤其是用于演示类的需求。后续可以把 etf 的一些指标计算比如，比如 20 日动量等，即可以支持指标选基金。

**代码和数据已经上传至星球 - 量化专栏。** 

近期文章：

[多策略风险收益对比——风险平价，动量以及 RSRS](http://mp.weixin.qq.com/s?__biz=MzIwNTU2ODMwNg==&mid=2247487537&idx=1&sn=6f37beaf5eb686b48a79c974bdf69efd&chksm=972fb06ca058397afabd114d04c6ca5bdb12a90125ac81ae3169c7cd6f6e882e94ba391606b1&scene=21#wechat_redirect)  

[wxpython+streamlit：AI 量化平台更新 gui](http://mp.weixin.qq.com/s?__biz=MzIwNTU2ODMwNg==&mid=2247487530&idx=1&sn=b2dabd31887def7726f2f25fbd9c1f38&chksm=972fb077a05839612506122d165ff0f5c9deb59c8f7394b07720e47f7422b1cf36e2e86c5205&scene=21#wechat_redirect)
