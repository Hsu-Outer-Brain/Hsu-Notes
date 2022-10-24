# 揭开 Python 对象神秘的面纱 | Python源码剖析
[揭开 Python 对象神秘的面纱 | Python 源码剖析](https://fasionchan.com/python-source/object-model/pyobject/) 

 面向对象理论中 “ **类** ” 和 “ **对象** ” 这两个重要概念，在 _Python_ 内部均以对象的形式存在。 “类” 是一种对象，称为 **类型对象** ；“类”实例化生成的 “对象” 也是对象，称为 **实例对象** 。

根据对象不同特点还可进一步分类：

| **类别** | **特点**    |
| ------ | --------- |
| 可变对象   | 对象创建后可以修改 |
| 不可变对象  | 对象创建后不能修改 |
| 定长对象   | 对象大小固定    |
| 变长对象   | 对象大小不固定   |

那么，对象在 _Python_ 内部到底长啥样呢？

由于 _Python_ 是由 _C_ 语言实现的，因此 _Python_ 对象在 _C_ 语言层面应该是一个 **结构体** ，组织对象占用的内存。 不同类型的对象，数据及行为均可能不同，因此可以大胆猜测：不同类型的对象由不同的结构体表示。

对象也有一些共性，比如每个对象都需要有一个 **引用计数** ，用于实现 **垃圾回收机制** 。 因此，还可以进一步猜测：表示对象的结构体有一个 **公共头部** 。

到底是不是这样呢？—— 接下来在源码中窥探一二。

## PyObject，对象的基石

在 _Python_ 内部，对象都由 _PyObject_ 结构体表示，对象引用则是指针 _PyObject_ \* 。 _PyObject_ 结构体定义于头文件 _object.h_ ，路径为 _Include/object.h_ ，代码如下：

\| 

```
1
2
3
4
5

```

 \| 

```c
typedef struct _object {
    _PyObject_HEAD_EXTRA
    Py_ssize_t ob_refcnt;
    struct _typeobject *ob_type;
} PyObject;

```

 \|

除了 _\_PyObject_HEAD_EXTRA_ 宏，结构体包含以下两个字段：

-   **引用计数** ( ob_refcnt )
-   **类型指针** ( ob_type )

**引用计数** 很好理解：对象被其他地方引用时加一，引用解除时减一； 当引用计数为零，便可将对象回收，这是最简单的垃圾回收机制。 **类型指针** 指向对象的 **类型对象** ，**类型对象** 描述 **实例对象** 的数据及行为。

回过头来看 _\_PyObject_HEAD_EXTRA_ 宏的定义，同样在 _Include/object.h_ 头文件内：

\| 

```
 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12

```

 \| 

```c
#ifdef Py_TRACE_REFS
/* Define pointers to support a doubly-linked list of all live heap objects. */
#define _PyObject_HEAD_EXTRA            \
    struct _object *_ob_next;           \
    struct _object *_ob_prev;

#define _PyObject_EXTRA_INIT 0, 0,

#else
#define _PyObject_HEAD_EXTRA
#define _PyObject_EXTRA_INIT
#endif

```

 \|

如果 _Py_TRACE_REFS_ 有定义，宏展开为两个指针，看名字是用来实现 **双向链表** 的：

\| 

```
1
2

```

 \| 

```c
struct _object *_ob_next;
struct _object *_ob_prev;

```

 \|

结合注释，双向链表用于跟踪所有 **活跃堆对象** ，一般不启用，不深入介绍。

对于 **变长对象** ，需要在 _PyObject_ 基础上加入长度信息，这就是 _PyVarObject_ ：

\| 

```
1
2
3
4

```

 \| 

```c
typedef struct {
    PyObject ob_base;
    Py_ssize_t ob_size; /* Number of items in variable part */
} PyVarObject;

```

 \|

变长对象比普通对象多一个字段 _ob_size_ ，用于记录元素个数：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-10-24%2016-59-26/7289dda0-2519-494f-9af3-f46554e63afe.png?raw=true)

至于具体对象，视其大小是否固定，需要包含头部 _PyObject_ 或 _PyVarObject_ 。 为此，头文件准备了两个宏定义，方便其他对象使用：

\| 

```
1
2

```

 \| 

```c
#define PyObject_HEAD          PyObject ob_base;
#define PyObject_VAR_HEAD      PyVarObject ob_base;

```

 \|

