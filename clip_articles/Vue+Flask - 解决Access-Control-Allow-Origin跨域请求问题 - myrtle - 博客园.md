# Vue+Flask - 解决Access-Control-Allow-Origin跨域请求问题 - myrtle - 博客园
### 首先上图：

Fn+F12 或者 F12，选择 Network 再查看 Headers

![](https://img2020.cnblogs.com/blog/1748669/202103/1748669-20210329012430997-2109829885.png)

 ![](https://img2020.cnblogs.com/blog/1748669/202103/1748669-20210329012450294-768881146.png)

 ![](https://img2020.cnblogs.com/blog/1748669/202103/1748669-20210329012532132-1704627818.png)

 查阅了大量 blog 后解决了 vue 前端跨域问题，status code 变成了 200ok，但是 response 仍然没有数据，才发现后端也要解决跨域问题（我麻了呀）

![](https://img2020.cnblogs.com/blog/1748669/202103/1748669-20210329012758635-1136207737.png)

![](https://img2020.cnblogs.com/blog/1748669/202103/1748669-20210329012821029-1481962189.png)

### Vue 前端解决跨域问题

附上代码：

.env.development 文件

VUE_APP_BASE_API = '/dev-api'

vue.config.js 文件，修改 devServer:

![](https://common.cnblogs.com/images/copycode.gif)

devServer: {
    port: port,
    open: true,
    overlay: {
      warnings: false,
      errors: true },
    proxy: {
      \["/dev-api"]:{
            target:'[http://127.0.0.1:5000'](http://127.0.0.1:5000'),
              changeOrigin:true,
                pathRewrite: {
                    \['^' + "/dev-ap"]: '' }
            }
    },
    after: require('./mock/mock-server.js')
  },

![](https://common.cnblogs.com/images/copycode.gif)

查询时的 url 设置成'/teacher/edit'就好了，不需要添加 / dev-api，在 pathRewrite 中已经抵消了

修改后查询的链接会自动改成如下：

但是此时仍要解决后端跨域问题

### Flask 解决跨域问题

我直接修改的\_\_init\_\_.py 文件

添加了如下代码

![](https://common.cnblogs.com/images/copycode.gif)

from flask import Flask from App.api import init_api from App.ext import init_ext from App.settings import envs from flask_cors import CORS# 添加的

def create_app():
    app \\= Flask(\_\_name\_\_)
    CORS(app,resources\\=r'/\*')# 添加的
    app.config.from_object(envs.get("develop"))

    init\_ext(app)
    init\_api(app) return app

![](https://common.cnblogs.com/images/copycode.gif)

更多有关 flask-cors 模块的用法可参考：[https://flask-cors.readthedocs.io/en/latest/](https://flask-cors.readthedocs.io/en/latest/)

参考文档：

[https://www.jianshu.com/p/43aa317d7683](https://www.jianshu.com/p/43aa317d7683)

[https://blog.csdn.net/weixin_42902669/article/details/90728697](https://blog.csdn.net/weixin_42902669/article/details/90728697) 
 [https://www.cnblogs.com/myrtle/p/14590775.html](https://www.cnblogs.com/myrtle/p/14590775.html)
