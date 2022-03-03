# 风控模型—群体稳定性指标(PSI)深入理解应用 - 知乎
## **风控业务背景**

在风控中，**稳定性压倒一切**。原因在于，一套风控模型正式上线运行后往往需要很久（通常一年以上）才会被替换下线。如果模型不稳定，意味着模型不可控，对于业务本身而言就是一种不确定性风险，直接影响决策的合理性。这是不可接受的。

本文将从稳定性的直观理解、群体稳定性指标（Population Stability Index，PSI）的计算逻辑、PSI 背后的含义等多维度展开分析。

> **目录**  
> Part 1. 稳定性的直观理解  
> Part 2. 群体稳定性指标（Population Stability Index，PSI）的理解  
> Part 3. 相对熵（KL 散度）的理解  
> Part 4. 相对熵与 PSI 之间的关系  
> Part 5. PSI 指标的业务应用  
> Part 6. PSI 的计算代码（Python）  
> 致谢  
> 版权声明  
> 参考资料

## **Part 1. 稳定性的直观理解**

在日常生活中，我们可能会看到每月电表、水表数值的变化。直观理解上的系统稳定，通常是指**某项指标波动小（低方差**），指标曲线几乎是一条水平的直线。此时，我们就会觉得系统运行正常稳定，很有安全感。

在数学上，我们通常可以用**变异系数（**Coefficient of Variation，CV**）**来衡量这种数据波动水平。变异系数越小，代表波动越小，稳定性越好。

> 变异系数的计算公式为：变异系数 C·V =（ 标准偏差 SD / 平均值 Mean ）× 100%

那么，是不是只用用变异系数就可以了呢？方便、直观。——答案是否定的。在机器学习建模时，我们基于假设 “**历史样本分布等于未来样本分布**”。因此，我们通常认为：

> 模型或变量稳定 &lt;=> 未来样本分布与历史样本分布之间的偏差小。

然而，实际中由于受到客群变化（互金市场用户群体变化快）、数据源采集变化（比如爬虫接口被风控了）等等因素影响，实际样本分布将会发生偏移，就会导致模型不稳定。

## **Part 2. 群体稳定性指标（Population Stability Index，PSI）的理解**

如果你有基础的风控建模经验，想必会很熟悉 PSI（Population Stability Index）指标。PSI 反映了**验证样本**在各分数段的分布与**建模样本**分布的稳定性。在建模中，我们常用来**筛选特征变量、评估模型稳定性**。

那么，PSI 的计算逻辑是怎样的呢？很多博客文章都会直接告诉我们，**稳定性是有参照的**，因此**需要有两个分布——实际分布**（actual）**和预期分布**（expected）。其中，在建模时通常以训练样本（In the Sample, INS）作为预期分布，而验证样本通常作为实际分布。验证样本一般包括样本外（Out of Sample，OOS）和跨时间样本（Out of Time，OOT）。

我们从直觉上理解，是不是可以**把两个分布重叠放在一起，比较下两个分布的差异有多大**？