例如，对于大小固定的 **浮点对象** ，只需在 _PyObject_ 头部基础上， 用一个 **双精度浮点数** _double_ 加以实现：

\| 

```
1
2
3
4
5

```

 \| 

```c
typedef struct {
    PyObject_HEAD

    double ob_fval;
} PyFloatObject;

```

 \|

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-10-24%2016-59-26/2120e945-556a-4395-9ea6-4ddff9c9ebee.png?raw=true)

而对于大小不固定的 **列表对象** ，则需要在 _PyVarObject_ 头部基础上， 用一个动态数组加以实现，数组存储列表包含的对象，即 _PyObject_ 指针：

\| 

```
1
2
3
4
5
6

```

 \| 

```c
typedef struct {
    PyObject_VAR_HEAD

    PyObject **ob_item;
    Py_ssize_t allocated;
} PyListObject;

```

 \|

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-10-24%2016-59-26/aa199464-ac97-4b8c-9bdd-81ca7817344f.png?raw=true)

如图， _PyListObject_ 底层由一个数组实现，关键字段是以下 _3_ 个：

-   _ob_item_ ，指向 **动态数组** 的指针，数组保存元素对象指针；
-   _allocated_ ，动态数组总长度，即列表当前的 **容量** ；
-   _ob_size_ ，当前元素个数，即列表当前的 **长度** ；

列表容量不足时， _Python_ 会自动扩容，具体做法在讲解 _list_ 源码时再详细介绍。

最后，介绍两个用于初始化对象头部的宏定义。 其中，_PyObject_HEAD_INIT_ 一般用于 **定长对象** ，将引用计数 _ob_refcnt_ 设置为 _1_ 并将对象类型 _ob_type_ 设置成给定类型：

\| 

```
1
2
3

```

 \| 

```c
#define PyObject_HEAD_INIT(type)        \
    { _PyObject_EXTRA_INIT              \
    1, type },

```

 \|

_PyVarObject_HEAD_INIT_ 在 _PyObject_HEAD_INIT_ 基础上进一步设置 **长度字段** _ob_size_ ，一般用于 **变长对象** ：

\| 

```
1
2

```

 \| 

```c
#define PyVarObject_HEAD_INIT(type, size)       \
    { PyObject_HEAD_INIT(type) size },

```

 \|

后续在研读源码过程中，将经常见到这两个宏定义。

## PyTypeObject，类型的基石

在 _PyObject_ 结构体，我们看到了 _Python_ 中所有对象共有的信息。

对于内存中的任一个对象，不管是何类型，它刚开始几个字段肯定符合我们的预期： **引用计数** 、 **类型指针** 以及变长对象特有的 **元素个数** 。

随着研究不断深入，我们发现有一些棘手的问题没法回答：

-   不同类型的对象所需内存空间不同，创建对象时从哪得知内存信息呢？
-   对于给定对象，怎么判断它支持什么操作呢？

对于我们初步解读过的 _PyFloatObject_ 和 _PyListObject_ ，并不包括这些信息。 事实上，这些作为对象的 **元信息** ，应该由一个独立实体保存，与对象所属 **类型** 密切相关。

注意到， _PyObject_ 中包含一个指针 _ob_type_ ，指向一个 **类型对象** ，秘密就藏在这里。类型对象 _PyTypeObject_ 也在 _Include/object.h_ 中定义，字段较多，只讨论关键部分：

\| 

```
 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
18

```

 \| 

```c
typedef struct _typeobject {
    PyObject_VAR_HEAD
    const char *tp_name; /* For printing, in format "<module>.<name>" */
    Py_ssize_t tp_basicsize, tp_itemsize; /* For allocation */

    /* Methods to implement standard operations */
    destructor tp_dealloc;
    printfunc tp_print;

    getattrfunc tp_getattr;
    setattrfunc tp_setattr;

    // ...
    /* Attribute descriptor and subclassing stuff */
    struct _typeobject *tp_base;

    // ......
} PyTypeObject;

```

 \|

可见 **类型对象** _PyTypeObject_ 是一个 **变长对象** ，包含变长对象头部。

专有字段有：

