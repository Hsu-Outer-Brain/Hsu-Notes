# 机器学习（19）---XGBoost入门_xgboost应用-CSDN博客
[机器学习（19）---XGBoost 入门\_xgboost 应用 - CSDN 博客](https://blog.csdn.net/m0_62881487/article/details/133170247?spm=1001.2014.3001.5502) 

* * *

## 一、概述

### 1.1 使用 XGBoost 库

 1\. 我们有两种方式使用我们的 xgboost 库。第一种方式是直接使用 xgboost 库自己的建模流程：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/01078180-388b-4aa9-a2db-67beb27a93b8.png?raw=true)

 2\. 其中最核心的，是`DMtarix()`这个读取数据的类，以及`train()`这个用于训练的类。与[sklearn](https://so.csdn.net/so/search?q=sklearn&spm=1001.2101.3001.7020)把所有的参数都写在类中的方式不同，xgboost 库中必须先使用字典设定参数集，再使用`train()`来将参数集输入，然后进行训练。会这样设计的原因，是因为 XGB 所涉及到的参数实在太多，全部写在 xgb.train() 中太长也容易出错。`params`可能的取值以及`xgboost.train`的列表如下：

```
`params {eta, gamma, max_depth, min_child_weight, max_delta_step, subsample, colsample_bytree,
colsample_bylevel, colsample_bynode, lambda, alpha, tree_method string, sketch_eps, scale_pos_weight, updater,
refresh_leaf, process_type, grow_policy, max_leaves, max_bin, predictor, num_parallel_tree}

xgboost.train (params, dtrain, num_boost_round=10, evals=(), obj=None, feval=None, maximize=False,
early_stopping_rounds=None, evals_result=None, verbose_eval=True, xgb_model=None, callbacks=None,
learning_rates=None)` 

*   1
*   2
*   3
*   4
*   5
*   6
*   7


```

 3\. 第二种方式是使用 xgboost 库中的 sklearn 的 API。这是说，我们可以调用如下的类，并用我们 sklearn 当中惯例的实例化、fit 和 predict 的流程来运行 XGB，并且也可以调用属性比如 coef\_等等。当然，这是我们回归的类，我们也有用于分类，用于排序的类。他们与回归的类非常相似，因此了解一个类即可。

```
`class xgboost.XGBRegressor (max_depth=3, learning_rate=0.1, n_estimators=100, silent=True,
objective='reg:linear', booster='gbtree', n_jobs=1, nthread=None, gamma=0, min_child_weight=1, max_delta_step=0,
subsample=1, colsample_bytree=1, colsample_bylevel=1, reg_alpha=0, reg_lambda=1, scale_pos_weight=1,
base_score=0.5, random_state=0, seed=None, missing=None, importance_type='gain', **kwargs)` 

*   1
*   2
*   3
*   4


```

 4\. 调用 xgboost.train 和调用 sklearnAPI 中的类 XGBRegressor，需要输入的参数是不同的，而且看起来相当的不同。但其实，这些参数只是写法不同，功能是相同的。比如说，我们的[params](https://so.csdn.net/so/search?q=params&spm=1001.2101.3001.7020)字典中的第一个参数 eta，其实就是我们 XGBRegressor 里面的参数 learning_rate，他们的含义和实现的功能是一模一样的。所以对我们来说，使用 xgboost 中设定的建模流程来建模，和使用 sklearnAPI 中的类来建模，模型效果是比较相似的，但是 xgboost 库本身的运算速度（尤其是交叉验证）以及调参手段比 sklearn 要简单。

### 1.2 XGBoost 的三大板块

 xgboost 本身的核心是基于梯度提升树实现的集成算法，整体来说可以有三个核心部分：集成算法本身，用于集成的弱评估器，以及应用中的其他过程。三个部分中，前两个部分包含了 XGBoost 的核心原理以及数学过程，最后的部分主要是在 XGBoost 应用中占有一席之地。

![](https://i-blog.csdnimg.cn/blog_migrate/444967920051d2a0996f70501206b760.png#pic_center)

## 二、集成算法及重要参数

### 2.1 概述

 1\. 集成算法通过在数据上构建多个弱评估器，汇总所有弱评估器的建模结果，以获取比单个模型更好的回归或分类表现。弱评估器被定义为是表现至少比随机猜测更好的模型，即预测准确率不低于 50% 的任意模型。

 2\. 集成算法中提升法（Boosting）是逐一构建弱评估器，经过多次迭代逐渐累积多个弱评估器的方法。提升法的中最著名的算法包括 Adaboost 和梯度提升树，XGBoost 就是由梯度提升树发展而来的。梯度提升树中可以有回归树也可以有分类树，两者都以 CART 树算法作为主流，XGBoost 背后也是 CART 树，这意味着 XGBoost 中所有的树都是二叉的。

 3\. 我们以梯度提升回归树为例子来了解一下 Boosting 算法。首先，梯度提升回归树是专注于回归的树模型的提升集成模型，其建模过程大致如下：最开始先建立一棵树，然后逐渐迭代，每次迭代过程中都增加一棵树，逐渐形成众多树模型集成的强评估器。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/917894d9-96e7-493d-a6ab-5b430649161d.png?raw=true)

 4\. 对于决策树而言，每个被放入模型的任意一个样本 i 最终都会落到一个叶子节点上。而对于回归树，每个叶子节点上的值是这个叶子节点上所有样本的均值。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/2ba70f4f-7969-426f-8bbb-2e49307c3fc6.png?raw=true)

 对于梯度提升回归树来说，每个样本的预测结果可以表示为所有树上的结果的加权求和：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/815edb17-e969-46b2-9cee-dc24f93b8ec3.png?raw=true)

 其中，K 是树的总数量，k 代表第 k 棵树，rk 是这棵树的权重，hk 表示这棵树上的预测结果。

 5\. XGB 作为 GBDT 的改进，在预测结果上却有所不同。这个叶子权重就是所有在这个叶子节点上的样本在这一棵树上的回归取值，用 fk(xi) 或 w 来表示。假设这个集成模型中总共有 棵决策树，则整个模型在这个样本 i 上给出的预测结果为：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/0fd07dd8-192d-44c6-b033-6a55f7183498.png?raw=true)

