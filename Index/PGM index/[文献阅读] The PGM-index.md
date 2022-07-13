# PGM-Index

原文地址：http://learned.di.unipi.it/publication/pgm-index-fully-dynamic-compressed-learned-index/pgm-index-fully-dynamic-compressed-learned-index.pdf





本文提出了一种新型的学习型索引，其支持前任查询、范围查询、更新操作，称之为分段集合模型索引（Piecewise Geometric Model index, PGM-Index）。

从题目可以看出，dynamic 动态，compressed 压缩，这两个关键词分别表明PGM-Index除了支持基本的查询外，还可以插入和删除元素以达到更新的目的，此外还对模型进行了压缩，从而达到更好的空间效率。

因此，可以简略看一下文章的标题，第二章PGM-INDEX是讲最基本的PGM-Index数据结构，第三章DYNAMIC PGM-INDEX是讲其如何支持插入和删除。而第四、第五和第六章分别对应其三个变体，第四章COMPRESSED PGM-INDEX是讲如何对索引进行压缩，第五章DISTRIBUTION-AWARE PGM-INDEX是讲对查询操作的分布进行感知，从而特殊优化索引以达到更好的查询效率，第六章THE MULTICRITERIA PGM-INDEX，多标准PGM-Index是讲如何让PGM在空间和时间上找到平衡，以满足不断变化的应用场景。





## 1 Introduction



### 问题定义

**本文要解决全动态可索引字典问题，该问题定义如下：**

> fully-dynamic indexable dictionary problem

设计一个数据结构，能够正确地存储一个多重集S中所有的key，并且高效的支持如下几个操作：

![](.\imgs\Snipaste_2022-07-03_16-10-56.png)

- member(x)，判断关键字x是否属于多重集S
- lookup(x)，给定一个key，若该key已被插入，则返回其value
- predecessor(x)，翻译软件叫“前任”，返回所有小于x的数据中最大的那个，其实可以简单理解为排好序的数组中，x的前一个数据
- range(x, y)，范围查询，给出[x, y]关键字x和y之间的所有对应的value
- insert和delete很好理解，插入和删除对应的（K, V）

在本文中，我们把member、lookup、predecessor称之为点查询，range称之为范围查询

其实，对于实现上述点查询和范围查询，我们只需要关注实现**rank(x)**原语即可，其含义为：返回多重集S中小于x的key的数量，只要实现了rank(x)，则上述查询操作均可以借助rank实现：

- 首先假设多重集S中的所有key都已排好序存储在数组A中，则
- member(x): A[rank(x)] == x
- predecessor(x): A[rank(x)-1]
- range(x, y): 从rank(x)对应的数组A中的位置开始向后，直到其key大于y为止



### 相关研究

现存的解决上述问题的经典索引数据结构：（1）哈希索引（2）B树（3）位图索引（4）字典树trie索引

但是他们都有一些问题，哈希索引不支持前序查询和范围查询，位图索引维护的代价太高，字典树往往存储指针，而指纹对应的键以非压缩的方式存储，空间消耗大。因此，**目前主流数据库主要采用B树及其变体作为存储引擎**。

最近，一种新型的数据结构，学习型索引，Learned Index被提出，其关键思想是：训练一个模型Model，让其学习函数rank，以期能够做到将key映射到集合S对应的排序数组A中的位置pos

> The key idea underlying these new data structures is that indexes are models that we can train to learn the function *rank* that maps the keys in the input set S to their positions in the array A.

更具体一点，我们将关键字k（其中k∈S）看作是笛卡尔平面上的点（k, rank(k)），如下图所示：

![](D:\Data\Blog\PGM index\imgs\Snipaste_2022-07-03_16-36-18.png)

学习型索引就是要训练一个模型$f$，在给定key后，模型返回该key在A中的大概位置pos，即$pos=f(key)$，然后使用二分法在[pos-$\varepsilon$, pos+$\varepsilon$]范围内找是否有待查询的key。

上面简要介绍了什么是学习型索引，文章中给出了两种学习型索引：（参考我的前面的博客）

- RMI 递归模型索引
- FITing-Tree 

***强烈建议先看完上述两篇论文后再看本篇**，后面的介绍也是基于已经看完这两篇论文后的*

本文提出的PGM-Index并不像RMI、FITing-Tree那样混合了传统的索引和学习型索引。（RMI的最后一个stage中的模型若error超过阈值，则将这个模型替换为B+-Tree，FITing-Tree在确定一个key应该使用哪个线性模型的过程中采用了B+-Tree来加快查找速度）