-   **类型名称** ，即 _tp_name_ 字段；
-   类型的继承信息，例如 _tp_base_ 字段指向基类对象；
-   创建实例对象时所需的 **内存信息** ，即 _bp_basicsize_ 和 _tp_itemsize_ 字段；
-   该类型支持的相关 **操作信息** ，即 _tp_print_ 、 _tp_getattr_ 等函数指针；

_PyTypeObject_ 就是 **类型对象** 在 _Python_ 中的表现形式，对应着面向对象中 “**类**” 的概念。 _PyTypeObject_ 结构很复杂，但是我们不必在此刻完全弄懂它。 先有个大概的印象，知道 _PyTypeObject_ 保存着对象的 **元信息** ，描述对象的 **类型** 即可。

接下来，以 **浮点** 为例，考察 **类型对象** 和 **实例对象** 在内存中的形态和关系：

\| 

```
1
2
3
4
5
6

```

 \| 

```python
>>> float
<class 'float'>
>>> pi = 3.14
>>> e = 2.71
>>> type(pi) is float
True

```

 \|

_float_ 为浮点类型对象，系统中只有唯一一个，保存了所有浮点实例对象的元信息。 而浮点实例对象就有很多了，圆周率 _pi_ 是一个，自然对数 _e_ 是另一个，当然还有其他。

代码中各个对象在内存的形式如下图所示：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-10-24%2016-59-26/4a1465bd-a0e1-48f8-abf4-b79e9ab78d26.png?raw=true)

其中，两个浮点 **实例对象** 都是 _PyFloatObject_ 结构体， 除了公共头部字段 _ob_refcnt_ 和 _ob_type_ ，专有字段 _ob_fval_ 保存了对应的数值。

浮点 **类型对象** 是一个 _PyTypeObject_ 结构体，保存了类型名、内存分配信息以及浮点相关操作。

实例对象 _ob_type_ 字段指向类型对象， _Python_ 据此判断对象类型， 进而获悉关于对象的元信息，如操作方法等。 再次提一遍，_float_ 、_pi_ 以及 _e_ 等变量只是一个指向实际对象的指针。

由于浮点 **类型对象** 全局唯一，在 _C_ 语言层面作为一个全局变量静态定义即可，_Python_ 的确就这么做。 浮点类型对象就藏身于 _Object/floatobject.c_ 中， _PyFloat_Type_ 是也：

\| 

```
 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12

```

 \| 

```c
PyTypeObject PyFloat_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "float",
    sizeof(PyFloatObject),
    0,
    (destructor)float_dealloc,                  /* tp_dealloc */

    // ...
    (reprfunc)float_repr,                       /* tp_repr */

    // ...
};

```

 \|

其中，第 _2_ 行初始化 _ob_refcnt_ 、 _ob_type_ 以及 _ob_size_ 三个字段； 第 _3_ 行将 _tp_name_ 字段初始化成类型名称 _float_ ；再往下是各种操作的函数指针。

注意到 _ob_type_ 指针指向 _PyType_Type_ ，这也是一个静态定义的全局变量。 由此可见，代表 “ **类型的类型** ” 即 _type_ 的那个对象应该就是 _PyType_Type_ 了。

## PyType_Type，类型的类型

我们初步考察了 _float_ 类型对象，知道它在 _C_ 语言层面是 _PyFloat_Type_ 全局静态变量。 类型是一种对象，它也有自己的类型，也就是 _Python_ 中的 _type_ ：

\| 

```
1
2

```

 \| 

```python
>>> float.__class__
<class 'type'>

```

 \|

自定义类型也是如此：

\| 

```
1
2
3
4
5

```

 \| 

```python
>>> class Foo(object):
...     pass
...
>>> Foo.__class__
<class 'type'>

```

 \|

那么， _type_ 在 _C_ 语言层面又长啥样呢？

围观 _PyFloat_Type_ 时，我们通过 _ob_type_ 字段揪住了 _PyType_Type_ 。 的确，它就是 _type_ 的肉身。 _PyType_Type_ 在 _Object/typeobject.c_ 中定义：

\| 

```
 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12

```

 \| 

```c
PyTypeObject PyType_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "type",                                     /* tp_name */
    sizeof(PyHeapTypeObject),                   /* tp_basicsize */
    sizeof(PyMemberDef),                        /* tp_itemsize */
    (destructor)type_dealloc,                   /* tp_dealloc */

    // ...
    (reprfunc)type_repr,                        /* tp_repr */

    // ...
};

```

 \|