> XGB vs GBDT 的一个核心区别：求解预测值的方式不同。GBDT 中预测值是由所有弱分类器上的预测结果的加权求和，其中每个样本上的预测结果就是样本所在的叶子节点的均值。而 XGBT 中的预测值是所有弱分类器上的叶子权重直接求和得到，计算叶子权重是一个复杂的过程。

### 2.2 XGBoost 的简单建模

 1\. 从上式可以看出，在集成中我们需要的考虑的第一件事是我们的超参数 K，即究竟要建多少棵树。

| 参数含义           | xgb.train()     | xgb.XGBRegressor()  |
| -------------- | --------------- | ------------------- |
| 集成中弱评估器的数量     | num_round，默认 10 | n_estimators，默认 100 |
| 训练中是否打印每次训练的结果 | slient，默认 False | slient，默认 True      |

 2\. 建模：

```
`from xgboost import XGBRegressor as XGBR
from sklearn.ensemble import RandomForestRegressor as RFR
from sklearn.linear_model import LinearRegression as LinearR
from sklearn.datasets import load_boston
from sklearn.model_selection import KFold, cross_val_score as CVS, train_test_split as TTS
from sklearn.metrics import mean_squared_error as MSE
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from time import time
import datetime
data = load_boston()

X = data.data
y = data.target
Xtrain,Xtest,Ytrain,Ytest = TTS(X,y,test_size=0.3,random_state=420)

