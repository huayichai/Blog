# [文献阅读] ROVEC: Runtime Optimization of Vectorized Expression Evaluation for Column Store



原文地址：

期刊：TKDE 2021 （CCF-A）



注，我写的文章中一些简称的意思：

- 数据的宽度，指的是要表示一个数据，需要多少bit





第一步看题目：

- ROVEC是文章中提出的框架名字
- Vectorized Expression Evaluation矢量化表达式求值，看到vectorized这个词，我想到的是本文用到了SIMD技术，Expression Evaluation一般是指数据库操作中的断言predicate的计算过程
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







