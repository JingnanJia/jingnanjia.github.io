---
layout:     post
title:      Bloom filter 的概念及原理
subtitle:   浅谈 Bloom filter
date:       2019-7-20
author:     Jia
header-img: img/Bloom.jpg
catalog: true
tags:
    - Bloom filter
    - index
---


## 1.概念

Bloom filter是一种空间效率很高的数据索引结构，它利用bit数组很简洁地表示一个集合，Bloom filter 的主要用来判断某个或某些元素是否属于某个集合，在判断是否属于某个集合时，有可能会把不属于这个集合的元素误认为属于这个集合（false  positive）。因此，Bloom filter不适合那些“零错误”的应用场合。而在能容忍低错误率的应用场合下，Bloom filter可以通过极少的错误换取存储空间的极大节省。

## 2.原理

结合下图具体来看Bloom filter是如何通过使用位数组表示集合。
![](https://raw.githubusercontent.com/JingnanJia/jingnanjia.github.io/master/img/1.png)

①：初始状态，此时Bloom filter是一个新建的包含m位的位数组，每一位都置为0。

②：插入状态，为了将包含n个元素的集合S={x1, x2,…,xn}和Bloom filter建立关系，Bloom filter使用k个相互独立的哈希函数（Hash Function）（如图所示，使用了3个函数），k个哈希函数分别将集合中的每个元素映射到{1,…,m}的范围中（例如，对任意一个元素x，第i个哈希函数映射的位置Hashi(x)就会被置为1）。

③：查询状态，通过上述步骤，我们已将集合S={x1, x2,…,xn}和Bloom filter建立起映射关系，接下来就通过查询来判断某个元素是否属于集合S。如上图③所示，我们对待查询的元素y通过k次哈希函数找寻对应的k个bit位，如果k个位置都是1，那么我们就认为y是集合中的元素，否则就认为y不是集合中的元素（上图③中结果所示y1不属于当前集合，y2属于当前集合）。

## 3.与生俱来的错误率

错误率（false positive rate）：在判定某元素是否属于当前集合时，其结果有两种：属于或者不属于。对于“不属于”的判定结果，我们可以保证其正确性(false negative rate=0)；但是对于“属于”的判定结果，我们无法保证其正确性(false positive rate0)。

错误率形成原因：哈希函数的冲突所致，即属于当前集合的元素a和不属于当前集合的元素通过哈希函数所共享的bit位（即图③中的y2的判定结果是属于当前集合的，但是也可能是误判）。

量化错误率：量化错误率是一种估计，属于概率计算，用来评估当前Bloom filter 的质量。首先，Bloom filter具有m的bit位，k个hash函数，带插入集合有n个元素。然后是插入过程中产生错误率，即计算任意一个元素可能被Bloom filter误判的概率。

（1）m位数组中某特定bit位，当一个元素经一次hash处理没有被置为1的概率为（1-1/m），则n个元素经过k次hash函数处理后仍然未被置位的概率：

（2）此特定bit位在n个元素经过k个hash函数处理后被置位为1的概率：

（3）经过插入阶段，我们可知，某一特定bit位被置为1的概率如上式所示，则在查询阶段，某一个待查询的元素的k个bit位均为1，可判定该元素在当前集合中，但是同时也存在误判的概率（错误率），且误判的概率为：

（4）由极限准则可知，可将误判率公式简化为：

（5）求误判率最小值的参数，并且从上式可以看出，当m增大或者n减小时，都会使误判率更小：

即当k=ln2(m/n)时，此时的误判率最低。此时的Bloom filter的质量最好，优化后的误判率P(error)为：

## 4.存在的问题及解决方法

       问题：Bloom filter作为一种数据索引结构，仅支持插入和查询，不支持删除操作。原因：仍旧是Bloom filter中hash的冲突（有hash函数的地方就一定有冲突的存在）。

       解决方法：目前解决Bloom filter删除问题有很多种方法，比如Bloom filter[1]的变种Counting Bloom filter[2]、d-left Counting Bloom filter[3]，还有cuckoo filter[4]等，所列的这三项工作比较经典（参考文献见文章结尾），当然也有其它的工作，这里就不再进行延申了。

## 5.Bloom filter与其变体的性能对比

       Bloom filter与其变体的性能对比：空间开销、查询性能、是否支持删除。


## 6.Bloom filter的用法及应用场景

        用法：在应用Bloom filter时首先要确定两个参数，（1）即用户决定建立映射关系的元素集合的个数n；（2）用户希望的误判率大小P(error)。由上面我们第三步分析的结果可推导出此时Bloom filter所需bit的大小m：


再由满足最小误判率的条件可推导出此时所需的hash函数的个数k：


以上，为一个优化的Bloom filter的完整求解过程。

        应用场景：

        Bloom filter属于existence index，作为一种空间高效的数据索引结构，经常被数据库用于测试元素是否是某集合的成员。通常用于测试某些cold storage(例如disk)中是否存在某个元素，以减少cold storage的I/O次数。这里使用Bloom filter来提高查询性能，可以过滤掉一些“不必要的查询操作”，因为cold storage的一次I/O操作的性能开销非常大，（例如disk的一次I/O操作，包括寻道时间、旋转延迟和数据传输等三部分）。



### 参考文献：

- [1] B. Bloom, "Space/Time Trade-offs in Hash Coding with Allowable Errors", http://www.dragonwins.com/domains/getteched/bbc/literature/Bloom70.pdf,  1970.

- [2] L. Fan, et. al, "Summary Cache: A Scalable Wide-Area Web Cache Sharing Protocol", http://pages.cs.wisc.edu/~jussara/papers/00ton.pdf, Sep, 1998.

- [3] M. Mitzenmacher , et. al, "An Improved Construction for Counting Bloom Filters", http://www.eecs.harvard.edu/~michaelm/postscripts/esa2006b.pdf, 2006.

- [4] Bin Fan et. al, " Cuckoo Filter: Practically Better Than Bloom", https://www.cs.cmu.edu/~dga/papers/cuckoo-conext2014.pdf, Dec, 2014.

