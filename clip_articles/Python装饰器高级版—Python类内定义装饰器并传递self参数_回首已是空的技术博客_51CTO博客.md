# Python装饰器高级版—Python类内定义装饰器并传递self参数_回首已是空的技术博客_51CTO博客
本文重点：解决了类里面定义的装饰器，在同一个类里面使用的问题，并实现了装饰器的类属性参数传递

目录：

一、基本装饰器

二、在类里定义装饰器，装饰本类内函数

三、类装饰器

正文：

**一、基本装饰器**

        **装饰不带参数的函数**

**装饰带一个参数的函数**

    **装饰带不定长参数的函数**

    通常装饰器不只装饰一个函数，每个函数参数的个数也不相同  

    这个时候使用不定长参数\*args,\*\*kwargs  

**装饰器带参数**

**二、在类里定义装饰器，装饰本类内函数：** 

类装饰器，装饰函数和类函数调用不同的类函数

    **把装饰器写在类里**

        在类里面定义个函数，用来装饰其它函数，严格意义上说不属于类装饰器。

**装饰器装饰同一个类里的函数**

背景：想要通过装饰器修改类里的 self 属性值。

**三、类装饰器**

    **定义一个类装饰器，装饰函数，默认调用\_\_call\_\_方法**

**定义一个类装饰器，装饰类中的函数，默认调用\_\_get\_\_方法**

    实际上把类方法变成属性了，还记得类属性装饰器吧，@property  

    下面自已做一个 property  

    **做一个求和属性 sum，统计所有输入的数字的和**  

举报文章

请选择举报类型

内容侵权 涉嫌营销 内容抄袭 违法信息 其他

上传截图

格式支持 JPEG/PNG/JPG，图片不超过 1.9M

已经收到您得举报信息，我们会尽快审核

相关文章

* * *

 [https://blog.51cto.com/yishi/2354752](https://blog.51cto.com/yishi/2354752)
