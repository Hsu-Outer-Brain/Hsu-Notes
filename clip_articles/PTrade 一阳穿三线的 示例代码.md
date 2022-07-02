# PTrade 一阳穿三线的 示例代码
```
LIMIT\_HOLD = 10
import pandas as pd
import numpy as np
from decimal import Decimal
# 初始化模块
def initialize(context):
    #设置比较基准,默认是000300.SS
    set\_benchmark('000300.SS')
    #设置佣金费率,默认是万三佣金费率和5元的最低交易佣金
    set\_commission(commission\_ratio=0.0003, min\_commission =5.0)
    #设置固定滑点,即委托价格与最后的成交价格的价差,默认是0
    set\_fixed\_slippage(fixedslippage=0.2)
    #设置滑点比例,默认为0.1,具体影响还与当时成交数量与该周期最大可成交量有关
    set\_slippage(slippage = 0.2)
    #设置成交比例,默认0.25,即指本周期最大成交数量为本周期市场可成交总量的四分之一
    set\_volume\_ratio(volume\_ratio = 0.2)
    #设置交易股票池
    # g.security = '600570.SS'
    # set\_universe(g.security)
    g.limit\_num = LIMIT\_HOLD

# 盘前处理
def before\_trading\_start(context, data):
    # 今日已委托买入股票池
    g.order\_stocks = \[\]
    # 每天开盘价存储
    g.open\_price\_info = {}
    #获取今日日期
    g.current\_date = context.blotter.current\_dt.strftime("%Y%m%d")
    #获取当日的股票池
    g.stock\_list = get\_Ashares(g.current\_date)
    # g.stock\_list = get\_index\_stocks('000300.XBHS')
    g.stock\_list = list(set(g.stock\_list))
    #获取股票的状态ST、停牌、退市
    st\_status = get\_stock\_status(g.stock\_list, 'ST')
    halt\_status = get\_stock\_status(g.stock\_list, 'HALT')
    delisting\_status = get\_stock\_status(g.stock\_list, 'DELISTING')
    #将三种状态的股票剔除当日的股票池
    for stock in g.stock\_list.copy():
        if st\_status\[stock\] or halt\_status\[stock\] or delisting\_status\[stock\]:
            g.stock\_list.remove(stock)
    #获取股票历史数据
    history = get\_history(70, '1d', \['close','volume'\], g.stock\_list, fq='dypre', include=False)
    history = history.swapaxes("minor\_axis", "items")
    g.close\_df\_dict = {}
    for stock in g.stock\_list.copy():
        #log.info(stock)
        df = history\[stock\]
        #过滤历史停牌的数据    
        df = df\[df\['volume'\]>0\]                    
        #如果非停牌日期>=60,#增加到历史数据存储字典,否则移除
        if len(df.index)<60:   
            g.stock\_list.remove(stock)
            continue
        
        stock\_exrights = get\_stock\_exrights(stock, date=g.current\_date)
        if stock\_exrights is not None:
            # log.info(stock\_exrights)
            # A\_fact = stock\_exrights\['dynamic\_exer\_forward\_a'\].iloc\[0\]
            A\_fact = stock\_exrights\['dynamic\_exer\_forward\_a'\].iloc\[0\]
            # log.info(type(A\_fact))
            B\_fact = stock\_exrights\['dynamic\_exer\_forward\_b'\].iloc\[0\]
            # log.info(A\_fact)
            # log.info(B\_fact)
            df\['price'\] = 0
            df\['price'\] = df.apply(lambda x: x\['close'\]\*A\_fact + B\_fact, axis =1)
            # log.info(df)
            close\_array = df\['price'\].values\[:\]
            g.close\_df\_dict\[stock\] = close\_array
        else:
            close\_array = df\['close'\].values\[:\]
            g.close\_df\_dict\[stock\] = close\_array
    #设置今天的股票池
    set\_universe(g.stock\_list)
    g.count = 0
    
# 盘中运行
def handle\_data(context, data):
    K\_count = get\_K\_count(context)
    if K\_count == 1:
        position\_last\_close\_init(context)
        for stock in g.stock\_list:
            g.open\_price\_info\[stock\] = data\[stock\].open            
    if K\_count != 230:
        return
    g.data = data
    sell\_function(context)
    buy\_function(context)
    
# 盘后处理
def after\_trading\_end(context, data):
    position\_list = \[\]
    for stock in context.portfolio.positions:       
        if context.portfolio.positions\[stock\].amount != 0:
            position\_list.append(stock)
    log.info(('持仓股:', position\_list))
    g.limit\_num = LIMIT\_HOLD - len(position\_list)
    log.info('校准可买股票数量为%s'%g.limit\_num)
    
# 买入模块
def buy\_function(context):
    buy\_stocks = get\_buy\_stocks()
    portfolio\_info = context.portfolio
    cash = portfolio\_info.cash
    #   判断最大持仓数字的全局变量g.limit\_num大于等于1
    if g.limit\_num>=1:
        #   判断成立则通过当前现金与可买入股票数量的余数赋值给平均每股可买变量mean\_cash
        mean\_cash = cash/g.limit\_num
    #   判断不成立返回当前现金
    else:
        mean\_cash = cash
    if buy\_stocks:
        buy\_list = filter\_limitup\_stock(buy\_stocks)
        log.info('最终下单股票池%s'%buy\_list)
        for stock in buy\_stocks:
            # log.info(stock)
            if (stock not in g.position\_last\_map) and (stock not in g.order\_stocks):
                #   判断最大持仓数字的全局变量g.limit\_num大于等于1
                if g.limit\_num >=1:
                    order\_id = order\_value(stock, mean\_cash)
                    #   判断该次买入的股票是否已在持仓信息的dict中
                    if order\_id is not None:
                        # log.info(stock)
                        #   如果下单成功，最大持仓数字的全局变量g.limit\_num减去1
                        g.order\_stocks.append(stock)
                        g.limit\_num-=1
                        close\_array = g.close\_df\_dict.get(stock)
                        log.info(('已下单买入：',stock))
                        # g.buy\_date\[stock\] = g.current\_date     

# 卖出模块
def sell\_function(context):
    for stock in g.position\_last\_map:
        flag = sell\_decision(stock)
        if flag:
            order\_id = order\_target(stock, 0)
            if order\_id is not None:
                g.limit\_num+=1
                log.info(('已下单卖出：',stock))

# 卖出决策
def sell\_decision(stock):
    curr\_price = g.data\[stock\].price
    close\_array = g.close\_df\_dict.get(stock)
    MA10 = get\_MA(10, close\_array, current\_price=curr\_price)
    if curr\_price < MA10:
        return True
    else:
        return False        


  
# 买入股票池决策
def get\_buy\_stocks():
    '''
    一阳穿三线形态个股筛选
    '''
    buy\_list = \[\]
    # log.info('111')
    # log.info(len(g.stock\_list))
    for stock in g.stock\_list.copy():
        
        close\_array = g.close\_df\_dict.get(stock)
        curr\_close = g.data\[stock\].price
        curr\_open = g.open\_price\_info.get(stock)
        ma\_short = get\_MA(5, close\_array, current\_price=curr\_close)
        ma\_mid = get\_MA(20, close\_array, current\_price=curr\_close)
        ma\_long = get\_MA(60, close\_array, current\_price=curr\_close)
        # log.info('stock:%s,close\_array:%s,curr\_close:%s,curr\_open:%s'%(stock,close\_array\[-5:\],curr\_close,curr\_open))
        cond1 = curr\_open<ma\_short and curr\_open<ma\_mid and curr\_open<ma\_long
        if not cond1:
            continue
        cond2 = curr\_close>ma\_short and curr\_close>ma\_mid and curr\_close>ma\_long
        if not cond2:
            continue
        cond3 = ma\_short>ma\_mid and ma\_short>ma\_long
        if not cond3:
            continue
        price\_range = (curr\_close - close\_array\[-1\])/close\_array\[-1\]
        if 0.02 < price\_range < 0.05:
            buy\_list.append(stock)
    return buy\_list

#保留小数点两位
def replace(x):
    y = Decimal(x)
    y = float(str(round(x, 2)))  
    return y    
    
#剔除涨停
def filter\_limitup\_stock(stocks):
    stock\_list = stocks.copy()
    for stock in stock\_list.copy():
        last\_close = g.close\_df\_dict.get(stock)\[-1\]
        curr\_price = g.data\[stock\].price
        # log.info('222')
        # log.info(last\_close)
        limitup\_price = last\_close\*1.1
        # log.info(limitup\_price)
        limitup\_price = replace(limitup\_price)
        if curr\_price >= limitup\_price:
            stock\_list.remove(stock)
    return stock\_list
    

#获取均线方法
def get\_MA(days, close\_array, current\_price=None):
    if current\_price is None:
        MA = mean(close\_array\[-days:\])
    else:
        MA = (close\_array\[-days:-1\].sum()+current\_price)/days
    return MA
                

   
#获取最新分钟K线数
def get\_K\_count(context):
    date = str(context.blotter.current\_dt)
    hour = int(date\[11:13\])
    minute = int(date\[14:16\])    
    #log.info(date)
    #log.info(hour)
    #log.info(minute)
    min\_num = 0
    if hour == 1:
        min\_num = minute-29
    elif hour == 2:
        min\_num = minute + 31
    elif hour == 3:
        min\_num = minute + 91
    elif hour == 5:
        min\_num = minute + 121
    elif hour == 6:
        min\_num = minute + 181
    min\_num -= 1
    return min\_num
    
#生成昨日持仓股票列表
def position\_last\_close\_init(context):
    g.position\_last\_map = \[
        position.sid
        for position in context.portfolio.positions.values()
        if position.amount != 0
    \]

```

 [https://articles.zsxq.com/id_1ffvwad5fhww.html](https://articles.zsxq.com/id_1ffvwad5fhww.html)
