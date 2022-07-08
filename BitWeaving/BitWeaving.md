# [文献阅读] BitWeaving: Fast Scans for Main Memory Data Processing

本文介绍了两种数据存储布局storage layout以及配套的数据扫描scan方法，以期充分利用SIMD来加快数据处理。



## 1 Introduction

DBMS中一个经常被使用的操作是全表扫描full table scan，对于每一条数据，需要进行谓词predicate判断，可以理解为SQL中的WHERE语句。

为了加快扫描速度，一些研究中使用了SIMD技术（具体可以参考我之前写的文章），例如将压缩的列数据划分到一个128-bit SIMD word中的4个32-bit的槽中，这样可以同时处理4个32bit的数据，但这个方案有两个缺点：（1）被压缩的数据往往不那么恰好是32bit，假如数据仅9bit，那么需要用填0的方式把9bit扩展为32bit，才能放入SIMD中处理，这样就浪费了23bit的带宽；（2）将9bit扩展为32bit也需要浪费额外的性能来处理。

本文提出的BitWeaving技术，可以充分利用SIMD中word的带宽。BitWeaving有两个版本：BitWeaving/V和BitWeaving/H，他俩对应两种不同的底层数据存储布局。这两个方案的输出均为一个结果比特数组result bit vector，每一个输入tuple均对应该数组中一个bit，该bit表示这个tuple是否match应用到该列上的predicate。



## 2 Overview

先介绍一些后文使用的定义（概念）。

**code**

在列数据库中有很多压缩列数据的算法，比如null-suppression, prefix-suppression, frame of reference, order-preserving dictionary encoding，这些压缩方案都把原始数据转换为定长的编码，因此，本文将这些定长的编码称为code，后续对数据的处理中，直接在code上进行，并不使用解压缩后的数据。



**column-scalar scan**

列扫描操作的输入为：（1）n个k-bit的code（2）谓词，例如基本的比较$= \neq < > \leq \geq$

常数也被编码为code，然后列扫描操作目的是寻找满足谓词的数据。

举个例子，在列a上，寻找满足a<5的数据。

> The focus of this paper is on speeding up scan queries on columnar data in main memory data processing engines. Our framework targets the single-table predicates in the WHERE clause of SQL. More specifically, the framework allows conjunctions, disjunctions, or arbitrary boolean combinations of the following basic comparison operators: $=, \neq, <, >, \leq, \geq,$BETWEEN.

当执行完列扫描后，扫描结果为result bit vector，我们可以根据索引号来获取相关的其他列的数据



## 3 Bit-Parallel methods

本章介绍了两种底层数据存储布局以及它们各自的列扫描方案。



### 3.1 The Horizontal bit-parallel (HBP) method

**stroage layout**

<img src=".\imgs\Snipaste_2022-07-08_17-32-25.png" style="zoom:50%;" />

直接上例子讲解，左侧是10个code数据，右侧是存储布局。这里，我们假设code是3bit，而一个处理器字长为8bit（即一次能处理的数据的长度）。v代表的就是一个处理器字。每个code之间都用1bit的0来作为分隔符。至于为什么将c1和c5放在一个处理器字内，我们后续揭晓。

这里再进行更通用的定义：一个code是k-bit，但我们使用k+1-bit来存储一个code，多出的那个bit是分隔符。我们用w表示处理器字（processor word）的宽度，那么一个处理器字内能够容纳code的数量为$\lfloor \frac{w}{k+1} \rfloor$，多余的部分用0来填充。随后，我们将$(k+1) \times \lfloor \frac{w}{k+1} \rfloor$个code组成为一个段segment，也即一个段中包含k+1个处理器字。最后呢，segment中code的排列方式如下图所示，竖着排列。

<img src=".\imgs\Snipaste_2022-07-08_17-45-18.png" style="zoom:50%;" />



**column-scalar scan**

在扫描后，分隔符用来存储比较的结果。

> Formally, a function $f_o(X,C)$ takes as input a comparison operator $o$, a comparison constant $C$, and a processor word $X$ that contains a vector of $\lfloor \frac{w}{k+1} \rfloor$ codes in the for $X=(x_1,x_2,...,x_{\lfloor \frac{w}{k+1} \rfloor})$, and outputs a vector $Z=(z_1,z_2,...,z_{\lfloor \frac{w}{k+1} \rfloor})$, where $z_i=10^k$ if $x \space o \space C=true$, or $z_i=0^{k+1}$ if $x \space o \space C=false$. Note that in the notation above for $z_i$, we use exponentiation to denote bit repetition, e.g. $1^40^2=111100,10^k=1\underbrace{00..00}_{k}$.

**上面这段英文很重要，主要就是说明将比较结果存在分隔符位。**

接下来介绍如何在上述存储布局下进行数据比较，注，这里的x和y表示code。

- INEQUALITY（$\neq$）。如果$x_i\neq y_i$，那么$x_i\bigoplus y_i \neq 0^{k+1}$，那么$(x_i\bigoplus y_i) +01^k = 1*^k$。因此$((x_i \bigoplus y_i) +01^k) \wedge 10^k = 10^k$就表示x和y不相等。因此，应用到处理器字上，$Z=((X \bigoplus Y) + 01^k01^k...01^k) \wedge 10^k10^k...10^k$ 就是返回结果
- EQUALITY（$=$）
- LESS THAN（$<$）
- LESS THAN OR EQUAL TO（$\leq$）

其余的道理相同，这里就只描述一下不等于是怎么处理的，感兴趣的可以参考原文，我就不翻译了。

![](.\imgs\Snipaste_2022-07-08_18-17-13.png)

![](.\imgs\Snipaste_2022-07-08_18-17-32.png)

我们知道，比较的结果存储在分隔符位置上，那么，我们对一个段中的所有处理器字全部处理后，就需要把每个处理器字的result bit vector合并。怎么合并结果呢？我们再看一下图1，处理后，只有分隔符位置上的bit是有意义的，其余地方的数据无意义，v1保持不变，v2整体右移1位，v3右移2位，v4右移3位，然后$v1 \vee v2 \vee v3 \vee v4$，这样就将结果合并啦。这里就解释了为什么前文HBP存储布局中要那么摆放数据了，是为了后面合并时方便处理。

<img src=".\imgs\Snipaste_2022-07-08_18-27-52.png" style="zoom:80%;" />

算法比较简单，不详细讲了，其中第5行就是数据右移的过程。





### 3.2 The Vertical bit-parallel (VBP) method

**stroage layout**

<img src=".\imgs\Snipaste_2022-07-08_18-38-18.png" style="zoom: 50%;" />

前文HBP中，是一个处理器字内存储完整的几个code，但这种方案会导致空间浪费，因此提出了VBP，能够充分利用所有的空间。

v1中存储code的第一个bit，v2中存储code的第二个bit，...，以此类推。



**column-scalar scan**

在列扫描时，相当于从第一个bit开始，一个一个比特的对比

<img src=".\imgs\Snipaste_2022-07-08_18-45-33.png" style="zoom:50%;" />

1-10行：构造数据

$m_{lt}$记录某个code是否小于C2

$m_{eq1}$记录某个code的值是否与C1还依然相同







