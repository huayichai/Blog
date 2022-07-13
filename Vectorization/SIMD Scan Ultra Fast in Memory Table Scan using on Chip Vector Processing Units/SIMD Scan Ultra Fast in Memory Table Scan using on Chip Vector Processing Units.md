# SIMD Scan Ultra Fast in Memory Table Scan using on Chip Vector Processing Units



本文主要关于利用向量化技术面向**列式存储的内存数据库**做table scan操作的加速

> In this paper, we introduce a novel SIMD approach for in-memory fast table scan operations working on compressed table columns. We utilize the latest SIMD capabilities of each core in super scalar multi-core processors to efficiently decompress in-memory table columns and search for a scan value with a considerably lower latency.

看上面这段总结性的文字，其实主要优化的有两点：

- 高效解压缩列数据
- 以较低延迟在列中查询符合条件的数据

文章把这两个贡献分别叫做：

- Vectorized Value Decompression
- Vectorized Predicate Handling





在第二章它介绍了一个轻量级的压缩算法，Numeric Compression（NC）压缩算法。其实就是对于一个数组中的所有数，找出最大值，然后找一个能够涵盖这个最大值的最近凑的bit数即可表示所有值。因此，一个列中原本32bit的数据，可以用n-bit来表示。文中给的例子是用9bit表示。

随后开始介绍**如何用SIMD来加速解压缩**的过程。主要分为三个过程，（1）16-Byte对齐（2）4-Byte对齐（3）bit对齐。为什么是这几个数字呢？先介绍一下前提，文章用的128bit的向量寄存器，所以是16Byte，然后呢，压缩前的数据宽度是32bit，所以是4Byte。这三个步骤的主要意思是，首先从内存中读出16Byte的数据，然后将其中的4个压缩数据按32bit对齐到128bit的向量寄存器中，最后将每个压缩的数据展开为32bit。

其实重点就是**对齐**！





### **Vectorized Value Decompression**

**（1）16-Byte 对齐**

<img src=".\imgs\Snipaste_2022-07-13_17-07-33.png" style="zoom:80%;" />

就是将内存中的128bit的数据存入向量寄存器中，假设压缩后的数据为9bit，因此一次性能读取14个完整的数据



**（2）4-Byte 对齐**

<img src=".\imgs\Snipaste_2022-07-13_17-11-00.png" style="zoom:80%;" />

4个数据4个数据的处理，分别将前四个数据放到32bit对齐的位置上，如图所示。需要注意的是，这里每个数据的左右两边都有无意义bit，在下一步里要去掉。

一种特殊情况，假设被压缩后的数据是27bit，则会出现一个压缩数据占据了5个Byte的特殊情况，这种情况详细参考论文吧，我这里就不记录了。



**（3）Bit 对齐**

<img src=".\imgs\Snipaste_2022-07-13_17-14-56.png" style="zoom:80%;" />

最后一步就是将数据低字节对齐。采用位移+掩码的方式

如上图所示，先让每个数据分别右移不同的bit，以保证左边的无效bit数量是对齐的，然后再统一左移相同bit。

再SIMD中，没有分别让4个32bit的数据右移不同bit的操作，因此这里采用乘法来解决，一个数据乘2代表右移1bit，因此，只要每个数据乘的常数不同，就能实现上述要求。





### **Vectorized Predicate Handling**

这一节介绍如何加速查询。其实非常简单，就是将4个解压缩后的数据装进向量寄存器，然后直接用SIMD比较即可。

<img src=".\imgs\Snipaste_2022-07-13_17-21-11.png" style="zoom:80%;" />

上图其实就是只执行了解压缩的前两个步骤，第三个步骤只用掩码去除点无效bit，然后直接构造相应的常数去compare即可。最后输出result bit vector







### Implementation

有一个问题，就是你读完128bit的数据后，再继续读下一个128bit的数据非常麻烦，因为压缩的数据bit位置是不对齐的。比如第一个图中，第一个128bit包含了V0~V13一共14个数据，但是V14的部分bit在这个128bit中，这导致下一轮处理时，读取不到V14，从而非常难处理。

文章在实现章节，明确说了要求内存中的数据是128bit对齐的！