> Specically, we design a fully-dynamic learned index, the Piecewise Geometric Model index (PGM-index), which orchestrates an optimal number of linear models based on a fixed maximum error tolerance in a novel recursive structure.



## 2 PGM-Index

本章介绍PGM-Index的数据结构是怎样的，并且还介绍查询算法。

首先介绍符号定义，S是一个多重集，其中包含n个keys，PGM-Index有一个参数$\varepsilon$，它是模型预测的最大单边误差。A表示多重集S中数据排序后的数组

最基本的PGM-Index有两个关键点，先给大家呈上示意图，然后下面我们详细介绍每个关键点：

- PLA-Model （Piecewise Linear Approximation model，分段线性近似模型）
- recursive index structure（递归索引结构）

![](D:\Data\Blog\PGM index\imgs\Snipaste_2022-07-03_17-04-36.png)

**PLA-Model**

> 注：与FITing-Tree相似，采用分段的线性模型

其作用就如图1所示，是将key映射到A中的一个近似pos，误差在$2\varepsilon$内。

为什么称之为“分段Piecewise”？因为单独一个线性模型不足以在误差$2\varepsilon$内将**所有**key映射到A上，因此这里与FITing-Tree一样采用多个线性模型组成了PLA-Model（这里一个线性模型称为Segment，一个PLA-Model参考图2中的一层），一个Segment中包含三部分，{start key, slope, intercept}，即{起始key，斜率，截距}

**recursive index structure**

递归索引结构

这个名字很熟悉呀，不就是RMI（recursive model index）中的递归嘛！没错，相同的理解！

为了适应key的分布，PGM-Index使用了多层PLA-Model，我们先使用所有的key来构建最底层的PLA-Model，然后提取每个Segment中的key形成一个新的集合，然后对该集合再次构建PLA-Model，如此向上递归下去，直到最高层的PLA-Model只剩一个Segment停止。

伪代码如下图所示：

<img src=".\imgs\Snipaste_2022-07-03_17-57-47.png" style="zoom:50%;" />



**总结一下，每个PLA-Model是PGM-Index的一层，每个Segment是PLA-Model中的一个node**

PGM-Index与RMI和FITing-Tree相比有三个优点：

- PGM-Index使用线性模型（即Segment）作为数据结构所有级别Level的恒定空间路由表（即PLA-Model），而其他索引（例如FITing-tree、B-tree及其变体）使用存储大量关键字的空间消耗节点，这些关键字的存储位置仅与磁盘页面大小有关，因此导致无法利用数据分布中可能存在的规律性。而PGM-Index每一层均是PLA-Model，能够有效利用数据分布的规律。
- PLA-Model只需耗费常数时间就能定位到一个$2\varepsilon$的子集，而B+-Tree和FITing-Tree随着元素数量的增多，其查询时间也增长。（解释：我的理解，因为PGM-Index中每一层都是用PLA-Model，换句话说就是利用PLA-Model一层一层定位到最终的segment，而FITing-Tree是使用B+-Tree定位到最终的segment，PLA-Model中使用的是线性模型，每个模型可以拟合多个数据，因此后续再插入数据，也不会导致PGM-Index的层数快速增长，但是，FITing-Tree的B+-Tree，每次插入数据，会导致树的形状发生变化，其高度增长速度比PGM快。仅自己的理解，可能是错的！）
- 在本文中，我们观察到，计算一层PLA-Model中的最小Segment数是一个众所周知的计算几何问题，它允许在线性时间和空间中的最优解，因此超过了fitting-tree和RMI的次优建议。（这个下文再讲）



### 最优的PLA-Model

上面也介绍过了，一个Segment是一个三元组（key; slope; intercept），$f_s(k)=k \times slope + intercept$

何谓最优？其实就是最小化PLA-Model里Segment的数量。

FITing-Tree里采用的算法shrinking cone在计算PLA-Model的时间上是线性的，但计算结果（即segment的数量）不能保证是最优的。

有趣的是，这个问题已经在有损压缩和时间序列相似性搜索领域（lossy compression and similarity search of time series，是这么翻译吧？不太懂？）被广泛地研究。

因此，上述问题被转换为构造一个点集合的凸包问题。这个点集合为${(k_i,rank(k_i))}$，而且是一个递增的。

> As long as the convex hull can be enclosed in a (possibly rotated) rectangle of height no more than $2\varepsilon$, the index $i$ is incremented and the set is extended.

