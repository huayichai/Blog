# [文献阅读] Conditional Cuckoo Filters

## 前言：

- 原文请参见：

  [Conditional Cuckoo Filters | Proceedings of the 2021 International Conference on Management of Data (acm.org)](https://dl.acm.org/doi/10.1145/3448016.3452811#:~:text=We propose the Conditional Cuckoo Filter%2C a simple,cuckoo filters to handle insertion of duplicate keys.)

- 本人水平有限，文章很多地方每太读懂，肯定有不少错误，希望大家批评指正。（本科生，刚刚开始做科研，呜呜呜，太菜了）

- 本文按照原论文章节顺序，简要介绍了各节主要内容。目前只读了它大概的算法是怎么样的，后面的优化部分和实验部分没看。



## 正文阅读：

### 摘要：

本文提出了条件布谷鸟过滤器（CCF），主要有两点贡献：

- 在集合成员测试中，允许使用谓词
- 为了解决cuckoo不能重复插入2b以上个元素的问题，提出了链式技术

条件布谷鸟过滤器（CCF）能够极大减少join运算中tuples的数量



### 1 介绍：

介绍什么是**带谓词的**近似集合成员测试？

之前的过滤器都是存储（key,value），CCF存储的是（key, attributes），在进行查询时，需要指定key和关于attributes的谓词，也即先通过key找到所有key符合的条目，然后再从找出的条目中依次判断是否有符合谓词的项

​                 ![img](https://docimg1.docs.qq.com/image/zdJ_ghiPv3lf3kQr8Zr9rQ.png?w=586&h=78)        

*k:(k,a)意思是，每一行关键字是k，可以通过k索引到行。然后行中存储了k以及很多的属性attributes*

*Sp意思是测试项目x是否在数据集中，并且其属性也要满足谓词P*



注意：CCF的谓词只支持**相等**



- CCF跟CF一样只存指纹

  In the case of a CCF, the value is a sketch of attribute columns.

- CCF中允许key相同的情况，因为其attribute不同。
- key相同的情况对应CF中，就是插入重复的key呗



*我感觉他文章中的sketch想表达的意思是数据结构*



### 2 相关工作：

说别人的工作没考虑过谓词这回事



### 3 Filters in join processing

这段讲了filter在数据库join时如何运用的。

我目前是这么理解的（可能不对）：3个表，要根据某列（后面称为key）把表连接起来。每个表预先都已经有了自己关于key的filter，假如要连接某个key，则只需先去其他两个表的filter查有无这个key，若有则真的去连接。这样的话能加速，并且文中说这样减少了产生的中间表的大小。

它说CCF的目的就是给每个表建立CCF，然后先根据attribute筛选出小规模的hash table，这样更快一些。比如，电影表中，太大了，如果我们想要找好莱坞电影中的某些东西，也要去用整个电影表对应的filter来过滤，所以本文提出我们先对filter进行筛选，筛出好莱坞的filter，然后根据这个filter再弄！这样就大大减少了中间表的大小。



### 4 Preliminaries（预备工作）

前面介绍什么是cuckoo



#### 4.3 Multisets

本文提出，cuckoo对多集和重复插入元素的支持有限。

> **什么是多重集Multisets？**
>
> ![src](https://docimg4.docs.qq.com/image/v1bnROcY5JWw4lCEoHVfEw.png?w=1280&h=111.86682520808561)        
>
> *我感觉可以理解成有序的数组*

文章说解决cuckoo不能支持重复插入的方案有两种，一种是给每个entry加一个计数器，另一种是允许插入重复指纹。本文说CCF采用的就是第二种思路。

文章说cuckoo虽然能插入2b个重复元素，但经过测试，如果有重复元素插入，会使cuckoo的负载率急剧下降



### 5 Conditional cuckoo filters

CCF能够存储属性信息，需要解决两个问题：

- 如何在少量空间里存储一组属性
- 如何处理重复键key的插入

解决方案：

- 对于怎么存，有三种方案，用一个vector of fingerprints或者bloom filter或者两者的混合
- 对于如何解决冲突，使用链式机制，允许在遇到冲突时，使用多于2个bucket，或者从fingerprints vector 切换到 bloom filter



#### 5.1 Attribute fingerprint vectors

本节介绍如何构建fingerprints vector，其实就是把所有属性值（attribute value）hash成一个4bit or 8bit的指纹，连一起，就完事了？？？

然后它说4bit或8bit虽然FPR高，但问题不大，因为即使再高，只要多几个谓词筛选，都不会有很大影响



#### 5.2 Bloom filter attribute sketches

没看懂，它说在每个entry后面加上一个bloom，把所有属性指纹存bloom里面

然后给了算法1和算法2，两个都是宏观上的查询策略，没有说明白底层怎么存的数据![img](https://docimg6.docs.qq.com/image/SMEcpUxscveuyHDh-YLiEA.png?w=1044&h=408)        

给关键字k和某个属性的值a，查询符合这个谓词的k存不存在

![img](https://docimg1.docs.qq.com/image/4NcRDQkrwE7OizeILxI7iw.png?w=1033&h=460)        

只给了属性的值，算法遍历整个cuckoo的所有entry，判断每一个属性是否符合该谓词，若符合，就把这个entry放入它新建立的cuckoo。之后返回这个cuckoo。意思就是，根据谓词筛选出所有符合的，无论key是多少

这真有提升嘛？返回一个cuckoo，虽然是经过谓词筛选过的，但是无论从大小还是速度上，都没变啊？（它返回的cuckoo，其实就是不符合谓词的地方，置为0而已）



### 6 Multiset representations

介绍它为了解决重复插入问题，有两个方案：

- 把finggerprints vecttor转换为bloom（这样不会丧失删除能力吗？）
- 链式机制



#### 6.1 Bloom filter conversion

这里它说每个entry需要多一位bit来标记它是一个bloom还是vector

所以这个bloom到底是存在d|a|bit上还是另外开个空间，挂在entry后面呢？而且换成bloom后还能踢来踢去吗？

- 看懂了，假设已经在2b个entrys空间内存了d个相同的指纹了，现在进行bloom转化，把这d|a|个bit大小的空间换成bloom的组织形式

- 若要踢走某个元素，需要重构bloom



#### 6.2 Chaining

链式技术，就是对于两个候选桶，认为设定一个值d，即最多存d个相同的key，如果超出d个,则按照它给的公式去计算新候选桶位置。这种链式插入，最多重复Lmax次。（注意，这里计算出一个新的桶地址l，相当于还用之前xor方式确定另一个候选桶地址）

![img](https://docimg4.docs.qq.com/image/5Bx0VfVhAPJUOSMTY_Zl8Q.png?w=273&h=37)        

查询时，给定k和谓词，他会在两个候选桶找是否满足，若不满足，则统计两个候选桶中重复的key数量是否到达d，若到d，则计算另外候选桶的位置l`，然后从另外两个候选桶继续找，一直这么递归下去，直到找到符合谓词的key，或者迭代Lmax次后返回true。

**引理1：**CCF中副本的数量受d的限制

**引理2：**链式技术，最多有l1....ln个，n<Lmax，如果当前的插入实在第li里，则表明其前面的所有bucket都各自插了d个相同的key

**引理3：**CCF没有假阴性

## 回答疑问：

1. 本文提出的谓词解决的方法，之前cuckoo也能实现啊？那本文究竟搞了什么东西？

   本文其实就是你说的用cuckoo实现谓词，把之前存value，现在改存属性值。

   这就是所谓的“老算法应用在新场景下”，虽说这个思路很简单，但只要是没有人提出来，大家都不会往这个方向想，但是有人提出来了，别人想想又很简单。

   而且，我觉得本文主要创新点不在于这个谓词，而是主要解决了cuckoo重复元素不能插入的超过2b次这个问题

2. 本文整体写作思路是什么？

   - 首先把谓词引入了filter，介绍了过滤器也能应用到谓词这个领域，不仅仅是判断key。
   - 然后又根据谓词，介绍了两种类型的查询算法

   - - 根据（key，attribute）查是否有符合的条目
     - 根据attribute查出所有符合谓词的key，返回CF，相当于是一个初步筛选

   - 由于提出了谓词，所以要介绍一下对于（key,attribute）该怎么存储

   - - fingerprints vector：key + attribute value
     - bloom filter：把attribute value插入bloom

   - 由于提出了谓词，所以肯定会出现大量key相同，但attribute不相同的插入，所以要解决cuckoo不能大量重复插入相同key的问题

   - - 为什么会出现这种现象呢？因为cuckoo是根据key来计算两个桶的，所以两个桶2b个位置会很快被占满
     - 本文提出了链式技术，要求最多可以被重新链Lmax次，当两个bucket插入相同key达到d的情况下，就加入新的链接

3. 链式技术NB在哪？

   在基本实现可无限制增加相同key的情况下，几乎不需要引入额外的空间，即没有添加任何指示位bit

4. attribute的数量是不是要限制？

   key + attribute的fingerprints既然替代了之前cuckoo的每个entry，则肯定是要限制长度统一，比如之前entry的长度12bit，当时只存了一个key的指纹，现在假设我们存4个attribute，每个占4bit，则总共每个entry需要12+4*4=28bit

   注意，简单的理解，每个CCF对应的是一个数据库的表，key代表的是表的行，attribute代表一个表的列