内建类型和自定义类对应的 _PyTypeObject_ 对象都是这个通过 _PyType_Type_ 创建的。 _PyType_Type_ 在 _Python_ 的类型机制中是一个至关重要的对象，它是所有类型的类型，称为 **元类型** ( _meta class_ )。 借助元类型，你可以实现很多神奇的高级操作。

注意到， _PyType_Type_ 将自己的 _ob_type_ 字段设置成它自己 (第 _2_ 行)，这跟我们在 _Python_ 中看到的行为是吻合的：

\| 

```
1
2
3
4

```

 \| 

```c
>>> type.__class__
<class 'type'>
>>> type.__class__ is type
True

```

 \|

至此，元类型 _type_ 在对象体系里的位置非常清晰了：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-10-24%2016-59-26/788ffc2b-5a43-471f-80e0-05143bd6fe24.png?raw=true)

## PyBaseObject_Type，类型之基

_object_ 是另一个特殊的类型，它是所有类型的基类。

那么，怎么找到它背后的实体呢？ 理论上，通过 _PyFloat_Type_ 中 _tp_base_ 字段顺藤摸瓜即可。

然而，我们发现这个字段在并没有初始化：

这又是什么鬼？

接着查找代码中 _PyFloat_Type_ 出现的地方，我们在 _Object/object.c_ 发现了蛛丝马迹：

\| 

```
1
2

```

 \| 

```c
if (PyType_Ready(&PyFloat_Type) < 0)
    Py_FatalError("Can't initialize float type");

```

 \|

敢情 _PyFloat_Type_ 静态定义后还是个半成品呀！ _PyType_Ready_ 对它做进一步加工，将 _PyFloat_Type_ 中 _tp_base_ 字段初始化成 _PyBaseObject_Type_ ：

\| 

```
 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13

```

 \| 

```c
int
PyType_Ready(PyTypeObject *type)
{
    // ...

    base = type->tp_base;
    if (base == NULL && type != &PyBaseObject_Type) {
        base = type->tp_base = &PyBaseObject_Type;
        Py_INCREF(base);
    }

    // ...
}

```

 \|

_PyBaseObject_Type_ 就是 _object_ 背后的实体，先一睹其真容：

\| 

```
 1
 2
 3
 4
 5
 6
 7
 8
 9
10

```

 \| 

```c
PyTypeObject PyBaseObject_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "object",                                   /* tp_name */
    sizeof(PyObject),                           /* tp_basicsize */
    0,                                          /* tp_itemsize */
    object_dealloc,                             /* tp_dealloc */

    // ...
    object_repr,                                /* tp_repr */
};

```

 \|

注意到， _ob_type_ 字段指向 _PyType_Type_ 跟 _object_ 在 _Python_ 中的行为时相吻合的：

\| 

```
1
2

```

 \| 

```python
>>> object.__class__
<class 'type'>

```

 \|

又注意到 _PyType_Ready_ 函数初始化 _PyBaseObject_Type_ 时，不设置 _tp_base_ 字段。 因为继承链必须有一个终点，不然对象沿着继承链进行属性查找时便陷入死循环。

\| 

```
1
2

```

 \| 

```python
>>> print(object.__base__)
None

```

 \|

至此，我们完全弄清了 _Python_ 对象体系中的所有实体以及关系，得到一幅完整的图画：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-10-24%2016-59-26/84ea919b-0d6d-4d2e-8693-547deb7c2229.png?raw=true)

虽然很多细节还没来得及研究，这也算是一个里程碑式的胜利！让我们再接再厉！

洞悉 _Python_ 虚拟机运行机制，探索高效程序设计之道！

到底如何才能提升我的 _Python_ 开发水平，向更高一级的岗位迈进呢？ 如果你有这些问题或者疑惑，请订阅我们的专栏 [Python 源码深度剖析](https://www.imooc.com/read/76) ，阅读更多章节：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-10-24%2016-59-26/2bcb9a4f-02d4-4124-8bf2-138956fa20f1.png?raw=true)

【Python 源码剖析】系列文章首发于公众号【小菜学编程】，敬请关注：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-10-24%2016-59-26/76fed3d4-1635-4d9d-9052-4086a2bd8311.png?raw=true)