使用一个高度不超过$2\varepsilon$的长方体框住这些点，直到不能框住，再用相同的方法继续构造凸包。线性函数要求把上述长方体划分为两半，即对角线。

注：文章并没有详细解释算法的具体流程，本人对这个算法也不了解，后续有时间看过后把具体算法补上



### Indexing the PLA-Model

本节介绍如何索引PLA-Model，使得能够快速确定某个key应该使用哪个Segment。

构造好PLA-model后，其实就是段的数组$[S_0,S_1,...,S_{m-1}]$，我们Lookup的第一步就是如何根据待查询的key确定应该使用哪个段。

- 比较简单的方案，只构造一个PLA-Model，里面包含有所有的key，直接用二分查找的方案

- 或者可以采用FITing-Tree的多路查找树（B+-Tree），如果这么做，那么基本就跟FITing-Tree没区别了，唯一区别在于这里构造的是最优PLA-Model

因此，本文采用的方案是**recursive index structure**，上面介绍过了，这里不赘述了。简单说，先构建最底层的PLA-Model，然后收集所有段的首个key合成一个新的集合，再为这个集合构建PLA-Model，一直递归下去，最高层只有一个段。如图2所示。

查找时，从最顶层出发，利用线性模型预估出下一层segment的位置，然后在$2\varepsilon$范围内查找合适的段，之后重复上述操作，一层一层向下找，直到最后一层映射到具体的（K, V）。Query的伪代码如下：

<img src=".\imgs\Snipaste_2022-07-03_20-45-42.png" style="zoom:50%;" />

PGM的空间消耗不是线性增长的，而是跟数据的分布趋势有关。（我认为，如果数据很多都符合同一个线性模型，这样模型数量就少，从而空间就节省。）





## 3 Dynamic PGM-Index

本章节主要介绍PGM-Index的插入和删除操作。

现有学习型索引插入操作的实现方案是，将元素按序插入到相应段的缓存中，当缓存满了，将缓存与主索引合并，合并需要重新训练。这个方案在key非常多时，效率较低。本文提出两个插入策略：（1）面向时序数据（2）面向一般数据

- 如果是时间序列的数据，插入的数据肯定是在数组A的最后面，那么如果最后一个段能够存放这个数据，且满足ε的条件，就直接放在最后一个段；否则新建一个段，然后向上层一层一层更新Segment。在这种策略下，每层更新最多只涉及到一个Segment的添加，因此需要的I/O少。

- 如果是一般的数据，即插入的位置可以是任意的。这里则采用**LSM-Tree**更新数据的思想。

  准备一组集合$[S_0...S_b]$，大小分别为$2^0,...,2^b$，其中$b=logn$。为每个集合构建PGM-Index。
  这里采用类似于LSM-Tree的思想，向顶层加数据，顶层满了归并到下一层，一直这么递归下去。
  具体来说，最开始所有集合是空的，每次插入元素，都从下标0开始，找到第一个空的集合$S_i$，然后将该集合前面所有集合的PGM-Indexs加上新的数据合并成一个PGM-index放在$S_i$上，即$S_i$存放了前面所有集合的数据，此时前面所有集合的数据均清零。（其实，PGM-index合并很快，因为所有数据都是有序的，合并这个PGM就是简单拼接，可能需要考虑一下接口之间的model是否需要合并即可）



最终，整个PGM-Index可以索引$2^{b+1}-1$个数据，如果还不够，那么直接在那组集合尾部添加一个2倍大小的集合，即$2^{b+1}$，就可以继续插入了

对于删除操作，在需要删除的元素上添加一个特殊标记即可，原文没有详细描述，让我们自行参考一篇论文。



## 4 Compressed PGM-Index

*注：由于文章篇幅的原因，论文中变体使用的技术没有详细展开，而是让读者自行参考其他论文。*

本章主要介绍PGM-Index的变体：压缩的PGM-Index

压缩PGM-Index的关键是提供一个对key，slope，intercept的无损压缩算法

> 原文说：由于对key的压缩算法已经很成熟了，让我们自行参考文献
>
> A. Moat and A. Turpin. Compression and Coding Algorithms. Springer, Boston, MA, USA, 2002.
>
> G. Navarro. Compact data structures: A practical approach. Cambridge University Press, New York, NY, USA, 2016.



**压缩intercept**

