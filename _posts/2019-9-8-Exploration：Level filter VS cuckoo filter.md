---
layout:     post
title:      Exploration：Level filter VS. cuckoo filter
subtitle:   Level filter is a variant of level hashing
date:       2019-9-8
author:     Jia
header-img: img/level_head.jpg
catalog: true
tags:
    - level hashing
    - cuckoo filter
    - SIMD
---


## 前言
   近些年来，非易失性内存一直是存储领域比较热门的一个话题。非易失性内存可以提供非易失性，写入非易失性内存的数据在断电之后依然能够保持下来，因而可以将其认为是一种新型的存储设备。

将哈希表应用于非易失性内存，有三个主要的挑战：

* **保证一致性需要很大开销**。由于非易失性内存的访问粒度很小，多次不同地址的更新无法保证同时被持久化。且由于硬件缓存的影响，CPU发出的store指令并不会马上将数据发送给非易失性内存，因而需要手动调用耗时的CLFLUSH/CLFLUSHOPT/CLWB等指令才能保证写出数据的持久化。在非易失性内存的实际使用中，多使用日志（logging），写时拷贝（copy-on-write）等方法配合CLFLUSH等指令保证一致性，而这会导致大量的额外操作，影响性能；

* **大量的写会降低性能**。除了上述的CLFLUSH指令之外，大多数非易失性内存技术的写入延迟要高于读取延迟，而现有哈希表操作中通常有大量的数据移动操作，严重影响性能；

* **哈希表在调整大小时开销很大**。随着负载因子（load factor）的增大，为了保证访问性能减少冲突，哈希表需要调整大小（resize）。传统的方法一般需要分配一个大小为旧表两倍的空间的新哈希表，然后把所有的键值对都重新哈希到新表中。这会造成大量的写入操作，影响性能和增加非易失性内存的磨损。

   针对以上三个问题，我们组的大师兄左鹏飞提出一种叫做层次化哈希（level hashing）(文章见参考文献)的哈希方法。这种方法在非易失性内存上对写入操作专门优化，用很小的代价来保证一致性，且能够高效地调整哈希表的大小。

## 1. level hashing
   层次化哈希表的结构如图所示。根据Facebook和Baidu发布的键值负载，大多数的键值对的大小小于一个硬件缓存行大小，为了提高硬件缓存的利用效率，层次化哈希表在一个桶（bucket）中放置多个槽位（slot），因而一个桶中可以放多个键值对。此外，对于每个键，有两个哈希函数与其对应（图中hash1和hash2）。也就是说有两个桶可以用来存放这个键所对应的键值对。最后，整个哈希表的结构分为上下两层（图中的TL和BL）。每两个相邻的桶对应了下层的一个桶，即上层的两个桶2n和2n+1对应了下层的n。在实际存储中，上下两层分别为长度为N和N/2的一维数组。