![](https://pic2.zhimg.com/v2-2e0a202197517e22ff21280e324f36ad_b.jpg)

图 1 - 分数分布差异对比

PSI 的计算公式也告诉我们，的确是这样的：

![](https://www.zhihu.com/equation?tex=psi+%3D+%5Csum_%7Bi%3D1%7D%5E%7Bn%7D%7B%28A_i+-+E_i%29%7D+%2A+ln%28A_i+%2F+E_i%29+%5Ctag%7B1%7D)

> **PSI = SUM( (实际占比 - 预期占比）\* ln(实际占比 / 预期占比) )**

大多数同学到了这里，遵循拿来主义，直接应用。但是，大家有没有思考过以下几个问题：

**Q1:** 为什么要在计算公式中**引入对数—— ln(实际占比 / 预期占比)**？

**Q2:** 为什么 log 的底数是自然常数 e，而不是其他常数（比如 2）？（信息论中基常常选择为 2，因此信息的单位为**比特 bits**；而机器学习中基常选择为自然常数，因此单位常被称为**奈特 nats**）

**Q3**: 如果只是衡量两个分布的差异，为什么不直接定义成以下公式呢？这个公式含义是把各个分数段里的人群占比差异进行累加放大。

> **自定义的稳定性指标 = SUM(实际占比 - 预期占比）**

让我们先保留这些疑问，回到 PSI 的计算公式，以图 2 表格数据为例介绍下计算步骤。

-   **step1**：将**变量预期分布**（excepted）进行**分箱**（binning）离散化，统计各个分箱里的样本占比。  
    注意：  
    a) 分箱可以是等频、等距或其他方式，分箱方式不同，将导致计算结果略微有差异；  
    b) 对于**连续型**变量（特征变量、模型分数等），分箱数需要设置合理，一般设为 10 或 20；对于离散型变量，如果分箱太多可以提前考虑合并小分箱；分箱数太多，可能会导致每个分箱内的样本量太少而失去统计意义；分箱数太少，又会导致计算结果精度降低。
-   **step2**: 按相同分箱区间，对**实际分布（actual）**统计各分箱内的样本占比**。** 
-   **step3**: 计 算各分箱内的**A - E**和**Ln(A / E)**，计算**index = (实际占比 - 预期占比）\* ln(实际占比 / 预期占比) 。** 
-   **step4**: 将各分箱的 index 进行求和，即得到最终的 PSI。

![](https://pic3.zhimg.com/v2-41d13c635bdbf91ca084dde9bd2bfe5a_b.jpg)

图 2 - PSI 计算表格

在计算得到 PSI 指标后，这个数字又代表什么业务含义呢？**PSI 数值越小，两个分布之间的差异就越小，代表越稳定。** 

![](https://pic1.zhimg.com/v2-df25e14b6c7cfe4f8bd8bae5533dc5f8_b.png)

图 3 - PSI 数值的业务含义

## **Part 3. 相对熵（KL 散度）的理解**

相对熵（relative entropy），又被称为 Kullback-Leibler 散度（Kullback-Leibler divergence）或信息散度（information divergence），是两个概率分布间差异的**非对称性**度量。

划重点——**KL 散度不满足对称性。** 

我们先不考虑相对熵的计算逻辑，先看它的物理含义是什么？

> 相对熵可以衡量两个随机分布之间的**" 距离 “**。  
> 1）当两个随机分布相同时，它们的**相对熵为零**；当两个随机分布的差别增大时，它们的相对熵也会增大。  
> 2）注意⚠️：相对熵是一个从**信息论角度量化距离**的指标，与**数学概念上的距离**有所差异。数学上的距离需要满足：非负性、**对称性**、同一性、传递性等；而**相对熵不满足对称性**。

看到这里，是不是感觉和 PSI 的概念非常相似？

> 当两个随机分布完全一样时，PSI = 0；反之，差异越大，PSI 越大。

直觉告诉我们**相对熵和 PSI 应该存在着某种隐含的关系，真相正在慢慢浮出水面。** 

我们再来看相对熵的计算公式。**在信息理论中，相对熵等价于两个概率分布的信息熵（Shannon entropy）的差值。** 

![](https://www.zhihu.com/equation?tex=KL%28P%7C%7CQ%29%3D-%5Csum_%7Bx+%5Cin+X%7DP%28x%29log%5Cfrac%7B1%7D%7BP%28x%29%7D+%2B+%5Csum_%7Bx+%5Cin+X%7DP%28x%29log%5Cfrac%7B1%7D%7BQ%28x%29%7D+%5C%5C+%3D+%5Csum_%7Bx+%5Cin+X%7DP%28x%29log%5Cfrac%7BP%28x%29%7D%7BQ%28x%29%7D+%5Ctag%7B2%7D)

其中，P(x) 表示数据的**真实分布**，而 Q(x) 表示数据的**观察分布**。上式可以理解为：

> **概率分布携带着信息，可以用信息熵来衡量。**  
> **若用观察分布 Q(x) 来描述真实分布 P(x)，还需要多少额外的信息量？**

![](https://pic3.zhimg.com/v2-372b29196647230da027de164c915356_b.png)

图 4 - KL 散度具有非对称性

因此，**KL 散度是单向描述信息熵差异**。为了加强大家理解，接下来以数值案例介绍相对熵如何计算。

假如一个字符发射器，随机发出 0 和 1 两种字符，真实发出概率分布为 A，但实际不知道 A 的具体分布。通过观察，得到概率分布 B 与 C，各个分布的具体情况如下：

![](https://pic1.zhimg.com/v2-f82e0dadf2bb5d38f80005f5698f31c4_b.jpg)

图 5 - 三种数据分布

可以计算出得到 KL 散度如下：

![](https://www.zhihu.com/equation?tex=KL%28A%7C%7CB%29%3D1%2F2log%28%5Cfrac%7B1%2F2%7D%7B1%2F4%7D%29%2B1%2F2log%28%5Cfrac%7B1%2F2%7D%7B3%2F4%7D%3D1%2F2log%284%2F3%29+%5C%5C+KL%28A%7C%7CC%29%3D1%2F2log%28%5Cfrac%7B1%2F2%7D%7B1%2F8%7D%29%2B1%2F2log%28%5Cfrac%7B1%2F2%7D%7B7%2F8%7D%3D1%2F2log%2816%2F7%29+%5Ctag%7B3%7D+)

由上式可知，相对熵**KL(A||C) > KL(A||B)**，说明**A 和 B 之间的概率分布在信息量角度更为接近。** 而通过概率分布可视化观察，我们也**认为 A 和 B 更为接近**，两者吻合。

## **Part 4. 相对熵与 PSI 之间的关系**

接下来，我们从数学上来分析相对熵和 PSI 之间的关系。

![](https://www.zhihu.com/equation?tex=psi+%3D+%5Csum_%7Bi%3D1%7D%5E%7Bn%7D%7B%28A_i+-+E_i%29%7D+%2A+ln%28A_i+%2F+E_i%29+%5C%5C+++psi+%3D+%5Csum_%7Bi%3D1%7D%5E%7Bn%7D%7BA_i%7D+%2A+ln%28A_i+%2F+E_i%29++%2B+%5Csum_%7Bi%3D1%7D%5E%7Bn%7D%7BE_i%7D+%2A+ln%28E_i+%2F+A_i%29+%5Ctag%7B4%7D)

将 PSI 计算公式变形后可以分解为 2 项，其中：

> 第 1 项：实际分布（A）与预期分布（E）之间的 KL 散度—— ![](https://www.zhihu.com/equation?tex=KL%28A+%7C%7C+E%29)
>
> 第 2 项：预期分布（E）与实际分布（A）之间的 KL 散度—— ![](https://www.zhihu.com/equation?tex=KL%28E+%7C%7C+A%29)

因此，**PSI 本质上是实际分布（A）与预期分布（E）的 KL 散度的一个对称化操作**。其**双向**计算相对熵，并把两部分相对熵相加，从而更为全面地描述两个分布的差异。

至此，已经回答了上文的问题 Q1——**为什么要在计算公式中引入对数？**

可能有人还会问，那为什么不把 PSI 定义成下式呢？

![](https://www.zhihu.com/equation?tex=%5Cfrac%7B+KL%28A+%7C%7C+E%29+%2B+KL%28E+%7C%7C+A%29+%7D+%7B2%7D+%5Ctag%7B5%7D)

按笔者理解，一是为了保证数学公式的优雅；二是只要在同一尺度上比较，同时放大 2 倍也无所谓。

再回到上文的问题 Q2——**为什么 log 的底数是自然常数 e，而不是其他常数（比如 2）**？按笔者理解，两者都可以，从信息量角度而言，底数为 2 或许更为合理。

至于问题 Q3——**为什么不定义为 “SUM(实际占比 - 预期占比）”？**这是因为有正有负，会存在相互抵消的现象。另一方面，也确实没有从相对熵角度来定义更为合理。

![](https://pic1.zhimg.com/v2-91d772eb1f18088b787b2a3be1c89f74_b.jpg)

图 6 - PSI 与 KL 散度之间的关系

## **Part 5. PSI 指标的业务应用**

我们在业务上一般怎么用呢？一般以训练集（INS）的样本分布作为预期分布，进而跨时间窗按月 / 周来计算 PSI，得到如图 6 所示的 Monthly PSI Report，进而剔除不稳定的变量。同理，在模型上线部署后，也将通过 PSI 曲线报表来观察模型的稳定性。

> 入模变量保证稳定性，变量监控  
> 模型分数保证稳定性，模型监控

根据建模经验，给出一些建议：

1. 实际评估需要分不同粒度：时间粒度（按月、按样本集）、订单层次（放贷层、申请层）、人群（若没有分群建模，可忽略）。

2. 先在放贷样本上计算 PSI，剔除不稳定的特征；再对申请样本抽样（可能数据太大），计算 PSI 再次筛选。之前犯的错误就是只在放贷样本上评估，后来在全量申请订单上评估时发现并不稳定，导致返工。

3. 时间窗尽可能至今为止，有可能建模时间窗稳定，但近期时间窗出现不稳定。

4. PSI 只是一个宏观的指标，建议先看变量数据分布（EDD），看分位数跨时间变化来检验数据质量。我们无法得知 PSI 上升时，数据分布是左偏还是右偏。因此，**建议把 PSI 计算细节也予以保留，便于在模型不稳定时，第一时间排查问题。** 

![](https://pic3.zhimg.com/v2-fa80aca10ddf3ad41773d15f0c759bde_b.jpg)

图 7 - Monthly PSI Report

## Part 6. PSI 的计算代码（Python）

```python
import math
import numpy as np
import pandas as pd

def calculate_psi(base_list, test_list, bins=20, min_sample=10):
    try:
        base_df = pd.DataFrame(base_list, columns=['score'])
        test_df = pd.DataFrame(test_list, columns=['score']) 
        
        # 1.去除缺失值后，统计两个分布的样本量
        base_notnull_cnt = len(list(base_df['score'].dropna()))
        test_notnull_cnt = len(list(test_df['score'].dropna()))

        # 空分箱
        base_null_cnt = len(base_df) - base_notnull_cnt
        test_null_cnt = len(test_df) - test_notnull_cnt
        
        # 2.最小分箱数
        q_list = []
        if type(bins) == int:
            bin_num = min(bins, int(base_notnull_cnt / min_sample))
            q_list = [x / bin_num for x in range(1, bin_num)]
            break_list = []
            for q in q_list:
                bk = base_df['score'].quantile(q)
                break_list.append(bk)
            break_list = sorted(list(set(break_list))) # 去重复后排序
            score_bin_list = [-np.inf] + break_list + [np.inf]
        else:
            score_bin_list = bins
        
        # 4.统计各分箱内的样本量
        base_cnt_list = [base_null_cnt]
        test_cnt_list = [test_null_cnt]
        bucket_list = ["MISSING"]
        for i in range(len(score_bin_list)-1):
            left  = round(score_bin_list[i+0], 4)
            right = round(score_bin_list[i+1], 4)
            bucket_list.append("(" + str(left) + ',' + str(right) + ']')
            
            base_cnt = base_df[(base_df.score > left) & (base_df.score <= right)].shape[0]
            base_cnt_list.append(base_cnt)
            
            test_cnt = test_df[(test_df.score > left) & (test_df.score <= right)].shape[0]
            test_cnt_list.append(test_cnt)
        
        # 5.汇总统计结果    
        stat_df = pd.DataFrame({"bucket": bucket_list, "base_cnt": base_cnt_list, "test_cnt": test_cnt_list})
        stat_df['base_dist'] = stat_df['base_cnt'] / len(base_df)
        stat_df['test_dist'] = stat_df['test_cnt'] / len(test_df)
        
        def sub_psi(row):
            # 6.计算PSI
            base_list = row['base_dist']
            test_dist = row['test_dist']
            # 处理某分箱内样本量为0的情况
            if base_list == 0 and test_dist == 0:
                return 0
            elif base_list == 0 and test_dist > 0:
                base_list = 1 / base_notnull_cnt   
            elif base_list > 0 and test_dist == 0:
                test_dist = 1 / test_notnull_cnt
                
            return (test_dist - base_list) * np.log(test_dist / base_list)
        
        stat_df['psi'] = stat_df.apply(lambda row: sub_psi(row), axis=1)
        stat_df = stat_df[['bucket', 'base_cnt', 'base_dist', 'test_cnt', 'test_dist', 'psi']]
        psi = stat_df['psi'].sum()
        
    except:
        print('error!!!')
        psi = np.nan 
        stat_df = None
    return psi, stat_df

```

运行演示：

```python
>>> psi, stat_df = calculate_psi(base_list=list(df[df['draw_month'] == '2020-05'][var]), 
                   test_list=list(df[df['draw_month'] == '2021-02'][var]), 
                   bins=20, min_sample=10)
>>> stat_df
             bucket  base_cnt  base_dist  test_cnt  test_dist           psi
0           MISSING         0   0.000000         0   0.000000  0.000000e+00
1     (-inf,0.0199]       238   0.046777       511   0.065454  6.274898e-03
2   (0.0199,0.0217]       266   0.052280       477   0.061099  1.374765e-03
3   (0.0217,0.0235]       249   0.048939       467   0.059818  2.183942e-03
4   (0.0235,0.0249]       262   0.051494       380   0.048674  1.587606e-04
5   (0.0249,0.0265]       280   0.055031       453   0.058025  1.585503e-04
6   (0.0265,0.0279]       230   0.045204       329   0.042142  2.148738e-04
7   (0.0279,0.0297]       251   0.049332       410   0.052517  1.992932e-04
8   (0.0297,0.0316]       254   0.049921       388   0.049699  9.929669e-07
9   (0.0316,0.0336]       248   0.048742       386   0.049443  1.000043e-05
10  (0.0336,0.0358]       257   0.050511       395   0.050596  1.416201e-07
11  (0.0358,0.0382]       263   0.051690       363   0.046497  5.499267e-04
12  (0.0382,0.0405]       249   0.048939       343   0.043935  5.396963e-04
13  (0.0405,0.0431]       258   0.050708       316   0.040476  2.305601e-03
14   (0.0431,0.046]       256   0.050314       319   0.040861  1.967526e-03
15   (0.046,0.0492]       255   0.050118       321   0.041117  1.781819e-03
16  (0.0492,0.0528]       252   0.049528       314   0.040220  1.937663e-03
17  (0.0528,0.0578]       255   0.050118       391   0.050083  2.398626e-08
18  (0.0578,0.0641]       262   0.051494       369   0.047265  3.623085e-04
19  (0.0641,0.0749]       249   0.048939       452   0.057897  1.505794e-03
20     (0.0749,inf]       254   0.049921       423   0.054182  3.489647e-04
```

## 致谢

加强对各类指标的理解，才能巩固风控模型基本功。感谢 LP 帮助。

## **版权声明**

**欢迎转载分享**，**请在文章中注明作者和原文链接，感谢您对知识的尊重和对本文的肯定。** 

> 原文作者：求是汪在路上（知乎 ID）  
> 原文链接：[https://zhuanlan.zhihu.com/p/79682292/](https://zhuanlan.zhihu.com/p/79682292/)

⚠️著作权归作者所有。**商业转载请联系作者获得授权，非商业转载请注明出处，侵权转载将追究相关责任**。

## 参考资料

* * *

## **关于作者**：

在某互联网金融公司从事风控建模、反欺诈、数据挖掘等方面工作，目前致力于将实践经验固化分享，量化成长轨迹。欢迎交流 
 [https://zhuanlan.zhihu.com/p/79682292](https://zhuanlan.zhihu.com/p/79682292)
