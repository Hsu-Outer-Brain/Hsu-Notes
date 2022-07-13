# Invalid Host header 服务器域名访问出现的问题_Cookysurongbin的博客-CSDN博客
![](https://csdnimg.cn/release/blogv2/dist/pc/img/original.png)

版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。

vue-cli 搭建的环境，用[nginx](https://so.csdn.net/so/search?q=nginx&spm=1001.2101.3001.7020)做代理服务器，访问时显示：Invalid Host header

经查是因为新版的[webpack](https://so.csdn.net/so/search?q=webpack&spm=1001.2101.3001.7020)-dev-server 出于安全考虑，默认检查 hostname，如果 hostname 不是配置内的就不能访问。这样有 2 中方法，一种是设置跳过 host 检查，一种是直接 host 设置成你的地址。

1、关闭 host 检查

-   可以在 build 目录下的 webpack.dev.conf.js 文件，devServer 下添加 disableHostCheck: true，跳过检查

![](https://img-blog.csdnimg.cn/20190108151400496.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0Nvb2t5c3Vyb25nYmlu,size_16,color_FFFFFF,t_70)

-   同样的原理，可以在 package.json 文件修改 scripts 命令: webpack-dev-server --disableHostCheck=true

![](https://img-blog.csdnimg.cn/20190108152321515.png)

2、设置成你的 host，加入你的 host 是 xxx.com，同样 2 中方法，修改配置文件，和 script 命令

-   在 config 目录下修改 index.js 文件的 host，这个默认是 localhost，可修改成   xxx.com![](https://img-blog.csdnimg.cn/20190108152555259.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0Nvb2t5c3Vyb25nYmlu,size_16,color_FFFFFF,t_70)
-   package.json 的 script 语句: webpack-dev-server --host=xxx.com 或者 --public=xxx.com 
    [https://blog.csdn.net/Cookysurongbin/article/details/86077241/](https://blog.csdn.net/Cookysurongbin/article/details/86077241/)