def plot_learning_curve(estimator,title, X, y,
                        ax=None, 
                        ylim=None, 
                        cv=None, 
                        n_jobs=None 
                       ):

 from sklearn.model_selection import learning_curve
 import matplotlib.pyplot as plt
 import numpy as np

 train_sizes, train_scores, test_scores = learning_curve(estimator, X, y
                                                          ,shuffle=True
                                                          ,cv=cv
                                                          ,random_state=420
                                                          ,n_jobs=n_jobs)
 if ax == None:
    ax = plt.gca()
 else:
    ax = plt.figure()
 ax.set_title(title)
 if ylim is not None:
    ax.set_ylim(*ylim)
 ax.set_xlabel("Training examples")
 ax.set_ylabel("Score")
 ax.grid() 
 ax.plot(train_sizes, np.mean(train_scores, axis=1), 'o-'
         , color="r",label="Training score")
 ax.plot(train_sizes, np.mean(test_scores, axis=1), 'o-'
         , color="g",label="Test score")
 ax.legend(loc="best")
 return ax

cv = KFold(n_splits=5, shuffle = True, random_state=42)
plot_learning_curve(XGBR(n_estimators=100,random_state=420)
                    ,"XGB",Xtrain,Ytrain,ax=None,cv=cv)
plt.show()` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/5d2b71c9-8cac-4340-b00a-31660d946981.png?raw=true)

*   1
*   2
*   3
*   4
*   5
*   6
*   7
*   8
*   9
*   10
*   11
*   12
*   13
*   14
*   15
*   16
*   17
*   18
*   19
*   20
*   21
*   22
*   23
*   24
*   25
*   26
*   27
*   28
*   29
*   30
*   31
*   32
*   33
*   34
*   35
*   36
*   37
*   38
*   39
*   40
*   41
*   42
*   43
*   44
*   45
*   46
*   47
*   48
*   49
*   50
*   51
*   52
*   53
*   54


```

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/826b0862-0b71-4b01-90bd-7827269a0248.png?raw=true)

 训练集上的表现展示了模型的学习能力，测试集上的表现展示了模型的泛化能力，通常模型在测试集上的表现不太可能超过训练集。可以通过降低训练集学习能力或提高测试集的泛化能力使得测试集学习曲线与训练集学习曲线相逼近来提高模型效果。

### 2.3 n_estimators 学习曲线

```
`axisx = range(10,1010,50)
rs = []
for i in axisx:
  reg = XGBR(n_estimators=i,random_state=420)
  rs.append(CVS(reg,Xtrain,Ytrain,cv=cv).mean())
print(axisx[rs.index(max(rs))],max(rs))
plt.figure(figsize=(20,5))
plt.plot(axisx,rs,c="red",label="XGB")
plt.legend()
plt.show()` 

*   1
*   2
*   3
*   4
*   5
*   6
*   7
*   8
*   9
*   10


```

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/47783697-0db2-498d-93ec-697478f53efa.png?raw=true)

 在测试中输出交叉验证最优时参数为 60。但是，在绘制学习曲线时，很明显发现在 150 以后基本已经收敛，而过大的 n_estimators，也就是 xgboost 中二叉树的个数，会严重拖慢训练速度。所以我们需要考虑偏差 - 方差困境问题。

### 2.4 方差与泛化误差

 1\. 在[机器学习](https://so.csdn.net/so/search?q=%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0&spm=1001.2101.3001.7020)中，我们用来衡量模型在未知数据上的准确率的指标，叫做泛化误差（Genelization error）。一个集成模型 (f) 在未知数据集 (D) 上的泛化误差 E(f ; D)，由方差 (var)，偏差(bais) 和噪声 (ε) 共同决定。其中偏差就是训练集上的拟合程度决定，方差是模型的稳定性决定，噪音是不可控的。而泛化误差越小，模型就越理想。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/118943b8-8704-496d-bdb7-fddc291cb1bb.png?raw=true)

 2\. 在过去我们往往直接取学习曲线获得的分数的最高点，即考虑偏差最小的点，是因为模型极度不稳定，方差很大的情况其实比较少见。但现在我们的数据量非常少，模型会相对不稳定，因此我们应当将方差也纳入考虑的范围。在绘制学习曲线时，我们不仅要考虑偏差的大小，还要考虑方差的大小，更要考虑泛化误差中我们可控的部分。

```
`axisx = range(50,1050,50)
rs = []
var = []
ge = []
for i in axisx:
    reg = XGBR(n_estimators=i,random_state=420)
    cvresult = CVS(reg,Xtrain,Ytrain,cv=cv)
    
    rs.append(cvresult.mean())
    
    var.append(cvresult.var())
    
    ge.append((1 - cvresult.mean())**2+cvresult.var())

print(axisx[rs.index(max(rs))],max(rs),var[rs.index(max(rs))])

print(axisx[var.index(min(var))],rs[var.index(min(var))],min(var))

print(axisx[ge.index(min(ge))],rs[ge.index(min(ge))],var[ge.index(min(ge))],min(ge))
plt.figure(figsize=(20,5))
plt.plot(axisx,rs,c="red",label="XGB")
plt.legend()
plt.show()` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/5894a778-e120-4837-adb6-73ab4f7fe671.png?raw=true)

*   1
*   2
*   3
*   4
*   5
*   6
*   7
*   8
*   9
*   10
*   11
*   12
*   13
*   14
*   15
*   16
*   17
*   18
*   19
*   20
*   21
*   22
*   23


```

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/382e092d-b32d-4029-88fa-5979560b99b7.png?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/310d9e16-3f1d-478d-a4ac-e608a3f6c15c.png?raw=true)

