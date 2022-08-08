# geohash、google s2、uber h3三种格网系统比较（1） - 掘金
全球离散格网系统，是指将地球表面按照一定的规则划分为格网区域，通常对于格网系统，有固定格网和非固定格网之分，非固定格网诸如Delaunay三角网，泰森多边形，往往是通过已知点去构建的，位置点不同，则构建的网格也是不同的。非固定格网今天我们暂且不讨论，而去讨论在工程中更常见的固定格网系统。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3bb28c4cbc0643dcbc364be572441931~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

首先讨论什么是固定格网，固定格网顾名思义就是按照固定位置划分的格子，一个点仅会对应一个格子，且每个格子是尽量一致的。在实践中，诸如点的空间索引、热力图的计算、区域的划分等场景都会用到格子系统的概念。今天就来讨论一下常见的三种开源格子系统geohash、google s2、uber h3这三种空间格子系统的优劣和适用场景。 从格子系统的构建上来说大致分为三步。第一步将**球面（地球表面）转换为平面**，第二步对**平面多边形的覆盖**，第三步对**平面多边形进行编码**。下面将会对这三种固定格子系统进行每一步原理的大致讲解。为了讨论的简单，我们将地球视为一个正球体（实际上地球不是正球体）。

1 球面转换为平面
---------

有了解平面投影相关gis知识的同学可能了解常见的投影方式，比如墨卡托投影，正轴等角圆柱投影，就是将圆柱面贴在球面上，将地表通过球心投影到圆柱表名，再将圆柱面展开成长方形的过程。  
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/97963b8a36fb40a8a8cd746e2dfc648f~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)
  
这种投影是具有代表性的一类投影，优势和劣势同样明显。优势在于投影非常简单且容易辩识易读。劣势也非常明显，变形非常严重，尤其是在球体的南北两级，在展开后放大了非常多。这在构建全球级别的格子系统时是比较严重的缺点。 另外一种就是通过正多面体，对球体进行包围，然后再展开成平面。google s2和uber h3都是通过这种方式做的。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9ddd772d75bd4c3983988ed9f1431f51~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

google s2示意图

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b40001d6ebbc406a9cc4f3886e55f7db~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

uber h3示意图

正多面体即柏拉图体，在几何学中仅有五种，正四面体、立方体、正八面体、正十二面体和正二十面体。由于google s2的格子是正方形，因此选用了正八面体，而uber h3选用了正二十面体。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0ad9454d4826407590e004650f44ba05~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)
用正多面体拟合球体比圆柱面拟合球面的变形要小很多，且正多面体的面数越多，拟合球面越好，变形越小，但计算会更加复杂。这种投影方式的优势在于变形小。劣势在于计算相对复杂（面越多越复杂），且不易识读（下面google s2的展开图你能看出来南极洲在哪？） ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c62f81a7d3e141339654786f60262441~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)
 值得一提的是，google s2在做投影时，也做了一些变形的优化。 ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2afd76e41ff94f0697a4f90d3e8b80c5~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)
 三种格子系统所用的投影方式：

*   geohash：正轴等角圆柱投影
*   google s2：正六面体
*   uber h3：正二十面体 所以在格子系统的投影方式层面赋予的优劣：  
    **变形程度：uber h3 < google s2 < geohash**  
    **易识读程度：geohash > google s2 = uber h3**

2 平面多边形的覆盖
----------

全用同一种正多边形来铺满整个平面，叫做正镶嵌图。正镶嵌图有3种，也只有这3种。只有正三角形、正方形、正六边形才能如此镶嵌 ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ff5e393bad1c4b979b1ad8452d9768a0~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)
 对于不同的形状的选择会影响格子系统的两个特性。多级格子的覆盖、中心格子对周边格子的权重影响、格子的级别数量。这几个特性在不同的使用场景有这不同用处。

### 1多级覆盖特性

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/faa0b475db234365951bf85498ae52bb~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)
 多级覆盖特性对于构建空间索引有十分明显的优势，索引存储的时候可以按照树形结构进行存储。仅有三角形和正方形是可以多级覆盖的，六边形无此特性。 三种格子系统中，geohash用了矩形+正方形格子的方式（由于base32编码，中间的一级会变矩形），google s2采用了正方形格子，uber h3采用了六边形格子。同样uber h3对多级覆盖也做了一些优化，每层的格子旋转了19.1°，以近似进行覆盖下一级格子。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b80007045cbf459a82b226839eeaa0f0~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)
 ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/113130d37169403fad5c6bf1bf02209a~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)
 但可以看到uber h3中下一级格子还是没法完全覆盖上一级格子的。 因此多级覆盖特性：  
**geohash = google s2 > uber h3**

### 2 临近格子权重一致特性

这个特性最典型的应用场景是热力图，以滴滴的派单热力图举例，某格子内的供需不平衡，必然会影响到周围格子的供需情况，对于六边形来说，所有的格子都是边相邻，即周边格子的权重是一致的。 ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9c0cc32728844698b6bdd9618621a726~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)
 而对于三角形和正方形来说，除了边相邻之外，还有点相邻，因此边相邻和点相邻的权重值很难确定。 ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d91515b1d5aa4a50bcb126924e777889~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)
 所以通常在热力图的选择上，选择六边形是比正方形更好的选择。 因此临近格子权重一致特性上 **uber h3 > geohash = google s2**

### 3 格子级别数量

格子的级别数量从最大的格子级别开始，到最小的级别一共有多少个级别的统计。格子级别数量越多，格子从大到小的过渡越平滑，级别选择也更丰富。 geohash通常提供12级别的格子，从约5000km到约3cm。

| Geohash length | Cell width \* Cell height |
| --- | --- |
| 1 | ≤ 5,000km × 5,000km |
| 2 | ≤ 1,250km × 625km |
| 3 | ≤ 156km × 156km |
| 4 | ≤ 39.1km × 19.5km |
| 5 | ≤ 4.89km × 4.89km |
| 6 | ≤ 1.22km × 0.61km |
| 7 | ≤ 153m × 153m |
| 8 | ≤ 38.2m × 19.1m |
| 9 | ≤ 4.77m × 4.77m |
| 10 | ≤ 1.19m × 0.596m |```null
| 11 |	≤ 149mm	×	149mm |

```

| 12 | ≤ 37.2mm × 18.6mm |  
google s2提供30个级别的格子，从约8000km到约1cm ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b2679efd3185456595a9f781334323bf~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)
 uber h3提供15个级别的格子,从约1107km到约0.5m ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d16de2c90d734930acef849b6cf20e74~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)
 从级别数来看  
**google s2 > uber h3 > geohash**  
google s2更平滑，因此在数据级别跨度比较大的时候，google s2是更优的选择。