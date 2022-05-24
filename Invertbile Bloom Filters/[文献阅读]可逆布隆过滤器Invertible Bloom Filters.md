# [文献阅读] 可逆布隆过滤器 Invertible Bloom Filters

## 一、前言：

- 原文请参见：

	https://ieeexplore.ieee.org/abstract/document/5551131/

	PDF：https://arxiv.org/pdf/0704.3313.pdf

- 原论文主要研究Straggler Identication问题，其中提出了一个算法Invertible Bloom Filters（IBF）

	本文主要聚焦于介绍可逆布隆过滤器IBF



## 二、可逆布隆过滤器 Invertible Bloom Filters

论文首先简要介绍Bloom Filter（BF）和counting Bloom Filter（CBF）。本文不对这两个Filter做详细介绍，仅仅介绍论文中的一些符号定义，方便读者理解后续算法。

**Bloom Filter:**

- 数据集合S的大小为d，BF的假阳性率为$\varepsilon$
- BF中有一个哈希表B，B包含m个single-bit cells，其中$m=O(dlog(1/\varepsilon))$
- BF配备k个哈希函数$\{h_{1},...,h_{k}\}$，其中$k=\Theta(log(1/\varepsilon))$
- 每次插入元素x时，把k个哈希函数在哈比表中对应的**位**设置为1，即$B[h_{i}(x)].bit=1$

**counting Bloom Filter:**

- CBF与BF的主要区别是支持了删除操作
- CBF把BF中的每个single-bit cell替换为了计数器counter
- 置1操作改为了计数器加1，即$B[h_{i}(x)].count+=1$



### 1. IBF的数据结构

**哈希函数：**

- 除了k个哈希函数外，新增三个哈希函数$f_1,f_2,g$，其中
	- $f_1,f_2$将范围为$[0,n]$的整数映射到$[0,m]$
	- $g$将范围为$[0,n]$的整数映射到$[0,n^2]$



**Cell：**

最初BF的每个cell其实就是一个bit，CBF的每个cell是一个计数器，而IBF的cell包含三部分：

- count：计数器，与CBF功能相同
- idSum：记录映射到该cell的**原始**数据的和。假设原始数据为x，则$B[i].idSum+=x$。若$B[i]$存储了m个相同的原始数据x，则$B[i].idSum=mx$
- hashSum：记录映射到该cell的原始数据经g哈希后的和，即$B[i].hashSum+=g(x)$。若$B[i]$存储了m个相同的原始数据x，则$B[i].hashSum=mg(x)$

```
struct {
	count,
	isSum,
	hashSum
} Cell;
```

需要注意的是，`isSum`的大小至少为logn+logd bits，`hashSum`的大小至少为2logn+logd bits，其中n为每个元素的范围，d为数据集中元素的数量。这里logd的意思是能够应付每个元素都被映射到相同cell的情况下，求和不溢出。



**哈希表：**

除了哈希表B，IBF还有一个哈希表C。

- B中使k个哈希函数映射
- C中使用两个哈希函数f1,f2进行映射

新增一个哈希表的目的是，为了后面还原存储元素做准备。有些情况下，仅凭借B是不能够还原数据的。



**全局计数器：**

count：记录当前Filter共计已存储元素的数量





### 2. IBF的基本操作

论文介绍了两个操作：Insert、Delete

![](.\img\Snipaste_2022-05-24_15-47-19.png)

![](D:\Data\Blog\Invertbile Bloom Filters\img\Snipaste_2022-05-24_17-25-41.png)

根据伪代码可以看出，思路与CBF非常像。每次插入元素：

- 全局计数器count加1
- 哈希表B中k个哈希函数对应的cell中的{count, idSum, hashSum}均更新
- 哈希表C中2个哈希函数对应的cell中的{count, idSum, hashSum}均更新
- 其中，count+=1，idSum+=x, hashSum+=g(x)

删除操作为插入操作的逆过程，这里不详细介绍了。



### 3. IBF还原已存储的数据

**定义pure cell：**如果一个cell仅被一个item元素影响，则称之为pure（纯）的cell，也就是说某个cell没有被多个不同的item映射过。

- 有了pure cell，我们就可以很简单的计算出这个cell对应的原始数据了。即x=B[i].idSum/B[i].count。
- 当我们计算得到了一个原始数据x，我们就可以将这个x从Filter中删除，然后再尝试寻找其他pure cell。然后不断重复上述过程，直到计算得到所有的原始数据。



**关键点就在于如何找出pure cell？**

判断条件：g(B[i].idSum/B[i].count) == B[i].hashSum/B[i].count

通过g以及hashSum，我们可以轻松判断这个cell是否是pure的。

```c
while 存在i,s,t g(B[i].idSum/B[i].count) == B[i].hashSum/B[i].count do
    if B[i].count > 0 then
        记录原始数据x=B[i].idSum/B[i].count
        在哈希表B和C中使用Delete操作删除元素x
    else
        重新插入-B[i].count个元素x
    end if
end while
if count==0 then
    完成
else
    重复删除while循环内的操作，只不过判断的对象改为哈希表C
end if
```

![](.\img\Snipaste_2022-05-24_16-10-42.png)



### 4. 理论分析

有极低的概率，虽然某个cell被多个不同的item映射过了，但是其是符合判断条件的g(B[i].idSum/B[i].count) == B[i].hashSum/B[i].count

理论分析以后有需求再看！



### 5. 实验

- 还原原始数据的实验，分了两类

	- 第一类：仅用一个哈希表B
	- 第二类：用两个哈希表C

	两个哈希表均有101个cell

- 每次，都尽可能地多还原数据，直到再也找不到满足条件地pure cell为止。

- 并且，每还原一个数据，都去对比一下还原地是否正确



文章给出了直方图，展示了在不同数据集下，最大可逆的次数。实验结果可以看出，使用哈希表B和C一起，可以显著提升数据还原的能力。

![](.\img\Snipaste_2022-05-24_16-45-02.png)



## 三、总结：

优点：

- 能够还原数据



缺点：

- 要求原始数据必须是integer整数才行，而且大小限定范围为[0,n]。假如数据分布不均，有很小的数，也有非常大的数，这导致水桶效应。
- 空间效率太差。不仅存储了count，还要存储原始数据以及原始数据的哈希值。这点设计感觉背离了过滤器数据结构设计的初衷——以较少的空间判断元素是否在集合中。