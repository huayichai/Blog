# Data Blocks: Hybrid OLTP and OLAP on Compressed Storage using both Vectorization and Compilation

## Abstract

文中将数据库中的数据分为冷数据和热数据两类，并且约定，冷数据使用压缩算法压缩，不能被修改。

为冷数据设计了一种新型的压缩列数据存储格式：Data Blocks，此外，还在这种存储格式上引入了一种新型的轻量级索引PSMA。为了实现OLTP的高性能，可以直接在压缩数据上进行计算。

目前高性能的分析系统通常使用向量化查询引擎或者 JIT 即使查询编译。Data Blocks 支持细粒度的整合两种方法，将向量化扫描结果输出到 JIT 编译查询通道。





## 2. Related work

Vectorwise中，需要对列中的所有数据进行解压缩，这对点查询非常不友好，因此本文提出了在一种轻量级索引PSMA，可以用于OLAP业务。而使用向量化加速scan操作可以用于OLTP业务。

本文只考虑在压缩的数据上进行filter，而非常复杂的算子，例如join，我们还需要解压缩





## 3. Data Blocks

Data Blocks的目的是以可寻址的压缩方式存储一个或多个属性的数据，目标是保证高性能OLAP和OLTP的同时节约内存。

冷数据是不可修改的，只允许删除操作，删除的方法是在删除位置上添加一个flag。更新操作其实就是在冷数据中删除数据，然后将更新的数据插入热数据中。

Data Blocks有如下几个特性：

（1）根据数据的分布选择最优的压缩算法
（2）只允许使用字节可寻址的压缩算法，像BitWeaving这样的编码结构是不允许的
（3）谓词计算在压缩数据上进行，并且使用SIMD
（4）每个Block中都有一个SMA，里面包含这个block中的最大最小值，用于在scan时跳过block
（5）轻量级索引PSMA，用于将一个value映射到它出现的范围内



**Data Blocks Layout**

1. 第一个值介绍这个Block中存储了多少tuple
2. 后面是对属性的介绍信息，每个属性的介绍信息包括：该属性对应SMA的位置，PSMA位置，dictionary的位置，真实压缩数据存储的位置。
3. 随后是每个属性的数据块，包含第二条中的所有数据。 

详细描述见原文Figure 3

Data Blocks 中为每个属性都存储了一个 SMA。SMA 包含了该 Block 的最大和最小值，则对于有序的数据属性（如日期、主键等）SMA 就可以在查找时快速跳过无用 Block 内容。但对于属性值分布是较为均匀的，SMA 就不能发挥作用，甚至某个异常值就会大大影响到 SMA 的最大最小值。



**Positional SMAs**

这是一个索引PSMA，用于快速查找压缩数据中的某个数据。简单的说，给一个数据，然后PSMA会给你一个range范围，你在这个范围内可以找到该数据。

再稍微详细一点，就是给一个数据，PSMA对这个数据哈希得到table中的一个位置，该位置上存储一个range。

这个table是怎么构建的呢？就是把所有数据哈希到这个table的位置上entry，如果这个位置之前没有被其他数据映射过，则这里就存储该数据的pos；如果这个位置之前被其他数据映射过，则这里存储包含之前所有数据pos的一个范围。

哈希映射方法是什么呢？（1）求得查找值 V 与最小值的差值（2）从差值中找到第一个非零字节，记为 r，并计算该值对应的表位置 i = 差值 + r * 256



**Attribute Compression**

介绍了几种字节可寻址的压缩方案。



## 4. VECTORIZED SCANS IN COMPILING QUERY ENGINES

如果压缩算法的选择在block和column的级别，因为这样可以针对不同的数据类型和分布提升压缩率，但是也会导致不同的物理表示

这样就可Jit带来很大的挑战，因为生成代码的时候，需要兼容所有的storage layout，分支比较多的情况下，会导致编译的代码爆炸

Data Blocks自己决定最合适的压缩方案，但是由于不同压缩方案的处理数据方式不同，因此使用JIT编译执行中会有很多if else，导致效率问题

所以这里提出的方案是，把Scan和处理Pipline分离开，Scan部分用解释执行，而Pipeline部分用JIT，这样可以通过解释执行得到的数据，feed给compiled的pipeline



**Integration in HyPer**

这节没看懂！



**Finding Matches using SIMD Instructions**

这节主要讲，如何根据比较后的result bit vector来提取数据。

在进行谓词比较后，所有符合条件的数据的pos被存储到一个vector中。这一步的难点是如何将result bit vector（bit-mask）转换为32-bit的pos，这里给出了一个快速将bit-mask的信息转换为pos的方案，即提前准备好一个table，这个表中记录了不同bit-mask对应的pos情况。如果bit-mask宽度太大，则table就太大了，因此这里限制bit-mask是8bit，这样保证table的大小是8K，可以存入L1缓存



在获得了这个pos的数组后，文章说使用gather指令来提取离散内存中的数据。







> https://zhuanlan.zhihu.com/p/489441544
>
> https://www.cnblogs.com/fxjwind/p/12576026.html
>
> https://zhuanlan.zhihu.com/p/449583289