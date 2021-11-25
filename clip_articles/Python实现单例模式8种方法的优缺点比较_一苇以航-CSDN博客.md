# Python实现单例模式8种方法的优缺点比较_一苇以航-CSDN博客
转自 [https://blog.csdn.net/u012324798/article/details/104056562](https://blog.csdn.net/u012324798/article/details/104056562) 

## 结论先行

本文在比较了[Python](https://so.csdn.net/so/search?from=pc_blog_highlight&q=Python)实现[单例模式](https://so.csdn.net/so/search?from=pc_blog_highlight&q=%E5%8D%95%E4%BE%8B%E6%A8%A1%E5%BC%8F)8 种方法的优缺点之后，认为在[Python](https://so.csdn.net/so/search?from=pc_blog_highlight&q=Python)中使用元类实现[单例模式](https://so.csdn.net/so/search?from=pc_blog_highlight&q=%E5%8D%95%E4%BE%8B%E6%A8%A1%E5%BC%8F)效果最好，推荐使用元类实现单例。

## 为什么要使用单例模式？

通常情况下，一个类可以多次实例化，每个实例对象是相互独立的，即类在同一时刻可以有多种实例状态。

如果我们希望：

-   某个类在在同一时刻不能有多种实例状态，即【单态】（例如：操作系统的任务管理器、回收站、文件系统；应用程序的日志应用、网站的计时工具或 ID(序号) 生成器）；
-   或者某个类只需要一个实例就够用了，即【单例】，多的实例不会有什么坏的影响，但也没什么作用，只是徒增资源消耗而已（例如：数据库连接池、线程池、web 开发中读取配置文件）。

这时如果某个类在程序运行期间只有一个实例，就能实现上述需求（单例 / 单态）。

最简单的办法就是只实例化一次得到一个实例，自己记住，之后写代码的时候一直用它，不再实例化。问题是自己有可能忘了某个类已经实例化过了，而且别人想复用你的代码时也不知道你已经实例化过了，就很容易发生重复实例化，自然无法确保单例效果。为了解决这个问题，我们希望在这个类本身能够实现无论实例化多少次，每次返回的都是同一个实例的效果，这就是**单例模式**。这样后续使用这个类的人就无需关注是否已经实例化过，反正重复实例化得到的还是单例。

## 实例化过程原理

类 ()：即 obj = MyClass() ，执行 MyClass 的**元类**的\_\_call\_\_方法，如果没有使用 metaclass 指定元类，则默认元类为 type  
实例 ()：即 obj()，执行 MyClass 类的\_\_call\_\_方法

obj = MyClass() 这一实例化过程其实是调用其元类的\_\_call\_\_方法，如果未指定元类，则调用默认元类 type 的\_\_call\_\_方法，而这一方法是由 type_call 函数（C[Python](https://so.csdn.net/so/search?from=pc_blog_highlight&q=Python) 中的 C 代码）来实现的，其实现的基本逻辑简化后相当于 Python 的以下代码：即先调用 MyClass 的\_\_new\_\_方法创建实例，如果实例符合 2 个条件，则调用 MyClass 的\_\_init\_\_方法对实例进行初始化。

````null
def __call__(obj_type, *args, **kwargs):    obj = obj_type.__new__(*args, **kwargs)if obj is not None and isinstance(obj, obj_type):        obj.__init__(*args, **kwargs)```

由此可见，控制实例化过程的是3个特殊方法：

*   元类的\_\_call\_\_方法是实例化过程中最早调用的方法，它控制了类的\_\_new\_\_和\_\_init\_\_方法的调用
*   类的\_\_new\_\_方法是第二个调用的方法，影响创建实例的具体过程
*   类的\_\_init\_\_方法是最后调用的方法，影响实例的初始化

想要让MyClass成为[单例模式](https://so.csdn.net/so/search?from=pc_blog_highlight&q=%E5%8D%95%E4%BE%8B%E6%A8%A1%E5%BC%8F)，在实例化过程前加上条件判断就可以。问题是在哪里增加条件判断的代码？由于条件判断的代码所在位置不同，单例模式有多种实现方式。本文的重点就是比较分析这些实现方式的优缺点。

评价标准
----

为了比较网上比较流行的Python实现单例模式的8种方法，提出以下评价标准：

*   将单例模式的代码独立封装为Singleton方便复用，MyClass及其子类SubClass无需关心单例模式的实现
*   MyClass类有继承的基类OtherClass时不影响单例效果
*   单例模式不影响MyClass作为基类被SubClass继承，且子类SubClass也是单例模式，即SubClass每次实例化得到都是同一个SubClass实例
*   变量MyClass代表的是真正的MyClass类，能使用类属性、类方法
*   type(obj) 返回的是真正的MyClass类
*   type(obj) () 返回的也是单例，单例效果稳定
*   有 lazy loading的效果，即导入模块时没有发生实例化，不消耗资源，等到需要使用时再实例化
*   多线程并发环境下得到的也是单例

除了实现方式（一）、（二）以外，都使用以下测试代码：  
test.py

```null
from singleton_class import MyClassdef print_task(obj=None, arg=1):print('arg: %d, id(obj): %d' % (arg, id(obj)))print('arg: %d, before: %s' % (arg, before))print('arg: %d, after: %s' % (arg, after))if __name__ == '__main__':print('_____________________________MyClass MultiThreading Test______________________________________')        t = threading.Thread(target=print_task, args=(None, i))print('_____________________________MyClass______________________________________')print('MyClass: %s, type(MyClass): %s' % (MyClass, type(MyClass)))print('MyClass.class_attribute(): %s' % MyClass.class_attribute)print('由于type(MyClass): %s，变量MyClass不能使用类属性、类方法' % type(MyClass))print('type(obj): %s' % type(obj1))print('_____________________________SubClass______________________________________')super(SubClass, self).__init__()print('SubClass: %s, type(SubClass): %s' % (SubClass, type(SubClass)))print('SubClass.class_attribute(): %s' % SubClass.class_attribute)print('type(sub_obj): %s' % type(sub_obj1))        sub_obj3 = type(sub_obj1)()print('由于type(MyClass): %s，变量MyClass不能作为基类被继承' % type(MyClass))```

（一）使用类方法 getInstance 作为获取实例的接口
------------------------------

此实现方式将条件判断的代码放在了类方法 getInstance 中，必须调用类方法 getInstance 作为获取实例的接口才能获得单例，如果使用 obj = MyClass() 这种常规方式，则获取到的不是单例，非常不实用，就不测试分析了，直接跳过。

```null
if cls._instance is None:```

（二）使用模块（同名实例替换类变量）
------------------

此实现方式没有增加条件判断的代码，而是借助了模块的单例特性，确保只有一个实例存在。  
[为什么Python模块就是天然的单例模式？](https://blog.csdn.net/u012324798/article/details/104136270)

由于在程序运行期间模块只装载一次，并且模块A中的全局变量a绑定成了模块的属性，即A.a只有一个。如果A.a代表的是实例，并且切断了其他实例化的途径，则该类的实例就只有一个。

在声明 MyClass 类之后直接实例化一个单例保存在同名的全局变量 MyClass 中，即MyClass = MyClass()，这样变量 MyClass 代表的就是一个实例，不是类，也就不能通过 obj = MyClass() 这行代码进行实例化，会报错。为了不让这个这行代码报错，给MyClass类增加一个\_\_call\_\_方法（使得实例可以向函数一样通过 () 被调用），返回实例本身。这样每次通过 obj = MyClass() 这行代码进行“实例化”时，并没有真正进行实例化，而是调用了实例的\_\_call\_\_方法，返回实例本身，从而实现了单例模式。

singleton\_class.py

```null
def __call__(self, *args, **kwargs):class OtherClass(object):return super(OtherClass, cls).__new__(cls)super(OtherClass, self).__init__()class MyClass(Singleton, OtherClass):    class_attribute = 'class_default'super(__class__, self).__init__()```

test.py

```null
def print_task(obj, arg):print('arg: %d, id(obj): %d' % (arg, id(obj)))print('arg: %d, before: %s' % (arg, before))print('arg: %d, after: %s' % (arg, after))print('arg: %d before import' % arg)    # import语句是线程安全的，即使多线程并发导入同一个模块，也不会重复装载模块    from singleton_class import MyClassprint_task(MyClass(), arg)if __name__ == '__main__':print('_____________________________MyClass MultiThreading Test______________________________________')        t = threading.Thread(target=task, args=(i, ))    from ingleton_class import MyClassprint('_____________________________MyClass______________________________________')print('type(MyClass): %s' % type(MyClass))print('MyClass.class_attribute(): %s' % MyClass.class_attribute)print('_____________________________SubClass______________________________________')    # 注意：变量MyClass代表的是实例，不是类，不能直接用变量MyClass作为基类，而需使用MyClass.__class__作为基类    class SubClass(MyClass.__class__):            # 由于模块装载后，变量 SubClass 代表的是实例，不是类，不能使用super(SubClass, self).__init__()            # 要获得真实的SubClass类，需使用__class__super(__class__, self).__init__()print('SubClass: %s, type(SubClass): %s' % (SubClass, type(SubClass)))print('SubClass.class_attribute(): %s' % SubClass.class_attribute)print('type(sub_obj): %s' % type(sub_obj1))    sub_obj3 = type(sub_obj1)()```

输出：  
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ MyClass MultiThreading Test\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  
arg: 1 before import  
arg: 2 before import  
arg: 3 before import  
arg: 4 before import  
arg: 5 before import  
arg: 6 before import  
arg: 7 before import  
arg: 8 before import  
arg: 9 before import  
arg: 10 before import  
arg: 1, id(obj): 40451880  
arg: 1, before: {‘a’: ‘a\_default’, ‘x’: ‘x\_default’}  
arg: 1, after: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, id(obj): 40451880  
arg: 2, before: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 3, id(obj): 40451880  
arg: 3, before: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 3, after: {‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
arg: 4, id(obj): 40451880  
arg: 2, after: {‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
arg: 5, id(obj): 40451880  
arg: 5, before: {‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
arg: 5, after: {‘a’: ‘a\_5’, ‘x’: ‘x\_5’}  
arg: 6, id(obj): 40451880  
arg: 6, before: {‘a’: ‘a\_5’, ‘x’: ‘x\_5’}  
arg: 6, after: {‘a’: ‘a\_6’, ‘x’: ‘x\_6’}  
arg: 4, before: {‘a’: ‘a\_6’, ‘x’: ‘x\_6’}  
arg: 4, after: {‘a’: ‘a\_4’, ‘x’: ‘x\_4’}  
arg: 7, id(obj): 40451880  
arg: 8, id(obj): 40451880  
arg: 8, before: {‘a’: ‘a\_4’, ‘x’: ‘x\_4’}  
arg: 7, before: {‘a’: ‘a\_4’, ‘x’: ‘x\_4’}  
arg: 7, after: {‘a’: ‘a\_7’, ‘x’: ‘x\_7’}  
arg: 8, after: {‘a’: ‘a\_8’, ‘x’: ‘x\_8’}  
arg: 9, id(obj): 40451880  
arg: 9, before: {‘a’: ‘a\_8’, ‘x’: ‘x\_8’}  
arg: 10, id(obj): 40451880  
arg: 10, before: {‘a’: ‘a\_8’, ‘x’: ‘x\_8’}  
arg: 10, after: {‘a’: ‘a\_10’, ‘x’: ‘x\_10’}  
arg: 9, after: {‘a’: ‘a\_9’, ‘x’: ‘x\_9’}  
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ MyClass\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  
MyClass: <singleton\_class.MyClass object at 0x0000000002693F28>, type(MyClass): <class ‘singleton\_class.MyClass’>  
MyClass.class\_attribute(): class\_default  
class function  
arg: 1, id(obj): 40451880  
arg: 1, before: {‘a’: ‘a\_9’, ‘x’: ‘x\_9’}  
arg: 1, after: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, id(obj): 40451880  
arg: 2, before: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, after: {‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
type(obj): <class ‘singleton\_class.MyClass’>  
arg: 3, id(obj): 40101928  
arg: 3, before: {‘a’: ‘a\_default’, ‘x’: ‘x\_default’}  
arg: 3, after: {‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ SubClass\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  
SubClass: <\_\_ main\_\_.SubClass object at 0x00000000026752E8>, type(SubClass): <class ‘\_\_ main\_\_.SubClass’>  
SubClass.class\_attribute(): class\_default  
class function  
arg: 11, id(obj): 40325864  
arg: 11, before: {‘a’: ‘a\_default’, ‘x’: ‘x\_default’, ‘y’: ‘y\_default’}  
arg: 11, after: {‘a’: ‘a\_11’, ‘x’: ‘x\_11’, ‘y’: ‘y\_11’}  
arg: 12, id(obj): 40325864  
arg: 12, before: {‘a’: ‘a\_11’, ‘x’: ‘x\_11’, ‘y’: ‘y\_11’}  
arg: 12, after: {‘a’: ‘a\_12’, ‘x’: ‘x\_12’, ‘y’: ‘y\_12’}  
type(sub\_obj): <class ‘\_\_ main\_\_.SubClass’>  
arg: 13, id(obj): 40326032  
arg: 13, before: {‘a’: ‘a\_default’, ‘x’: ‘x\_default’, ‘y’: ‘y\_default’}  
arg: 13, after: {‘a’: ‘a\_13’, ‘x’: ‘x\_13’, ‘y’: ‘y\_13’}

优点：

*   （核心优点）无需设置双重校验锁，自动实现在多线程并发环境下仍然是单例，原因是实例化发生在模块装载时，而import语句是线程安全的，即使多线程并发导入同一个模块，也不会重复装载模块，从而确保了只有一个实例存在
*   MyClass类有继承的基类OtherClass时不影响单例效果
*   变量MyClass代表的是MyClass类的实例，能使用类属性、类方法
*   type(obj) 返回的是真正的MyClass类

缺点：

*   单例模式的代码不能完全独立封装，MyClass类及其子类SubClass为了实现单例模式，必须注意使用同名实例替换类变量
*   单例模式影响MyClass作为基类被SubClass继承，因为变量MyClass代表的是实例，不是类，不能直接用变量MyClass作为基类，而需使用MyClass.\_\_class\_\_作为基类
*   通过obj3 = type(obj1)() 得到的实例 obj3 不是单例，单例效果不稳定
*   没有达到lazy loading的效果，在装载模块时就完成实例化了

（三）使用函数装饰器
----------

此实现方式将MyClass这个变量替换为函数装饰器，从而可以在装饰器中进行条件判断。由于装饰器只是替换了MyClass这个变量，真正的实例化过程并未开始，当符合条件判断时，只需通过 obj = MyClass() 这种常规方式完成实例化即可。

singleton\_class.py

```null
    _instance_lock = threading.Lock()def _wrapper(*args, **kargs):                    _instance[cls] = cls(*args, **kargs)class OtherClass(object):return super(OtherClass, cls).__new__(cls)super(OtherClass, self).__init__()class MyClass(OtherClass):    class_attribute = 'class_default'super(__class__, self).__init__()```

输出：  
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ MyClass MultiThreading Test\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  
arg: 1, id(obj): 39810552  
arg: 2, id(obj): 39810552  
arg: 2, before: {‘a’: ‘a\_default’, ‘x’: ‘x\_default’}  
arg: 2, after: {‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
arg: 1, before: {‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
arg: 1, after: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 3, id(obj): 39810552  
arg: 4, id(obj): 39810552  
arg: 3, before: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 3, after: {‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
arg: 5, id(obj): 39810552  
arg: 5, before: {‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
arg: 5, after: {‘a’: ‘a\_5’, ‘x’: ‘x\_5’}  
arg: 4, before: {‘a’: ‘a\_5’, ‘x’: ‘x\_5’}  
arg: 6, id(obj): 39810552  
arg: 7, id(obj): 39810552  
arg: 7, before: {‘a’: ‘a\_5’, ‘x’: ‘x\_5’}  
arg: 6, before: {‘a’: ‘a\_5’, ‘x’: ‘x\_5’}  
arg: 6, after: {‘a’: ‘a\_6’, ‘x’: ‘x\_6’}  
arg: 7, after: {‘a’: ‘a\_7’, ‘x’: ‘x\_7’}  
arg: 4, after: {‘a’: ‘a\_4’, ‘x’: ‘x\_4’}  
arg: 8, id(obj): 39810552  
arg: 8, before: {‘a’: ‘a\_4’, ‘x’: ‘x\_4’}  
arg: 8, after: {‘a’: ‘a\_8’, ‘x’: ‘x\_8’}  
arg: 9, id(obj): 39810552  
arg: 9, before: {‘a’: ‘a\_8’, ‘x’: ‘x\_8’}  
arg: 9, after: {‘a’: ‘a\_9’, ‘x’: ‘x\_9’}  
arg: 10, id(obj): 39810552  
arg: 10, before: {‘a’: ‘a\_9’, ‘x’: ‘x\_9’}  
arg: 10, after: {‘a’: ‘a\_10’, ‘x’: ‘x\_10’}  
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ MyClass\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  
MyClass: <function Singleton.< locals >.\_ wrapper at 0x00000000025EEEA0>, type(MyClass): <class ‘function’>  
由于type(MyClass): <class ‘function’>，变量MyClass不能使用类属性、类方法  
arg: 1, id(obj): 39810552  
arg: 1, before: {‘a’: ‘a\_10’, ‘x’: ‘x\_10’}  
arg: 1, after: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, id(obj): 39810552  
arg: 2, before: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, after: {‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
type(obj): <class ‘singleton\_class.MyClass’>  
arg: 3, id(obj): 39727056  
arg: 3, before: {‘a’: ‘a\_default’, ‘x’: ‘x\_default’}  
arg: 3, after: {‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ SubClass\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  
由于type(MyClass): <class ‘function’>，变量MyClass不能作为基类被继承

优点：

*   将单例模式的代码独立封装，MyClass类无需关心单例模式的实现
*   MyClass有继承的基类OtherClass时不影响单例效果
*   type(obj) 返回的是真正的MyClass类
*   有 lazy loading 效果

缺点：

*   单例模式导致MyClass不能作为基类被SubClass继承
*   经过装饰后的变量MyClass代表的是一个\_wrapper函数，而不是真正的MyClass类，不能使用MyClass的类属性、类方法
*   obj3 = type(obj1)() 得到的实例 obj3 不是单例，单例效果不稳定
*   需手动设置双重校验锁，才能在保证多线程并发环境下得到的也是单例

（四）使用类装饰器
---------

此实现方式与（三）使用函数装饰器非常相似，没有本质区别，只不过装饰器的写法不同。  
singleton\_class.py

```null
    _instance_lock = threading.Lock()def __call__(self, *args, **kwargs):if self._instance is None:with Singleton._instance_lock:if self._instance is None:                    self._instance = self._cls(*args, **kwargs)class OtherClass(object):return super(OtherClass, cls).__new__(cls)super(OtherClass, self).__init__()class MyClass(OtherClass):    class_attribute = 'class_default'super(__class__, self).__init__()```

输出：  
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ MyClass MultiThreading Test\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  
arg: 1, id(obj): 39923272  
arg: 2, id(obj): 39923272  
arg: 2, before: {‘a’: ‘a\_default’, ‘x’: ‘x\_default’}  
arg: 2, after: {‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
arg: 1, before: {‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
arg: 1, after: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 3, id(obj): 39923272  
arg: 3, before: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 3, after: {‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
arg: 4, id(obj): 39923272  
arg: 4, before: {‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
arg: 4, after: {‘a’: ‘a\_4’, ‘x’: ‘x\_4’}  
arg: 5, id(obj): 39923272  
arg: 5, before: {‘a’: ‘a\_4’, ‘x’: ‘x\_4’}  
arg: 5, after: {‘a’: ‘a\_5’, ‘x’: ‘x\_5’}  
arg: 6, id(obj): 39923272  
arg: 6, before: {‘a’: ‘a\_5’, ‘x’: ‘x\_5’}  
arg: 6, after: {‘a’: ‘a\_6’, ‘x’: ‘x\_6’}  
arg: 7, id(obj): 39923272  
arg: 7, before: {‘a’: ‘a\_6’, ‘x’: ‘x\_6’}  
arg: 8, id(obj): 39923272  
arg: 7, after: {‘a’: ‘a\_7’, ‘x’: ‘x\_7’}  
arg: 9, id(obj): 39923272  
arg: 8, before: {‘a’: ‘a\_7’, ‘x’: ‘x\_7’}  
arg: 8, after: {‘a’: ‘a\_8’, ‘x’: ‘x\_8’}  
arg: 10, id(obj): 39923272  
arg: 9, before: {‘a’: ‘a\_8’, ‘x’: ‘x\_8’}  
arg: 9, after: {‘a’: ‘a\_9’, ‘x’: ‘x\_9’}  
arg: 10, before: {‘a’: ‘a\_9’, ‘x’: ‘x\_9’}  
arg: 10, after: {‘a’: ‘a\_10’, ‘x’: ‘x\_10’}  
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ MyClass\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  
MyClass: <singleton\_class.Singleton object at 0x00000000026260B8>, type(MyClass): <class ‘singleton\_class.Singleton’>  
由于type(MyClass): <class ‘singleton\_class.Singleton’>，变量MyClass不能使用类属性、类方法  
arg: 1, id(obj): 39923272  
arg: 1, before: {‘a’: ‘a\_10’, ‘x’: ‘x\_10’}  
arg: 1, after: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, id(obj): 39923272  
arg: 2, before: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, after: {‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
type(obj): <class ‘singleton\_class.MyClass’>  
arg: 3, id(obj): 39744456  
arg: 3, before: {‘a’: ‘a\_default’, ‘x’: ‘x\_default’}  
arg: 3, after: {‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ SubClass\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  
由于type(MyClass): <class ‘singleton\_class.Singleton’>，变量MyClass不能作为基类被继承

优缺点与函数装饰器相同，区别在于经过装饰后的变量MyClass代表的是Singleton类（装饰器）的实例，它也不代表真正的MyClass类。

（五）使用装饰器返回类
-----------

在实现方式（三）、（四）中，经过装饰后的变量MyClass代表的都不是类，不能作为基类被SubClass继承，为了解决这个问题，考虑使用装饰器返回类的实现方式。

由实例化过程原理分析，我们知道了：类的\_\_new\_\_方法是第二个调用的方法，影响创建实例的具体过程；类的\_\_init\_\_方法是最后调用的方法，影响实例的初始化。

那么，通过在类的\_\_new\_\_方法增加条件判断的代码，就能实现单例模式。但是没这么简单，必须要注意三个问题：

1.  每次调用\_\_new\_\_方法之后，会紧接着调用\_\_init\_\_方法。所以，必须给出一个标志，避免\_\_init\_\_方法重复调用导致成员变量被默认值覆盖。
2.  多线程并发环境下，如果MyClass类的\_\_init\_\_方法中有耗时操作，则装饰器内的\_\_init\_\_方法也要加双重校验锁。
3.  经过装饰后的变量MyClass代表的是class\_wrapper类，从而SubClass继承的就是class\_wrapper类，但SubClass真正想要继承的是MyClass类。于是，为了能够让SubClass继承到MyClass类，class\_wrapper类必须继承于MyClass类。

singleton\_class.py

```null
class class_wrapper(_cls):        _instance_lock = threading.Lock()if cls not in class_wrapper._instance.keys():with class_wrapper._instance_lock:if cls not in class_wrapper._instance.keys():                        class_wrapper._instance[cls] = super(class_wrapper, cls).__new__(cls)                        class_wrapper._instance[cls]._intialed = Falsereturn class_wrapper._instance[cls]with class_wrapper._instance_lock:super(class_wrapper, self).__init__()    class_wrapper.__name__ = _cls.__name__class OtherClass(object):return super(OtherClass, cls).__new__(cls)super(OtherClass, self).__init__()class MyClass(OtherClass):    class_attribute = 'class_default'super(__class__, self).__init__()```

注意： test.py中的SubClass需改为以下代码

```null
super(SubClass, self).__init__()```

*   1
*   2
*   3
*   4
*   5
*   6
*   7
*   8
*   9

输出：  
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ MyClass MultiThreading Test\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  
arg: 1, id(obj): 39289072  
arg: 1, before: {’\_ intialed’: True, ‘a’: ‘a\_default’, ‘x’: ‘x\_default’}  
arg: 1, after: {’\_ intialed’: True, ‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, id(obj): 39289072  
arg: 2, before: {’\_ intialed’: True, ‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, after: {’\_ intialed’: True, ‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
arg: 3, id(obj): 39289072  
arg: 3, before: {’\_ intialed’: True, ‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
arg: 3, after: {’\_ intialed’: True, ‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
arg: 4, id(obj): 39289072  
arg: 4, before: {’\_ intialed’: True, ‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
arg: 4, after: {’\_ intialed’: True, ‘a’: ‘a\_4’, ‘x’: ‘x\_4’}  
arg: 5, id(obj): 39289072  
arg: 5, before: {’\_ intialed’: True, ‘a’: ‘a\_4’, ‘x’: ‘x\_4’}  
arg: 5, after: {’\_ intialed’: True, ‘a’: ‘a\_5’, ‘x’: ‘x\_5’}  
arg: 6, id(obj): 39289072  
arg: 6, before: {’\_ intialed’: True, ‘a’: ‘a\_5’, ‘x’: ‘x\_5’}  
arg: 6, after: {’\_ intialed’: True, ‘a’: ‘a\_6’, ‘x’: ‘x\_6’}  
arg: 7, id(obj): 39289072  
arg: 7, before: {’\_ intialed’: True, ‘a’: ‘a\_6’, ‘x’: ‘x\_6’}  
arg: 7, after: {’\_ intialed’: True, ‘a’: ‘a\_7’, ‘x’: ‘x\_7’}  
arg: 8, id(obj): 39289072  
arg: 8, before: {’\_ intialed’: True, ‘a’: ‘a\_7’, ‘x’: ‘x\_7’}  
arg: 8, after: {’\_ intialed’: True, ‘a’: ‘a\_8’, ‘x’: ‘x\_8’}  
arg: 9, id(obj): 39289072  
arg: 9, before: {’\_ intialed’: True, ‘a’: ‘a\_8’, ‘x’: ‘x\_8’}  
arg: 9, after: {’\_ intialed’: True, ‘a’: ‘a\_9’, ‘x’: ‘x\_9’}  
arg: 10, id(obj): 39289072  
arg: 10, before: {’\_ intialed’: True, ‘a’: ‘a\_9’, ‘x’: ‘x\_9’}  
arg: 10, after: {’\_ intialed’: True, ‘a’: ‘a\_10’, ‘x’: ‘x\_10’}  
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ MyClass\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  
MyClass: <class ‘singleton\_class.Singleton.< locals >.class\_wrapper’>, type(MyClass): <class ‘type’>  
MyClass.class\_attribute(): class\_default  
class function  
arg: 1, id(obj): 39289072  
arg: 1, before: {’\_ intialed’: True, ‘a’: ‘a\_10’, ‘x’: ‘x\_10’}  
arg: 1, after: {’\_ intialed’: True, ‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, id(obj): 39289072  
arg: 2, before: {’\_ intialed’: True, ‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, after: {’\_ intialed’: True, ‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
type(obj): <class ‘singleton\_class.Singleton.< locals >.class\_wrapper’>  
arg: 3, id(obj): 39289072  
arg: 3, before: {’\_ intialed’: True, ‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
arg: 3, after: {’\_ intialed’: True, ‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ SubClass\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  
SubClass: <class ‘\_\_ main\_\_.SubClass’>, type(SubClass): <class ‘type’>  
SubClass.class\_attribute(): class\_default  
class function  
arg: 11, id(obj): 39187120  
arg: 11, before: {’\_ initialed’: True, ‘a’: ‘a\_default’, ‘x’: ‘x\_default’, ‘y’: ‘y\_default’}  
arg: 11, after: {’\_ initialed’: True, ‘a’: ‘a\_11’, ‘x’: ‘x\_11’, ‘y’: ‘y\_11’}  
arg: 12, id(obj): 39187120  
arg: 12, before: {’\_ initialed’: True, ‘a’: ‘a\_11’, ‘x’: ‘x\_11’, ‘y’: ‘y\_11’}  
arg: 12, after: {’\_ initialed’: True, ‘a’: ‘a\_12’, ‘x’: ‘x\_12’, ‘y’: ‘y\_12’}  
type(sub\_obj): <class ‘\_\_ main\_\_.SubClass’>  
arg: 13, id(obj): 39187120  
arg: 13, before: {’\_ initialed’: True, ‘a’: ‘a\_12’, ‘x’: ‘x\_12’, ‘y’: ‘y\_12’}  
arg: 13, after: {’\_ initialed’: True, ‘a’: ‘a\_13’, ‘x’: ‘x\_13’, ‘y’: ‘y\_13’}

优点：

*   MyClass类无需关心单例模式的实现
*   MyClass有继承的基类OtherClass时不影响单例效果
*   单例模式不影响MyClass作为基类被继承，且子类SubClass也是单例模式，即SubClass每次实例化得到都是同一个SubClass实例
*   经过装饰后的变量MyClass代表了class\_wrapper类，它是MyClass类的子类，从而能使用MyClass类的类属性、类方法
*   通过obj3 = type(obj1)() 得到的实例也是单例，单例效果稳定
*   有 lazy loading 效果

缺点：

*   单例模式的代码不能完全独立封装，子类SubClass需关心单例模式的实现（必须注意不能重载\_\_new\_\_方法，否则SubClass不是单例模式；\_\_init\_\_方法需注意避免重复初始化成员变量）
*   type(obj) 返回的是class\_wrapper类，不是真正的MyClass类，即obj实际上是class\_wrapper类的单例
*   需手动设置双重校验锁，才能在保证多线程并发环境下得到的也是单例

（六）使用基类
-------

在实现方式（五）中，经过装饰后的变量MyClass代表了class\_wrapper类，不是真正的MyClass类。为了解决这个问题，考虑使用基类的实现方式。

这种实现方式也是通过在类的\_\_new\_\_方法增加条件判断的代码。但是Singleton是基类，MyClass是子类，基类无法控制子孙类方法的调用，因此必须要注意三个问题：

1.  每次调用\_\_new\_\_方法之后，会紧接着调用\_\_init\_\_方法。所以，必须给出一个标志，避免\_\_init\_\_方法重复调用导致成员变量被默认值覆盖。
2.  基类Singleton无法控制子孙类方法的调用，如果子孙类有自己特殊的成员变量需要初始化，则\_\_init\_\_方法不能封装到基类Singleton中。
3.  多线程并发环境下，如果\_\_init\_\_方法中有耗时操作，则\_\_init\_\_方法也要加双重校验锁。

singleton\_class.py

```null
    _instance_lock = threading.Lock()def __new__(cls, *args, **kwargs):if cls not in Singleton._instance.keys():with Singleton._instance_lock:if cls not in Singleton._instance.keys():                    Singleton._instance[cls] = super(Singleton, cls).__new__(cls)                    Singleton._instance[cls]._intialed = Falsereturn Singleton._instance[cls]class OtherClass(object):return super(OtherClass, cls).__new__(cls)super(OtherClass, self).__init__()class MyClass(Singleton, OtherClass):    class_attribute = 'class_default'with self._instance_lock:super(MyClass, self).__init__()```

注意： test.py中的SubClass需改为以下代码

```null
super(SubClass, self).__init__()```

输出：  
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ MyClass MultiThreading Test\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  
arg: 1, id(obj): 39579888  
arg: 1, before: {’\_ intialed’: True, ‘a’: ‘a\_default’, ‘x’: ‘x\_default’}  
arg: 1, after: {’\_ intialed’: True, ‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, id(obj): 39579888  
arg: 2, before: {’\_ intialed’: True, ‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, after: {’\_ intialed’: True, ‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
arg: 3, id(obj): 39579888  
arg: 3, before: {’\_ intialed’: True, ‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
arg: 3, after: {’\_ intialed’: True, ‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
arg: 4, id(obj): 39579888  
arg: 4, before: {’\_ intialed’: True, ‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
arg: 4, after: {’\_ intialed’: True, ‘a’: ‘a\_4’, ‘x’: ‘x\_4’}  
arg: 5, id(obj): 39579888  
arg: 5, before: {’\_ intialed’: True, ‘a’: ‘a\_4’, ‘x’: ‘x\_4’}  
arg: 5, after: {’\_ intialed’: True, ‘a’: ‘a\_5’, ‘x’: ‘x\_5’}  
arg: 6, id(obj): 39579888  
arg: 6, before: {’\_ intialed’: True, ‘a’: ‘a\_5’, ‘x’: ‘x\_5’}  
arg: 6, after: {’\_ intialed’: True, ‘a’: ‘a\_6’, ‘x’: ‘x\_6’}  
arg: 7, id(obj): 39579888  
arg: 7, before: {’\_ intialed’: True, ‘a’: ‘a\_6’, ‘x’: ‘x\_6’}  
arg: 7, after: {’\_ intialed’: True, ‘a’: ‘a\_7’, ‘x’: ‘x\_7’}  
arg: 8, id(obj): 39579888  
arg: 8, before: {’\_ intialed’: True, ‘a’: ‘a\_7’, ‘x’: ‘x\_7’}  
arg: 9, id(obj): 39579888  
arg: 9, before: {’\_ intialed’: True, ‘a’: ‘a\_7’, ‘x’: ‘x\_7’}  
arg: 8, after: {’\_ intialed’: True, ‘a’: ‘a\_8’, ‘x’: ‘x\_8’}  
arg: 9, after: {’\_ intialed’: True, ‘a’: ‘a\_9’, ‘x’: ‘x\_9’}  
arg: 10, id(obj): 39579888  
arg: 10, before: {’\_ intialed’: True, ‘a’: ‘a\_9’, ‘x’: ‘x\_9’}  
arg: 10, after: {’\_ intialed’: True, ‘a’: ‘a\_10’, ‘x’: ‘x\_10’}  
_****************************MyClass****************************_\_\_\_\_\_\_\_\_\_  
MyClass: <class ‘singleton\_class.MyClass’>, type(MyClass): <class ‘type’>  
MyClass.class\_attribute(): class\_default  
class function  
arg: 1, id(obj): 39579888  
arg: 1, before: {’\_ intialed’: True, ‘a’: ‘a\_10’, ‘x’: ‘x\_10’}  
arg: 1, after: {’\_ intialed’: True, ‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, id(obj): 39579888  
arg: 2, before: {’\_ intialed’: True, ‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, after: {’\_ intialed’: True, ‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
type(obj): <class ‘singleton\_class.MyClass’>  
arg: 3, id(obj): 39579888  
arg: 3, before: {’\_ intialed’: True, ‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
arg: 3, after: {’\_ intialed’: True, ‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ SubClass\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  
SubClass: <class ‘\_\_ main\_\_.SubClass’>, type(SubClass): <class ‘type’>  
SubClass.class\_attribute(): class\_default  
class function  
arg: 11, id(obj): 39974392  
arg: 11, before: {’\_ intialed’: True, ‘a’: ‘a\_default’, ‘x’: ‘x\_default’, ‘y’: ‘y\_default’}  
arg: 11, after: {’\_ intialed’: True, ‘a’: ‘a\_11’, ‘x’: ‘x\_11’, ‘y’: ‘y\_11’}  
arg: 12, id(obj): 39974392  
arg: 12, before: {’\_ intialed’: True, ‘a’: ‘a\_11’, ‘x’: ‘x\_11’, ‘y’: ‘y\_11’}  
arg: 12, after: {‘_intialed’: True, ‘a’: ‘a\_12’, ‘x’: ‘x\_12’, ‘y’: ‘y\_12’}  
type(sub\_obj): <class '_\_ main\_\_.SubClass’>  
arg: 13, id(obj): 39974392  
arg: 13, before: {’\_ intialed’: True, ‘a’: ‘a\_12’, ‘x’: ‘x\_12’, ‘y’: ‘y\_12’}  
arg: 13, after: {’\_ intialed’: True, ‘a’: ‘a\_13’, ‘x’: ‘x\_13’, ‘y’: ‘y\_13’}

优点：

*   MyClass类有继承的基类OtherClass时不影响单例效果（前提条件：Singleton需作为第一基类）
*   单例模式不影响MyClass类作为基类被继承，且子类SubClass也是单例模式，即SubClass每次实例化得到都是同一个SubClass实例
*   变量MyClass代表的是真正的MyClass类，能使用类属性、类方法
*   type(obj) 返回的是真正的MyClass类
*   通过obj3 = type(obj1)() 得到的实例也是单例，单例效果稳定
*   有 lazy loading 效果

缺点：

*   单例模式的代码不能完全独立封装，MyClass类、SubClass类需关心单例模式的实现（必须注意不重载\_\_new\_\_方法，否则不是单例；\_\_init\_\_方法需避免重复初始化成员变量；Singleton需作为第一基类）
*   需手动设置双重校验锁，才能在保证多线程并发环境下得到的也是单例

（七）使用元类
-------

实现方式（五）、（六）都是通过在类的\_\_new\_\_方法中增加条件判断的代码来实现单例模式的，导致单例模式的代码不能完全独立封装。

回顾实例化过程的原理，元类的\_\_call\_\_方法是实例化过程中最早调用的方法，从而可以在元类的\_\_call\_\_方法中增加条件判断。

需要注意的是，由于此时实例化过程已开始，当符合条件判断时，需通过super函数调用父元类type的\_\_call\_\_方法完成实例化。

singleton\_class.py

```null
    _instance_lock = threading.Lock()def __init__(cls, *args, **kwargs):super(Singleton, cls).__init__(*args, **kwargs)def __call__(cls, *args, **kwargs):if cls._instance is None:with Singleton._instance_lock:if cls._instance is None:                    cls._instance = super(Singleton, cls).__call__(*args, **kwargs)class OtherClass(object):return super(OtherClass, cls).__new__(cls)super(OtherClass, self).__init__()class MyClass(OtherClass, metaclass=Singleton):    class_attribute = 'class_default'super(MyClass, self).__init__()```

输出：  
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ MyClass MultiThreading Test\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  
arg: 1, id(obj): 39340128  
arg: 1, before: {‘a’: ‘a\_default’, ‘x’: ‘x\_default’}  
arg: 2, id(obj): 39340128  
arg: 1, after: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, before: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, after: {‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
arg: 3, id(obj): 39340128  
arg: 3, before: {‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
arg: 3, after: {‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
arg: 4, id(obj): 39340128  
arg: 4, before: {‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
arg: 4, after: {‘a’: ‘a\_4’, ‘x’: ‘x\_4’}  
arg: 5, id(obj): 39340128  
arg: 5, before: {‘a’: ‘a\_4’, ‘x’: ‘x\_4’}  
arg: 5, after: {‘a’: ‘a\_5’, ‘x’: ‘x\_5’}  
arg: 6, id(obj): 39340128  
arg: 6, before: {‘a’: ‘a\_5’, ‘x’: ‘x\_5’}  
arg: 6, after: {‘a’: ‘a\_6’, ‘x’: ‘x\_6’}  
arg: 7, id(obj): 39340128  
arg: 7, before: {‘a’: ‘a\_6’, ‘x’: ‘x\_6’}  
arg: 7, after: {‘a’: ‘a\_7’, ‘x’: ‘x\_7’}  
arg: 8, id(obj): 39340128  
arg: 8, before: {‘a’: ‘a\_7’, ‘x’: ‘x\_7’}  
arg: 8, after: {‘a’: ‘a\_8’, ‘x’: ‘x\_8’}  
arg: 9, id(obj): 39340128  
arg: 9, before: {‘a’: ‘a\_8’, ‘x’: ‘x\_8’}  
arg: 9, after: {‘a’: ‘a\_9’, ‘x’: ‘x\_9’}  
arg: 10, id(obj): 39340128  
arg: 10, before: {‘a’: ‘a\_9’, ‘x’: ‘x\_9’}  
arg: 10, after: {‘a’: ‘a\_10’, ‘x’: ‘x\_10’}  
_****************************MyClass****************************_\_\_\_\_\_\_\_\_\_  
MyClass: <class ‘singleton\_class.MyClass’>, type(MyClass): <class ‘singleton\_class.Singleton’>  
MyClass.class\_attribute(): class\_default  
class function  
arg: 1, id(obj): 39340128  
arg: 1, before: {‘a’: ‘a\_10’, ‘x’: ‘x\_10’}  
arg: 1, after: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, id(obj): 39340128  
arg: 2, before: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, after: {‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
type(obj): <class ‘singleton\_class.MyClass’>  
arg: 3, id(obj): 39340128  
arg: 3, before: {‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
arg: 3, after: {‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
_****************************SubClass****************************_\_\_\_\_\_\_\_\_\_  
SubClass: <class ‘\_\_ main\_\_.SubClass’>, type(SubClass): <class ‘singleton\_class.Singleton’>  
SubClass.class\_attribute(): class\_default  
class function  
arg: 11, id(obj): 39982808  
arg: 11, before: {‘a’: ‘a\_default’, ‘x’: ‘x\_default’, ‘y’: ‘y\_default’}  
arg: 11, after: {‘a’: ‘a\_11’, ‘x’: ‘x\_11’, ‘y’: ‘y\_11’}  
arg: 12, id(obj): 39982808  
arg: 12, before: {‘a’: ‘a\_11’, ‘x’: ‘x\_11’, ‘y’: ‘y\_11’}  
arg: 12, after: {‘a’: ‘a\_12’, ‘x’: ‘x\_12’, ‘y’: ‘y\_12’}  
type(sub\_obj): <class ‘\_\_ main\_\_.SubClass’>  
arg: 13, id(obj): 39982808  
arg: 13, before: {‘a’: ‘a\_12’, ‘x’: ‘x\_12’, ‘y’: ‘y\_12’}  
arg: 13, after: {‘a’: ‘a\_13’, ‘x’: ‘x\_13’, ‘y’: ‘y\_13’}  
优点：

*   将单例模式的代码独立封装，MyClass及其子类SubClass无需关心单例模式的实现
*   MyClass类有继承的基类OtherClass时不影响单例效果
*   单例模式不影响MyClass类作为基类被继承，且子类SubClass也是单例模式，即SubClass每次实例化得到都是同一个SubClass实例
*   变量MyClass代表的是真正的MyClass类，能使用类属性、类方法
*   type(obj) 返回的是真正的MyClass类
*   通过obj3 = type(obj1)() 得到的实例也是单例，单例效果稳定
*   有 lazy loading 效果

缺点：

*   需手动设置双重校验锁，才能在保证多线程并发环境下得到的也是单例

（八）Borg单态模式（所有实例共享状态）
---------------------

上述7种实现方式都是通过确保只有一个实例存在，从而实现单态的，而Borg模式的思路是"实例的唯一性并不是重要的，我们应该关注的是实例的状态，只要所有的实例共享状态，行为一致，那就达到了单例的目的"。通过Borg模式，可以创建任意数量的实例，但因为它们共享状态，从而实现了单态。所以严格来说，Borg模式不是单例模式，而是单态模式。

Borg模式代码的核心是，将所有实例的属性字典都指向同一个内存地址，这样就能实现虽然实例有多个，但属性字典只有一个，从而所有的实例共享状态。

需要注意的是，实例可以有多个，所以\_\_new\_\_方法的调用不限次数；但初始化成员变量只能有一次，所以\_\_init\_\_方法只能调用一次。由于元类的\_\_call\_\_方法能够控制这两个方法的调用，所以采用元类封装Borg模式的代码。

```null
    _instance_lock = threading.Lock()def __init__(cls, *args, **kwargs):        cls._shared_state = dict()super(Singleton, cls).__init__(*args, **kwargs)def __call__(cls, *args, **kwargs):        obj = cls.__new__(cls, *args, **kwargs)        obj.__dict__ = cls._shared_stateif not cls._initialed and obj is not None and isinstance(obj, cls):with Singleton._instance_lock:if not cls._initialed and obj is not None and isinstance(obj, cls):                    obj.__init__(*args, **kwargs)class OtherClass(object):return super(OtherClass, cls).__new__(cls)super(OtherClass, self).__init__()class MyClass(OtherClass, metaclass=Singleton):    class_attribute = 'class_attribute'super(MyClass, self).__init__()```

输出：  
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ MyClass MultiThreading Test\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  
arg: 1, id(obj): 40301288  
arg: 1, before: {‘a’: ‘a\_default’, ‘x’: ‘x\_default’}  
arg: 1, after: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, id(obj): 40302072  
arg: 2, before: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 3, id(obj): 40302296  
arg: 3, before: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 3, after: {‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
arg: 4, id(obj): 40302520  
arg: 5, id(obj): 40302744  
arg: 5, before: {‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
arg: 4, before: {‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
arg: 6, id(obj): 40302968  
arg: 4, after: {‘a’: ‘a\_4’, ‘x’: ‘x\_4’}  
arg: 6, before: {‘a’: ‘a\_4’, ‘x’: ‘x\_4’}  
arg: 6, after: {‘a’: ‘a\_6’, ‘x’: ‘x\_6’}  
arg: 5, after: {‘a’: ‘a\_5’, ‘x’: ‘x\_5’}  
arg: 7, id(obj): 40303192  
arg: 2, after: {‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
arg: 7, before: {‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
arg: 7, after: {‘a’: ‘a\_7’, ‘x’: ‘x\_7’}  
arg: 8, id(obj): 40303416  
arg: 8, before: {‘a’: ‘a\_7’, ‘x’: ‘x\_7’}  
arg: 8, after: {‘a’: ‘a\_8’, ‘x’: ‘x\_8’}  
arg: 9, id(obj): 40303640  
arg: 9, before: {‘a’: ‘a\_8’, ‘x’: ‘x\_8’}  
arg: 9, after: {‘a’: ‘a\_9’, ‘x’: ‘x\_9’}  
arg: 10, id(obj): 40303864  
arg: 10, before: {‘a’: ‘a\_9’, ‘x’: ‘x\_9’}  
arg: 10, after: {‘a’: ‘a\_10’, ‘x’: ‘x\_10’}  
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ MyClass\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  
MyClass: <class ‘singleton\_class.MyClass’>, type(MyClass): <class ‘singleton\_class.Singleton’>  
MyClass.class\_attribute(): class\_attribute  
class function  
arg: 1, id(obj): 39514800  
arg: 1, before: {‘a’: ‘a\_10’, ‘x’: ‘x\_10’}  
arg: 1, after: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, id(obj): 39945160  
arg: 2, before: {‘a’: ‘a\_1’, ‘x’: ‘x\_1’}  
arg: 2, after: {‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
type(obj): <class ‘singleton\_class.MyClass’>  
arg: 3, id(obj): 39667864  
arg: 3, before: {‘a’: ‘a\_2’, ‘x’: ‘x\_2’}  
arg: 3, after: {‘a’: ‘a\_3’, ‘x’: ‘x\_3’}  
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ SubClass\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  
SubClass: <class ‘\_\_ main\_\_.SubClass’>, type(SubClass): <class ‘singleton\_class.Singleton’>  
SubClass.class\_attribute(): class\_attribute  
class function  
arg: 11, id(obj): 40303584  
arg: 11, before: {‘a’: ‘a\_default’, ‘x’: ‘x\_default’, ‘y’: ‘y\_default’}  
arg: 11, after: {‘a’: ‘a\_11’, ‘x’: ‘x\_11’, ‘y’: ‘y\_11’}  
arg: 12, id(obj): 40303360  
arg: 12, before: {‘a’: ‘a\_11’, ‘x’: ‘x\_11’, ‘y’: ‘y\_11’}  
arg: 12, after: {‘a’: ‘a\_12’, ‘x’: ‘x\_12’, ‘y’: ‘y\_12’}  
type(sub\_obj): <class ‘\_\_ main\_\_.SubClass’>  
arg: 13, id(obj): 40303248  
arg: 13, before: {‘a’: ‘a\_12’, ‘x’: ‘x\_12’, ‘y’: ‘y\_12’}  
arg: 13, after: {‘a’: ‘a\_13’, ‘x’: ‘x\_13’, ‘y’: ‘y\_13’}

优点：

*   将单态模式的代码独立封装，MyClass及其子类SubClass无需关心单态模式的实现
*   MyClass类有继承的基类OtherClass时不影响单态效果
*   单态模式不影响MyClass类作为基类被继承，且子类SubClass也是单态模式，即SubClass的所有实例都共享SubClass的单态
*   变量MyClass代表的是真正的MyClass类，能使用类属性、类方法
*   type(obj) 返回的是真正的MyClass类
*   通过obj3 = type(obj1)() 得到的实例也是单态
*   有 lazy loading 效果

缺点：

*   虽然所有实例的属性都指向同一个内存地址，但每个实例本身都分配了一个内存地址，这也是一种资源浪费
*   需手动设置双重校验锁，才能在保证多线程并发环境下得到的也是单态

参考文档
----

1.  [理解 Python对象实例化](https://liqiang.io/post/understanding-python-class-instantiation)
2.  [stackoverflow：Python method-wrapper type?](https://stackoverflow.com/questions/10401935/python-method-wrapper-type)
3.  [类和对象的创建过程（元类，\_\_ new\_\_,\_\_ init\_\_,\_\_ call\_\_）](https://www.cnblogs.com/huchong/p/8260151.html#_label1)
4.  [类的继承、元类](https://bbs.csdn.net/topics/390510751)
5.  [stackoverflow：is-there-a-simple-elegant-way-to-define-singletons](https://stackoverflow.com/questions/31875/is-there-a-simple-elegant-way-to-define-singletons/31887#31887)
6.  [Python中的单例模式的几种实现方式的及优化](https://www.cnblogs.com/huchong/p/8244279.html)
7.  [5种Python单例模式的实现方式](https://www.jb51.net/article/78063.htm)
8.  [Python单体模式的几种常见实现方法详解](https://www.jb51.net/article/119809.htm)
9.  [Python 编程，应该养成哪些好的习惯？](https://www.zhihu.com/question/28966220/answer/130285330)
10.  [Python单例模式(Singleton)的N种实现](https://segmentfault.com/a/1190000016497271)
11.  [单例模式的常见应用场景](https://www.cnblogs.com/gmq-sh/p/5948379.html)
12.  [深入理解设计模式（一）：单例模式](https://www.cnblogs.com/xuwendong/p/9633985.html)
13.  [【Python】闭包的实现原理，如何在内部函数修改外部函数的变量](https://blog.csdn.net/mydistance/article/details/86554904)
14.  [什么是面向对象？为什么要用面向对象编程？](https://blog.csdn.net/u011700168/article/details/79161724)
15.  [怎么从本质上理解面向对象的编程思想？](https://www.zhihu.com/question/305042684/answer/550196442) 
 [https://blog.csdn.net/bat67/article/details/108956889](https://blog.csdn.net/bat67/article/details/108956889)
````