![image](https://raw.githubusercontent.com/JingnanJia/jingnanjia.github.io/master/img/level(4).png)
   层次化哈希能保证在插入的时候，大多数情况下只需要最多一次数据移动，只有在很少情况下才需要先调整大小再进行插入。在进行插入x时，先查看两个哈希函数（hash1(x)和hash2(x)）对应的上层桶是否有空闲的槽位，如果有则将键值对直接放入槽位，插入完成；如果两个上层槽位都满了，则检查在这两个都满了的桶中，看是否有键值对可以放置到其另外一个哈希函数的上层桶中（如键k在hash1(x)或者hash2(x)桶中，则看其能否被移动到hash1(k)或者hash2(k)的桶中），如果有，则将其移动过去，那么当前需要插入的键值对就有空间可以存放了；如果没有，则进一步检查hash1(k)和hash2(k)所对应的下层桶是否有空位（即检查hash1(x)/2和hash2(x)/2桶），如果有，则可直接插入；否则，进一步检查两个下层桶中的键值对，是否能被移动到其对应的其他下层桶中，如果可以，则将其移动过去，以给新插入的键值对腾出地方；如果依然没有，则整个插入过程失败。如果插入失败，则需要将整个哈希表变大一倍之后进行插入。从上述的步骤可以看出，一个成功的插入，最多只会迁移一个已经存在的键值对。因而这个方法可以保证哈希表插入的效率。

由于上述的插入方法，给定一个键，其有可能被放置到四个桶内（两个上层桶，两个下层桶），因而在查询的时候需要将这个四个桶全部都遍历一遍。
![image](https://raw.githubusercontent.com/JingnanJia/jingnanjia.github.io/master/img/level(6).png)
   另外一个很重要的操作是哈希表的调整大小。这里以增大为例，在进行增大操作时，加入原哈希表的上层桶数为N，则分配一个新的2N个桶的数组，将其作为新的上层，原来的上层则作为新的下层，而原来的下层则成为过渡层（图中IL）。此后，需要将过渡层中的所有键值对重新哈希（rehash）到新的上层中。当所有的键值对都被重新哈希到上层之后，可以将过渡层的空间释放掉。哈希表恢复为两层，这个哈希表调整过程结束。在这个调整过程中，被重新哈希的键值约为整个哈希表中键值对总数的1/3，因而这种方法优于传统的调整方法。

## 2. cuckoo filter
Cuckoo Filter是Cuckoo Hash的变体，其核心思想是设置每个桶对应4个entry，借助计算指纹的哈希函数，通过两个哈希函数 ，映射出两个候选位置i1和i2。如果两个位置均已有元素，则采取“踢”操作，且cuckoo filter可能会造成无限的循环踢除操作，因此一般程序会设置一个踢除次数的上限（例如，500次），当超过上限后就需要重新resize。所以Cuckoo filter具有以下三点缺点(如下图左图所示，有图表示的是踢除操作)：
 * 吞吐量方面具有严重的尾延迟；
 * 存在大量的数据移动操作（踢除操作），即写操作；
 * resize之后，插入吞吐量集中在尾部；
![image](https://raw.githubusercontent.com/JingnanJia/jingnanjia.github.io/master/img/level(8).png)
## 3. level filter
   由于level hashing本身具备的写操作少，且in-place方式的resize特点，这些特点可以用于优化现有的cuckoo filter的插入操作。如下图所示，我们将cuckoo filter分层，并且限制cuckoo filter的踢除操作，且每层最多产生一次数据移动操作，减少原本大量的数据移动，这样带来的好处是，减少了数据的写操作，且每次resize只需resize原来1/3的数据量；但是同时降低了hash table的负载率，cuckoo filter的负载率可以达到95%，level filter的负载率可以达到90%（减少了数据移动，降低了负载率，仍处于不错的范围）。

   由于level hashing的插入顺序始终是从上层到下层，于是，在插入过程中加入一个负载因子参数，当负载因子达到一定值时（例如，60%）就从下层开始插入，减少上层的遍历过程，大大提高了插入吞吐量。
![image](https://raw.githubusercontent.com/JingnanJia/jingnanjia.github.io/master/img/level(1).png)
## 4. insert throughtput：Level filter VS. cuckoo filter
![image](https://raw.githubusercontent.com/JingnanJia/jingnanjia.github.io/master/img/level(5).png)
从上图我们可以看出优化过后的level filter的平均插入吞吐量相比较Cuckoo filter提高至1.5 ×~ 1.7×。

在查询性能方面，level filter最多查询四个buckets，而Cuckoo filter最多查询两个buckets，为了弥补之间的差距，可以使用SIMD指令进行优化。

### 参考文献：
- [1] Level hashing: Zuo P, Hua Y, Wu J. Write-optimized and high-performance hashing index scheme for persistent memory[C]//13th {USENIX} Symposium on Operating Systems Design and Implementation ({OSDI} 18). 2018: 461-476.
- [2] Cuckoo filter: Fan B, Andersen D G, Kaminsky M, et al. Cuckoo filter: Practically better than bloom[C]//Proceedings of the 10th ACM International on Conference on emerging Networking Experiments and Technologies. ACM, 2014: 75-88.