### 2.5 重要参数 subsample

 1\. 确认了有多少棵树之后，我们来思考一个问题：建立了众多的树，怎么就能够保证模型整体的效果变强呢？集成的目的是为了模型在样本上能表现出更好的效果，所以对于所有的提升集成算法，每构建一个评估器，集成模型的效果都会比之前更好。也就是随着迭代的进行，模型整体的效果必须要逐渐提升，最后要实现集成模型的效果最优。要实现这个目标，我们可以首先从训练数据上着手。

 2\. 我们训练模型之前，必然会有一个巨大的数据集。我们都知道树模型是天生过拟合的模型，并且如果数据量太过巨大，树模型的计算会非常缓慢，因此，我们要对我们的原始数据集进行有放回抽样（bootstrap）。有放回的抽样每次只能抽取一个样本，若我们需要总共 N 个样本，就需要抽取 N 次。每次抽取一个样本的过程是独立的，这一次被抽到的样本会被放回数据集中，下一次还可能被抽到，因此抽出的数据集中，可能有一些重复的数据。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/fecf9db4-3563-4774-9b31-ec2e4169cdcb.png?raw=true)

 3\. 在梯度提升树中，我们每一次迭代都要建立一棵新的树，因此我们每次迭代中，都要有放回抽取一个新的训练样本。不过，这并不能保证每次建新树后，集成的效果都比之前要好。因此我们规定，在梯度提升树中，每构建一个评估器，都让模型更加集中于数据集中容易被判错的那些样本。来看看下面的这个过程：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/df2143e7-a529-4000-8f12-a88ae6a5b7d1.png?raw=true)

 4\. 在 sklearn 中，我们使用参数 subsample 来控制我们的随机抽样。在 xgb 和 sklearn 中，这个参数都默认为 1 且不能取到 0，这说明我们无法控制模型是否进行随机有放回抽样，只能控制抽样抽出来的样本量大概是多少。

| 参数含义                    | xgb.train()    | xgb.XGBRegressor() |
| ----------------------- | -------------- | ------------------ |
| 随机抽样的时候抽取的样本比例，范围 (0,1] | subsample，默认 1 | subsample，默认 1     |

