# The Case for Learned Index Structures



我们常见的索引有三种：①B树，用于范围查询；②Hash索引，用于点查询；③BitMap索引，用于存在性检测。

对于上述的索引，本质上都是给定一个key，索引给出该key对应的数据的位置。稍微不同的地方在于，B树对应的数据是有序的（sorted）,Hash和BitMap对应的数据是无序的（unsorted）。

本文考虑是否可以使用机器学习模型Model代替索引的工作，因为，简单想一下，两者都是给定x，返回y。

为什么要这么考虑呢？使用机器学习模型的优点如下：

- 传统的索引是通用型数据结构，不可能针对数据的特征专门制定索引
- 而机器学习模型则能够以很小的代价学习到数据的特征（分布），从而以低成本自动合成专门的索引



## 范围索引 Range Index

本章对应构建B树的Learned版本

![](.\imgs\Snipaste_2022-07-01_17-02-00.png)

> an index is a model that takes a key as an input and predicts the position of the record

索引是把key作为输入，数据的位置position作为输出。对于范围索引来说，其数据都是有序的，因此，预测排序数组中给定key的位置的模型有效地近似于累积分布函数CDF：
$$
p=F(Key)*N
$$
其中，$F(key)$ 的返回值是[0, 1]，表示key对应的数据存在的大概位置的百分比，$N$表示数据的总量。这两个值相乘的结果$p$就是预测的大概位置。

相比B-Tree的通用设计，Learned Index Structures考虑了数据集的内在分布特点并将其用于优化索引的结构。Learned Index误差上限可控，只需要在误差范围内根据预测的位置向左或向右二分查找即可准确找到查找目标。



### **最简单的学习型索引**

对应原文的2.3节

> 作者尝试了用TensorFlow搭建一个每层32个神经元，两层全连接的神经网络，使用一个web server日志的数据集训练后发现效果远差于B-Tree。问题在于：
>
> - TensorFlow是用于大规模神经网络的训练的，小规模场景的调用开销变得不可忽视。
> - 欠拟合问题，机器学习模型可以很好的估计CDF的**整体趋势**，但在单一数据项上很难得到精确的表示。而B-Tree可以简单高效的使用if语句精确划分范围，为了优化“**最后一公里**” 机器学习模型要付出较大的存储空间和计算资源消耗。
> - B-Tree的CPU和cache行为是经过高度优化设计的，每次查找只需使用少量索引。机器学习模型则需要使用全部参数权重完成一次预测。

用简单的话来讲：作者用一个神经网络模型（NN）来替代B树索引，但是一个NN模型进能够拟合数据的大致分布（即整体趋势），如果想要精确找到key对应位置pos（即“最后一公里”），难度和代价太大。



### 递归模型索引

为了解决上面最简单的学习型索引的问题，本章提出了一个新的模型索引：递归模型索引RMI。

**The Learning Index Framework (LIF)**

学习索引框架LIF，我的理解是，他是用来管理训练模型的，我们把数据给它，它帮助我们训练好模型后，将模型的参数返还给我们，方便我们后续使用。



**The Recursive Model Index**

本小节的方案"递归模型索引"用于解决“最后一公里”问题

<img src=".\imgs\Snipaste_2022-07-01_17-36-28.png" style="zoom: 80%;" />

最终Learn Index Structures的模型使用了如上图所示的Staged Model实现。每一个Model都可以是任意一个机器学习模型，从最简单的线性回归（LR）到深度神经网络（DNN）都可以。实践中，越简单的模型越好（避免查找时在模型上花太多时间）。当进行查找时，最上层的模型（只有一个模型），将选择一个第二层的模型来处理这个key。然后第二层的模型，会接着选择一个下一层的模型来处理这个key，直到最底层的模型，才会给出这个key对应的预测位置。但实际上上层每个模型输出的都是预测位置，这个预测被用于选择下层模型（模型id = (预测位置 / 记录总数) * 该层模型数）。

整个Staged Model分层训练，先训练最顶层，然后进行数据分发。数据分发指的是，上层模型将key预测到哪个下层Model，该Model就拥有这条训练数据作为他的训练集。所以随着层数的加深，以及每一层模型数量的提升，每个越底层模型拥有的训练数据是越少的。这样的优点是，底层模型可以非常容易的拟合这一部分数据的分布（缺点是较少的数据量带来了模型的选择限制，复杂模型没法收敛）。

> On the top-layer a small ReLU neural net might be the best choice as they are usually able to learn a wide-range of complex data distributions, the models at the bottom of the model hierarchy might be thousands of simple linear regression models as they are inexpensive in space and execution time. 

本文中采用的结构是：只在顶层使用神经网络模型，在其余层使用线性回归模型。



**Hybrid Indexes**

混合索引。在实际中，如果数据的分布很难被学习到，可以考虑将最底层的Model替换为B树。

<img src="D:\Data\Blog\The Case for Learned Index Structures\imgs\Snipaste_2022-07-01_17-49-30.png" style="zoom:50%;" />

这里给出了构建**混合递归模型索引**的算法。

**1行：**获取该模型有几层stage

**2~3行：**将所有的数据存入tmp_records这个二位数组的第一个位置。这里tmp_records\[i][j]表示i层的第j个模型所需的训练数据

**4行：**遍历所有层级

**5行：**遍历当前层级中的所有模型

**6行：**使用对应的数据tmp_records\[i][j]训练出对应位置的模型index\[i][j]

**7~10行**：只要不是最后一层，那么我们就用刚刚训练好的模型来预测该模型上的数据tmp_records\[i][j]对应下一层的哪个模型，并将数据存入那个模型的tmp_records

**11~14行：**遍历最后一层的所有模型，如果某个模型的误差超过了阈值，则将其替换为B树



### 代码实现

由于作者在文章中没有标出代码实现，因此，我使用The Case for Learned Index Structures为关键词在Github中搜索，找到了Star数量最多的项目https://github.com/yangjufo/Learned-Indexes

该项目实现了一个2stages的模型

重点看其Learned_BTree.py中的代码，它的两个函数：

- hybrid_training() 该函数就是上面的伪代码的具体实现
- train_index() 该函数首先计算了偏差，然后将每层的每个模型存入了文件，最后构建了B-Tree与Learned index进行了性能对比







## 点查询索引 & 位图索引

留个坑，后面有空再写~







参考资料

> 论文原文 https://dl.acm.org/doi/pdf/10.1145/3183713.3196909
>
> 机器学习如何影响系统设计：Learned Index Structures浅析 https://blog.csdn.net/feelabclihu/article/details/115019188
>
> 科研狗看论文第二周（关于learned-index） https://www.codenong.com/cs109340711/
>
> 代码实现 https://github.com/yangjufo/Learned-Indexes