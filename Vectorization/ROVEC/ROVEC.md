# [文献阅读] ROVEC: Runtime Optimization of Vectorized Expression Evaluation for Column Store



原文PDF：http://www.cs.utah.edu/~lifeifei/papers/rovec-tkde.pdf

期刊：TKDE 2021 （CCF-A）



注，我写的文章中一些简称的意思：

- 数据的宽度，指的是要表示一个数据，需要多少bit



第一步看题目：

- ROVEC是文章中提出的框架名字
- Vectorized Expression Evaluation矢量化表达式求值，看到vectorized这个词，我想到的是本文用到了SIMD技术，Expression Evaluation一般是指数据库操作中的谓词predicate的计算过程
- Column Store，列式存储，说明框架ROVEC面向的是列数据库



## Abstract

本文提出了一个名为ROVEC的运行时优化框架，能够有效地优化基于SIMD的表达式求值。

> In this paper, we propose a runtime optimization framework named ROVEC that enables effective optimizations for SIMD-based expression evaluation.

- ROVEC尽最大努力缩减数据类型的宽度，以期提升SIMD操作时，并发处理的数据量
- ROVEC可以被应用到许多表达式求值的算子上（例如，table scan，theta join），可以处理许多不同类型的数据（例如，numeric, time, string）
- 作者将ROVEC在列数据库PolarDB-C上进行了实现



## 2 Background

这一章介绍了两个方面的背景知识

**Column-block storage format**

因为ROVEC是面向列数据库的，因此需要介绍一下列数据库的数据存储格式。

- 列式存储中，数据按不同的列分别存储到不同的物理文件中，而且相同列中的数据被划分未固定长度的块
- 列式存储中，将数据库中一列的数据分块后，压缩，然后存储到文件中。

- 每个分块Block都对应一个块摘要（Block-Summary），块摘要包含（最大值，最小值，和压缩方案等信息），由于数据分布的多样性，即使同一列的数据，不同的块之间也可能使用不同的压缩方案。



**Element-addressable scheme**

翻译下来是：元素可寻址方案，主要介绍了四个对列数据的压缩算法

1. Single-value scheme

   该方案应用于一个Block中所有数据都相同的情况下

2. Dictionary scheme

   将一个Block中所有的不同值构建一个字典，数据实际存储时，存储的是数据对应的索引

3. FOR scheme

   frame of reference的缩写，将数据从大到小排好序，然后只记录前后数据的增量。例如一组数据（60，90，92，97，101），使用FOR压缩后，变为（60，30，2，5，6），只有第一个数据是原始数据，其余的都是增量。增量数据的宽度

4. GCD scheme

   greatest common divisor的缩写，最大公约数。字面意思，就是求出该列中所有数据的最大公约数，然后进存储原始数据除最大公约数后的结果

方案一的思想是利用一个数据来代表整个块的数据，从而减少空间占用；方案二的思想是索引一般原始数据所占空间少；方案三四的思想是降低数据的宽度。



## 3 Design of ROVEC

本章主要介绍ROVEC框架的细节

首先强调了ROVEC利用块摘要来进行优化，传统的SQL层优化器使用列级别的统计信息来优化，因此ROVEC利用的信息更加细粒度，这样细粒度的信息能够更好的反应局部的数据分布情况，从而做出更好的优化。

其次强调的是ROVEC能够直接在压缩的数据上做表达式计算，而以前的方案需要先解压缩，然后再计算，这样不仅解压缩带来了消耗，而且解压缩后数据的宽度又提升了。



### 3.1 Key Optimization: Type Reduction

其实，只要把Type Reduction这个词理解了，正片文章就很容易阅读了！

> However, accessing compressed data inevitably incurs data type promotion.

这里原话是“访问压缩数据不可避免地导致数据类型的提升”，数据类型我们知道代表的是数据的宽度，一般压缩后的数据都比原数据占的比特位少，因此，一般的方案在访问压缩数据的时候，均需要把压缩的数据还原，这就导致了类型提升

