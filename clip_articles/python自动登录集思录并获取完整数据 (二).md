# python自动登录集思录并获取完整数据 (二)
根据前面第一篇文章的操作，打开浏览器，按下 F12， 然后登陆账户。

找到 POST 方法的请求。

![](https://article-images.zsxq.com/FthdvoYEVfMPs612f-sITqVqM6Vj)

点击这个 login_process, 看到里面的登录用户名和密码都被加密密，有个字段是 aes：1， 所以猜测是用的 aes 加密。

![](https://article-images.zsxq.com/Fm0keU9q67Tgx5GMu1tkx9OitH-V)

然后在集思录的源码里面查找 password 相关字符。

![](https://article-images.zsxq.com/Fj_5eMl-9w1G63uwINAYpHFH_UhG)

所以找到 jslencode 这个函数，然后加上公钥 397151C04723421F 进行加密就得到密码了。

这个函数在 JS 代码里面抠出来了， 放在在附件里面。下载下来文件名为 encode_jsl.js, 和下面的代码一起执行即可。

只需要把这个改为你自己的账户密码

jsl_user = '填入你的用户名'

jsl_password = '填入你的密码'

```
\# -\*- coding: utf-8 -\*-
# @File : login.py

import datetime
import time
import pandas as pd
import execjs
import os
import requests

jsl\_user = '填入你的用户名'
jsl\_password = '填入你的密码'

filename = 'encode\_jsl.txt'

path = os.path.dirname(os.path.abspath(\_\_file\_\_))
full\_path = os.path.join(path, filename)

headers = {
    'Host': 'www.jisilu.cn', 'Connection': 'keep-alive', 'Pragma': 'no-cache',
    'Cache-Control': 'no-cache', 'Accept': 'application/json,text/javascript,\*/\*;q=0.01',
    'Origin': 'https://www.jisilu.cn', 'X-Requested-With': 'XMLHttpRequest',
    'User-Agent': 'Mozilla/5.0(WindowsNT6.1;WOW64)AppleWebKit/537.36(KHTML,likeGecko)Chrome/67.0.3396.99Safari/537.36',
    'Content-Type': 'application/x-www-form-urlencoded;charset=UTF-8',
    'Referer': 'https://www.jisilu.cn/login/',
    'Accept-Encoding': 'gzip,deflate,br',
    'Accept-Language': 'zh,en;q=0.9,en-US;q=0.8'
}


def decoder(text): # 加密用户名和密码
    with open(full\_path, 'r', encoding='utf8') as f:
        source = f.read()

    ctx = execjs.compile(source)
    key = '397151C04723421F'
    return ctx.call('jslencode', text, key)


def get\_bond\_info(session): # 获取行情数据
    ts = int(time.time() \* 1000)
    url = 'https://www.jisilu.cn/data/cbnew/cb\_list/?\_\_\_jsl=LST\_\_\_t={}'.format(ts)
    data = {
        "fprice": None,
        "tprice": None,
        "curr\_iss\_amt": None,
        "volume": None,
        "svolume": None,
        "premium\_rt": None,
        "ytm\_rt": None,
        "rating\_cd": None,
        "is\_search": "N",
        "btype": "C",
        "listed": "Y",
        "qflag": "N",
        "sw\_cd": None,
        "bond\_ids": None,
        "rp": 50,
    }

    r = session.post(
        url=url,
        headers=headers,
        data=data
    )
    ret = r.json()
    result = \[\]
    for item in ret\['rows'\]:
        result.append(item\['cell'\])
    return result


def login(user, password): # 登录
    session = requests.Session()
    url = 'https://www.jisilu.cn/account/ajax/login\_process/'
    username = decoder(user)
    jsl\_password = decoder(password)
    data = {
        'return\_url': 'https://www.jisilu.cn/',
        'user\_name': username,
        'password': jsl\_password,
        'net\_auto\_login': '1',
        '\_post\_type': 'ajax',
    }

    js = session.post(
        url=url,
        headers=headers,
        data=data,
    )

    ret = js.json()

    if ret.get('errno') == 1:
        print('登录成功')
        return session
    else:
        print('登录失败')
        raise ValueError('登录失败')


def main(): # 主函数
    today = datetime.datetime.now().strftime('%Y%m%d')
    session = login(jsl\_user, jsl\_password)
    ret = get\_bond\_info(session)
    df = pd.DataFrame(ret)
    try:
        df.to\_excel('jsl\_{}.xlsx'.format(today), encoding='utf8')
    except Exception as e:
        print(e)
    else:
        print('导出excel成功')


if \_\_name\_\_ == '\_\_main\_\_':
    main()


```

运行后同样会生产一个 excel 文件，里面保存的是全量的可转债数据。

打开看看数据，嗯，一个也没少

![](https://article-images.zsxq.com/FizpYOgwUv3kTdxgRISV7u-7LriT)

完结 
 [https://articles.zsxq.com/id_31teufbfq8xz.html](https://articles.zsxq.com/id_31teufbfq8xz.html)