前面FITing-Tree中的截距intercept的定义跟y=ax+b中的b的意思是相同的，即当x=0的时候y的值代表截距。

这里为了压缩，改变了截距的定义。即，对于一个段$S_j=(key_j, slope_j, intercept_j)$，希望在段的坐标系里，$intercept_j$是递增的。

为了满足这个递增的条件，我们在根据输入k计算其位置pos的公式要改变为：$f_{s_j}(k)=(k-key_j)\times slopes_j+intercepts_j$，可以与前文的公式对比一下，这里多了$k-key_j$这样一个步骤，这样的话，就可以保证每个段的$intercepts_j$是递增的。

最后，文章又强调了intercepts是小于n的。

然后，让读者自行参考下文的提出的简洁数据结构，用该数据结构来存储所有的intercepts，空间是$mlog(n/m)+1.92m+O(m)$ bits，随机访问时间是O(1)

>  D. Okanohara and K. Sadakane. Practical entropy-compressed rank/select dictionary. In Proceedings of the SIAM Meeting on Algorithm Engineering & Expermiments, pages 60-70, Philadelphia, PA, USA, 2007.



**压缩slope**

每个段的斜率可以在一个区间内取值！（阅读FITing-Tree的论文可知求出的每个段，都有自己的斜率区间slope intervals，即[最小斜率，最大斜率]）

在本文中，求出的m个optimal segments的斜率区间为$I_0=(a_0,b_0),...,I_{m-1}=(a_{m-1},b_{m-1})$ ，因此，原本每个段的斜率$slope_j$属于斜率区间$I_j$，其中$j=0,...,m-1$

本文压缩的方法是：希望将原本m个不同值的斜率缩减到t个不同的值，然后为这t个值构建一个数组$T[0, t-1]$，然后原来直接存储斜率slpoe的地方改为存储数组T中的地址，这样就能使得每个斜率被编码为$\lceil logt \rceil$个bit。在实验部分证明，这个压缩算法能保证$t\ll m$，因此压缩效率显著。

算法的详细细节如下：

1. 首先按照字典序对斜率区间进行排序（先按a排序，然后按b排序），得到数组$I$
2. 然后对数组$I$进行扫描，以期最大化斜率区间的交集

举个例子：对于排序后的斜率区间$\{(2,7), (3,6), (4,8), (7,9),...\}$，我们可以看出$\{(2,7), (3,6), (4,8)\}$这三个区间有交集$(4,6)$，而斜率区间$(7,9)$与前三个没有共同交集。因此，我们可以设置前三个段的斜率区间均为同一个区间$(4,6)$，这样就达到了压缩的目的。然后继续重复上述过程，直到扫描完所有的斜率区间。





## 5 Distribution-aware PGM-Index

本章主要介绍变体：分布感知的PGM-Index

先前没有考虑过查询的分布，都是认为查询是均匀分布的，但在实际中，查询是斜偏分布的，这里，本文想实现一种索引，对经常执行的查询有更快的执行速度，比不经常被执行的查询快。

定义分布感知的数据格式：$S=\{(k_i,p_i)\}_{i=1,..,n}$，其中$p_i$是查询$k_i$的概率。

分布感知问题是：希望对$k_i$的查询时间为$O(log(1/p_i))$

具体实现方案：

对于不同查询概率的key，我们在训练Model的时候为其规定不同的误差，即$min(1/p_i,\varepsilon)$。简单理解一下，因为通过模型得到大概的pos后，还需要二分查找来最终确定，如果误差允许的范围大，则二分查找所需要的时间就长，相反，如果对于高频查询的key，将其允许的误差范围降低，则二分查找的就快。

前面介绍的是如何构造分布感知的PLA-Model，下面介绍如何构造分布感知的PGM-Index，即**每一层**的都是分布感知的。
定义q是一个段中最大查询概率，P是这个段中查询概率之和，然后q/P表示上一层中，这个段首个key的被查询概率。（“加权”思想）



## 6 The Multicriteria PGM-Index

本章主要介绍的变体：可以设置数据结构的参数，以调和其时间和空间性能，来应对不同的应用场景、设备





## 参考文献

> [文献阅读] The Case for Learned Index Structures https://zhuanlan.zhihu.com/p/536461846 
>
> 初探FITing-Tree https://zhuanlan.zhihu.com/p/433996803
>
> 学习索引论文献阅读 FITing-Tree: A Data-aware Index Structure https://mp.weixin.qq.com/s/22saUouEXCs9nqKOlFb00A