### 2.6 迭代决策树：重要参数 eta

 1\. 从数据的角度而言，我们让模型更加倾向于努力攻克那些难以判断的样本。但是，并不是说只要我新建了一棵倾向于困难样本的决策树，它就能够帮我把困难样本判断正确了。困难样本被加重权重是因为前面的树没能把它判断正确，所以对于下一棵树来说，它要判断的测试集的难度，是比之前的树所遇到的数据的难度都要高的，那要把这些样本都判断正确，会越来越难。如果新建的树在判断困难样本这件事上还没有前面的树做得好呢？如果我新建的树刚好是一棵特别糟糕的树呢？所以，除了保证模型逐渐倾向于困难样本的方向，我们还必须控制新弱分类器的生成，我们必须保证，每次新添加的树一定得是对这个新数据集预测效果最优的那一棵树。

 2\. 我们首先找到一个损失函数，这个损失函数应该可以通过带入我们的预测结果 来衡量我们的梯度提升树在样本的预测效果。然后，我们利用梯度下降来迭代我们的集成算法：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/e53d982f-36a1-442f-be04-5e28e2135529.png?raw=true)

 3\. 在逻辑回归中，我们自定义学习率α来干涉我们的迭代速率，在 XGB 中看起来却没有这样的设置，但其实不然。在 XGB 中，我们完整的迭代决策树的公式应该写作：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/7f3d6bd5-1e6a-41c9-ae24-e2c19ec1498e.png?raw=true)

 其中η读作 "eta"，是迭代决策树时的步长（shrinkage），又叫做学习率（learning rate）。和逻辑回归中的α类似，η越大，迭代的速度越快，算法的极限很快被达到，有可能无法收敛到真正的最佳。 越小，越有可能找到更精确的最佳值，更多的空间被留给了后面建立的树，但迭代速度会比较缓慢。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/62ebd6fe-604e-44e5-a251-92f6be898247.png?raw=true)

 4\. 在 sklearn 中，我们使用参数 learning_rate 来干涉我们的学习速率：

| 参数含义                          | xgb.train()           | xgb.XGBRegressor()              |
| ----------------------------- | --------------------- | ------------------------------- |
| 集成中的学习率，又称为步长以控制迭代速率，常用于防止过拟合 | eta，默认 0.3，取值范围\[0,1] | learning_rate，默认 0.1，取值范围\[0,1] |

## 三、XGBoost 的智慧

### 3.1 概述

 1\. 模块：`class xgboost.XGBRegressor (kwargs，max_depth=3, learning_rate=0.1, n_estimators=100, silent=True, objective='reg:linear', booster='gbtree', n_jobs=1, nthread=None, gamma=0, min_child_weight=1, max_delta_step=0, subsample=1, colsample_bytree=1, colsample_bylevel=1, reg_alpha=0, reg_lambda=1,scale_pos_weight=1, base_score=0.5, random_state=0, seed=None, missing=None, importance_type='gain')`。

 2\. 梯度提升算法中不只有梯度提升树，XGB 作为梯度提升算法的进化，自然也不只有树模型一种弱评估器。在 XGB 中，除了树模型，我们还可以选用线性模型，比如线性回归，来进行集成。虽然主流的 XGB 依然是树模型，但我们也可以使用其他的模型。

| xgb.train() & params                                                                                         | xgb.XGBRegressor()                                                                                                                                                       |
| ------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| xgb_model                                                                                                    | booster                                                                                                                                                                  |
| 使用哪种弱评估器。可以输入 gbtree，gblinear 或 dart。输入的评估器不同，使用的 params 参数也不同，每种评估器都有自己的 params 列表。评估器必须于 param 参数相匹配，否则报错。 | 使用哪种弱评估器。可以输入 gbtree，gblinear 或 dart。gbtree 代表梯度提升树，dart 是 Dropouts meet Multiple Additive Regression Trees，可译为抛弃提升树，在建树的过程中会抛弃一部分树，比梯度提升树有更好的防过拟合功能。输入 gblinear 使用线性模型。 |

 注：两个参数都默认为 "gbtree"，如果不想使用树模型，则可以自行调整。

### 3.2 XGB 的目标函数：重要参数 objective

 1\. 梯度提升算法中都存在着损失函数。不同于逻辑回归和 SVM 等算法中固定的损失函数写法，集成算法中的损失函数是可选的，要选用什么损失函数取决于我们希望解决什么问题，以及希望使用怎样的模型。比如说，如果我们的目标是进行回归预测，那我们可以选择调节后的均方误差 RMSE 作为我们的损失函数。如果我们是进行分类预测，那我们可以选择错误率 error 或者对数损失 log_loss。只要我们选出的函数是一个可微的，能够代表某种损失的函数，它就可以是我们 XGB 中的损失函数。

 2\. 在众多机器学习算法中，损失函数的核心是衡量模型的泛化能力，即模型在未知数据上的预测的准确与否，我们训练模型的核心目标也是希望模型能够预测准确。在 XGB 中，预测准确自然是非常重要的因素，但我们之前提到过，XGB 是实现了模型表现和运算速度的平衡的算法。普通的损失函数，比如错误率，均方误差等，都只能够衡量模型的表现，无法衡量模型的运算速度。回忆一下，我们曾在许多模型中使用空间复杂度和时间复杂度来衡量模型的运算效率。XGB 因此引入了模型复杂度来衡量算法的运算效率。因此 XGB 的目标函数被写作：传统损失函数 + 模型复杂度。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/eecdbb35-2977-4e22-a196-97a78c26bc2c.png?raw=true)

