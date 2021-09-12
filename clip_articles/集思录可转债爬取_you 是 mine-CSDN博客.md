# 集思录可转债爬取_you 是 mine-CSDN博客
```python


import json
import requests
import csv
import re
from lxml import etree

def get_dat():
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/51.0.2704.103 Safari/537.36",
    }

    newUrl ="https://www.jisilu.cn/data/cbnew/cb_list/?___jsl=LST___t=1584777951900"
    
    response = requests.get(newUrl)
    
    data = response.content.decode("utf-8")
    dat = json.loads(data)

    
    lst_data = []
    for one in dat['rows']:
        
        lst_dat = []
        
        id = one["id"]
        dat_cell = one["cell"]
        
        is_shui = dat_cell['force_redeem']
        if is_shui == None:
            
            name = dat_cell['bond_nm']
            
            price = dat_cell['price']
            
            premium_rt = dat_cell['premium_rt']
            
            rating_cd = dat_cell['rating_cd']
            
            put_convert_price = dat_cell['put_convert_price']
            
            force_redeem_price = dat_cell['force_redeem_price']
            
            last_time = dat_cell['year_left']
            
            dblow = dat_cell['dblow']

            
            xiangqing_url = 'https://www.jisilu.cn/data/convert_bond_detail/' + id
            xiangqing_response = requests.get(xiangqing_url)
            html = xiangqing_response.content.decode("utf-8")
            html = etree.HTML(html)
            lixi = html.xpath('.//td[@id="cpn_desc"]/text()')
            pattern = re.compile(r'\d+\.\d+?')  
            lixi = pattern.findall(lixi[0])
            shuhuijia = html.xpath('.//td[@id="redeem_price"]/text()')
            li_price = 0
            for li in lixi:
                li_price = li_price + float(li)
            try:
                jiancang = float(shuhuijia[0]) + (li_price - float(lixi[-1])) * 0.8
            except:
                jiancang = 0

            
            if jiancang != 0 and float(price) - jiancang < 3:
                is_oper = '建仓'
                if jiancang != 0 and float(price) - float(shuhuijia[0]) < 3:
                    is_oper = '加仓'
            else:
                is_oper = ''
            lst_dat.append(id)
            lst_dat.append(name)
            lst_dat.append(price)
            lst_dat.append(jiancang)
            lst_dat.append(shuhuijia[0])
            lst_dat.append('')
            lst_dat.append(premium_rt)
            lst_dat.append(rating_cd)
            lst_dat.append(put_convert_price)
            lst_dat.append(force_redeem_price)
            lst_dat.append(last_time)
            lst_dat.append(dblow)
            lst_dat.append(is_oper)
            lst_data.append(lst_dat)
        else:
            continue
    return lst_data

def wirte_csv(data):
    
    f = open('可转债.csv', 'w', encoding='utf-8', newline='')
    
    csv_writer = csv.writer(f)
    
    csv_writer.writerow(["代 码", "转债名称", "现 价", "建仓线", "加仓线", "重仓线", "溢价率", "评级",
                     "回售触发价", "强赎触发价", "剩余年限", "双低", "操作"])
    
    for dat in data:
        csv_writer.writerow(dat)
    
    f.close()


if __name__ == '__main__':
    data = get_dat()
    wirte_csv(data)


```

![](https://img-blog.csdnimg.cn/20200326161243388.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4OTQ5ODQ3,size_16,color_FFFFFF,t_70) 
 [https://blog.csdn.net/qq_28949847/article/details/105121043](https://blog.csdn.net/qq_28949847/article/details/105121043)
