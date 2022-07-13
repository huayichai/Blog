# Vectorization vs. Compilation in Query Execution



本文主要讲Vectorization和Compliation两种优化模型在数据库查询中的比较，最后提出将两者结合以达到更好的效果。

数据库的查询执行模型中，最经典也是最常用的是迭代式模型，即火山模型，对于每个算子都实现了next()方法，用于获取一个tuple的数据。但是这样获取一个tuple需要经过层层调用，代价太高。

因此引入向量化模型，即next()函数返回一组数据（一个block）而不是一个tuple，这样就能显著提升效率。

向量化模型的优点：（1）解释器花费的指令消耗减少为vector-size（2）一次执行可以处理一批数据，而且还可以利用编译器加入SIMD



向量化表达式处理一个或多个array，并将结果存入output array，并且系统确保array的长度是CPU缓存友好的。但是，这个机制需要包含额外的load和store的工作消耗。而编译可以避免这个问题，产生的中间结果保存在cpu寄存器中，不需要load和store。

随后，文章在三个数据库算子上进行了实验，证明Vectorization、Compliation、Vectorization-SIMD、Compliation-SIMD的性能

我只看了PROJECT操作，它文章中分析了使用SIMD后需要执行的指令数远远少于不适用SIMD。





目前就看了这些，关于编译的方法完全不懂。



