其中 i 代表数据集中的第 i 个样本，m 表示导入第 k 棵树的数据总量，K 代表建立的所有树 (n_estimators)

 3\. 第一项是衡量我们的偏差，模型越不准确，第一项就会越大。第二项是衡量我们的方差，模型越复杂，模型的学习就会越具体，到不同数据集上的表现就会差异巨大，方差就会越大。所以我们求解 Obj 的最小值，其实是在求解方差与偏差的平衡点，以求模型的泛化误差最小，运行速度最快。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/1ba933f5-b93f-4d7c-a540-99f71a99037c.png?raw=true)

 4\. XGBoost 的损失函数中自带限制方差变大的部分，也就是说 XGBoost 会比其他的树模型更加聪明，不会轻易落到图像的右上方。在应用中，我们使用参数 “objective" 来确定我们目标函数的第一部分 L，也就是衡量损失的部分。

| xgb.train()            | xgb.XGBRegressor()      | xgb.XGBClassifier()          |
| ---------------------- | ----------------------- | ---------------------------- |
| obj：默认 binary:logistic | objective：默认 reg:linear | objective：默认 binary:logistic |

 常用的选择有：

| 输入              | 选用的损失函数                          |
| --------------- | -------------------------------- |
| reg:linear      | 使用线性回归的损失函数，均方误差，回归时使用           |
| binary:logistic | 使用逻辑回归的损失函数，对数损失 log_loss，二分类时使用 |
| binary:hinge    | 使用支持向量机的损失函数，Hinge Loss，二分类时使用   |
| multi:softmax   | 使用 softmax 损失函数，多分类时使用           |

 5\. 在 xgboost 中，我们被允许自定义损失函数，但通常我们还是使用类已经为我们设置好的损失函数。我们的回归类中本来使用的就是 reg:linear，因此在这里无需做任何调整。注意：分类型的目标函数导入回归类中会直接报错。来试试看 xgb 自身的调用方式：  
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/aa312bb7-cbfe-4a9d-aea0-ae9929958e36.png?raw=true)

 6\. 由于 xgb 中所有的参数都需要自己的输入，并且 objective 参数的默认值是二分类，因此我们必  
其他参数相同的情况下，我们 xgboost 库本身和 sklearn 比起来，效果如何：

```
 `reg = XGBR(n_estimators=180,random_state=420).fit(Xtrain,Ytrain)
reg.score(Xtest,Ytest)
MSE(Ytest,reg.predict(Xtest))

import xgboost as xgb

dtrain = xgb.DMatrix(Xtrain,Ytrain)
dtest = xgb.DMatrix(Xtest,Ytest)

dtrain

param = {'silent':False,'objective':'reg:linear',"eta":0.1}
num_round = 180

bst = xgb.train(param, dtrain, num_round)

from sklearn.metrics import r2_score
r2_score(Ytest,bst.predict(dtest))
MSE(Ytest,bst.predict(dtest))` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2024-8-29%2010-29-10/cf08387d-62ee-4737-a163-38dcbf3f21ed.png?raw=true)

*   1
*   2
*   3
*   4
*   5
*   6
*   7
*   8
*   9
*   10
*   11
*   12
*   13
*   14
*   15
*   16
*   17
*   18
*   19
*   20


```

 看得出来，无论是从 R^2 还是从 MSE 的角度来看，都是 xgb 库本身表现更优秀！