**类型缩减Type Reduction其实就是对数据压缩，比如一个数据原本用16bit表示，现在压缩到8bit表示。**

举个例子，假如要计算两个列之间的断言，$COL_1>COL_2$，如果这两个列的数据类型都是$INT64$，我们进行类型缩减后，变为$INT16$，即用16位就能代表原来64位的数据，然后我们还能够直接在$INT16$上做断言，这样使用SIMD理论性能就提升了4倍。

那么如何在类型缩减，即压缩的数据上做断言呢？

下面介绍了四种压缩算法可以被转换成的算子：

> - Single-value can be represented as a single constant;
> - Dictionary can be represented as a dictionary lookup operator;
> - FOR can be represented as a pair of Cast and Add operators;
> - GCD can be represented as a pair of Cast and Mul operators.

再举一些例子：

- 使用Dictionary算法压缩的string类型的数据，可以直接再index上进行比较；
- 如果使用FOR算法压缩的两列之间的表达式计算为$COL_1>COL_2$，则可以转换为$COL_1^`>COL_2^`+(Min_2-Min_1)$，这里$COL_1^`，COL_2$是被压缩后的数据；
- 其他的算子，例如+ - \* 、都可以以相似的方式在压缩的数据上计算。



### 3.2 A Running Example

这一小节给出了ROVEC框架的运行实例，帮助我们先快速了解ROVEC的每个阶段的作用，重点都在下图中：

![](.\imgs\Snipaste_2022-07-07_09-28-35.png)

先介绍一下这个例子，要求解的表达式为$a+b>900\space and\space a<100$，其中的a和b类型都是$BIGINT$，并且使用**FOR压缩**算法分别压缩到$UINT8$和$UINT16$。a列的第N个Block表示为$a_N$，其Block-Summary表示为$\hat{a_N}$。

**阶段一：Block-summary based simplification**

如图1(a)所示，对于表达式$a<100$，结合Block-Summary，$Max(\hat{a_N})=80<100$，因此这个子表达式为True

基于Block-Summary，原表达式被简化为$a+b>900\space and\space True$



**阶段二：Compression-aware expression unfolding**

如图1(b)所示，将压缩的方案展开为子表达式，例如，ROVEC将列a展开为子表达式$CastToInt64(a_N)+Min(\hat{a_N})$，将列b展开为子表达式$CastToInt64(b_N)+Min(\hat{b_N})$，这样方便后续阶段的优化。



**阶段三：Constant folding**

如图1(c)所示，这一阶段意图消除所有的常数，$CastToInt64(a_N)+CastToInt64(b_N) > 900 − (Min(\hat{a_N}) −Min(\hat{b_N)})=400$



**阶段四：Type reduction**

如图1(d)所示，上述的三个阶段都是在根据Block-Summary信息来尽可能地消除常数，以期缩减列的取值范围，从而达到类型缩减。$a_N$和$b_N$的最大取值分别是80（80-0）以及2500（3000-500），则$a_N+b_N\leq2580$，可以被$UINT16$表示。因此，整个表达式类型缩减到$UINT16$。



最终，我们可以看到，如果没有ROVEC框架，我们使用SIMD处理的是64bit的数据，在使用ROVEC优化后，我们处理的是16bit的数据，这样理论上的并行性能提高了4倍。



### 3.3 Optimization Strategy in ROVEC

这一小节详细介绍ROVEC每个阶段的规则。

![](.\imgs\Snipaste_2022-07-07_10-04-36.png)



#### 3.3.1 Block-summary based simplification

利用Block-Summary统计信息来简化表达式，一般来说，会考虑min/max, SQL data type, storage data type, and compression schemes。

为了简化表达式，ROVEC通过DFS（深度优先遍历）来遍历输入的表达式以分辨可优化的子表达式。借助Block-Summary把某个子表达式优化为常数来替换语法树上的节点。



#### 3.3.2 Compression-aware Expression Unfolding

表达式展开对于类型缩减非常重要。

对于不同的压缩方案，表达式展开方式不同：

- GCD scheme，将最大公约数作为一个常数放在表达式中
- Single-value scheme，替换为block-summary中的值
- Dictionary scheme，将数据重新解释到相应的字典索引中
- FOR scheme，表达式减去最小值

>  Note that ROVEC does not restrict a block to be compressed by only one scheme. If a block is compressed by multiple schemes in cascade, ROVEC unfolds the applied schemes one by one according to their applied order. By doing so, instead of decompressing values directly, ROVEC unfolds the decompression process as a subexpression, which can be optimized in the follow-up stages.

ROVEC并不限制所有的block都使用相同的压缩方案。**在进行优化时，ROVEC并不是直接对压缩数据进行解压缩，而是将解压缩的过程展开为子表达式。**



#### 3.3.3 Constant Folding

通过Min,Max信息，将常数的计算消除掉。因为有Min,Max信息，也不需要担心上溢下溢问题。



#### 3.3.4 Type Reduction

为什么会提出Type Reduction？
（1）压缩算法将数据的宽度减少
（2）SIMD的并行处理特别喜欢短的数据，这样可以同时处理更多

在类型缩减后，进行计算时，数据可能出现上溢、下溢问题，因此，ROVEC首先需要根据Block-Summary信息来决定压缩到什么范围才能保证不溢出（原文中说要把类型缩减到tightest，最紧凑）。

<img src=".\imgs\Snipaste_2022-07-07_10-57-20.png" style="zoom:50%;" />

1-2行：采用DFS的方式，遍历整颗语法树

3行：评估得到节点n的最紧凑的类型

4行：如果评估出的输入类型宽度比当前节点的输入类型宽度小，则进入if

7行：替换当前节点的输出类型宽度

8行：因为已知当前节点的输入类型宽度要进行缩减，则去遍历当前节点的所有子树节点，对他们同样进行类型缩减

5-6行：处理是root节点的特殊情况

<img src=".\imgs\Snipaste_2022-07-07_12-16-10.png" style="zoom:50%;" />

算法二根据节点类型以及其子节点的输出范围来评估当前节点的输出范围。

1-5行：对于常数和Field数据，根据Block-Summary信息来评估其范围

6-7行：对于谓词判断，其取值为0或1

8-12行：对于Add等功能性算子，其输出范围与其子节点的输出范围有关

<img src=".\imgs\Snipaste_2022-07-07_12-33-14.png" style="zoom:50%;" />

算法三对当前节点及其子节点更新输入输出范围。被用于算法一中发现某个节点确定需要进行类型缩减，还会递归去其子节点更新。



## 4 System Implementation

这一章中介绍如何将ROVEC应用于现有的数据库PolarDB-C中。

如图2所示，ROVEC的输入是：逻辑表达式+Block Summary，ROVEC的输出是：优化的表达式。

PolarDB-C的SQL语句执行模型采用的是经典的火山模型，即数据是从孩子算子中一条一条获取的。

目前已经支持的算子：

- Table scan，ROVEC为每一个block的数据都进行表达式优化
- Theta join，系统中采用了block nested loop join算法来实现
- other operators，有了Table scan和Theta join，很多其他算子就可以被简单实现。这个作为未来工作。

后面，文章还介绍了如何应对Non-Element-Addressable Scheme的压缩方案，即如果压缩方案是元素不可寻址的，那么就需要首先对数据解压缩，然后再进行后续操作。

![](.\imgs\Snipaste_2022-07-07_13-02-14.png)

最后再附一张系统架构图，可以看出ROVEC作为一个插件被集成到了数据库中，是非侵入性的。在数据库系统的执行层，可以直接与ROVEC提供的API交互。





## 参考文献

> 列存储中常用的数据压缩算法 https://blog.csdn.net/bitcarmanlee/article/details/50938970
>
> 数据库查询执行模型 https://zhuanlan.zhihu.com/p/472063368
