---
layout:     post
title:      Learned Bloom filter
subtitle:   Machine Learning for Bloom filter
date:       2019-7-24
author:     Jia
header-img: img/Bloom.jpg
catalog: true
tags:
    - Bloom filter
    - index
    - Machine Learning
---


## 1. 背景

2018年，谷歌AI部门在SIGMOD会议（数据库领域的顶会）上发表的一篇exploratory research paper，论文名为《The Case for Learned Index Structures》，值得一提的是谷歌AI部门主要负责人Jeffrey Dean挂名其中，使得论文引起学术界和工业界的广泛关注。
>**“AI 与计算机系统架构”**
- Jeff Dean 等人从 2018 年 3 月开始发起 SysML 会议，聚焦于机器学习/深度学习相关的硬件基础设施和计算机系统。那么 AI 到底能够给计算机系统架构带来哪些
新的机会？
- “阿里计算平台掌门人”贾扬清认为可以从以下两个方面来看:Jeff Dean 在提到 SysML 的时候，其实提过这样一个概念，就是 Machine  Learning  for  Systems  and  Systems  for   Machine  Learning。今天我们做的更多的是 System  for  Machine  Learning，指的是当机器学习有这样一个需求的时候，我们怎么去构建一个系统来满足它的需求。另一方面，在计算机系统构建的过程中，我们还可以考虑怎么通过机器学习的方法跟数据驱动的方法来优化和设计系统，解决原来系统设计对人的经验的依赖问题，这是 Machine  Learning  for  System 可以解决的事情，不过目前这方面还处于相对比较早期的探索阶段，也是人工智能接下来还需要突破的瓶颈之一。

好了，言归正传。Learned Index的论文主要讲的是利用Machine Learning优化数据索引结构的性能，如空间性能、时间性能等，即Machine Learning for System 。而本文主要讲的是论文中优化Bloom filter的这部分（想阅读更多的内容见参考文献），利用Machine Learning 降低Bloom filter的空间开销。

## 2. Motivation

[上一篇博文](https://jingnanjia.github.io/2019/07/20/%E6%B5%85%E8%B0%88Bloomfilter/)我已经介绍了Bloom filter作为一种空间高效的数据索引结构，经常被数据库用于测试元素是否是某集合的成员。通常用于测试某些cold storage(例如disk)中是否存在某个元素。虽然布卢姆过滤器具有很高的空间效率，但它仍然占用大量内存。例如，对于10亿条记录，大约需要1.76Gigabytes的内存；对于0.01%的FPR(false positive rate，误判率)，我们大约需要2.23 Gigabytes的内存。所以，进一步降低Bloom filter的空间开销显得尤为重要。

## 3. 单一模型

Bloom filter视作二元分类问题：Bloom filter本身是用来判断某个元素是否属于某集合，可以将这个过程视作一个二元概率分类任务。也就是说，可以学习一个模型f(x)，它可以预测待查询的元素是“属于集合”还是“不属于集合”。例如，对于字符串，我们可以训练递归神经网络(RNN)或卷积神经网络(CNN)。但是，对于RNN或者CNN来说，其预测结果必定存在误差，即fpr(false positive rate)0且fnr(false negative rate)0。如此，直接把Bloom filter当作一个二元分类任务进行处理，显然不具有任何意义，因为Bloom filter存在的意义就是fnr=0，使得“不属于”的判定结果为真。

显然，单一模型并不能解决问题！！！

## 4. 复合模型

Bloom filter与二元分类相结合：为了确保学习化的Bloom filter与标准的Bloom filter具有一样的特性，即①fnr=0；②可设置预期的fpr大小，从而保证Bloom filter索引的质量。因此，文章提出通过设置阈值的方法来保证fnr=0，并且用户可以自定义预期的fpr。

（1）fnr=0的保证：如下图所示，Model为一个RNN或者CNN模型f(x)，f(x)为模型预测的结果值（f(x)[0,1]）。通过设置阈值，当f(x)时，我们可以认为当前元素key在集合中（允许fpr0）；当f(x)时，我们可以认为当前元素key不在集合中（fnr0，但不被允许），此时我们得到一个元素均满足f(x)的集合K，将集合K映射为一个Bloom filter，我们称之为overflow Bloom filter，当再次查询元素key的f(x)时，我们通过overflow Bloom filter来判定key是否属于集合（如下图右半部分所示，通过overflow Bloom filter判定的结果没有false negative，即fnr=0）。这样的复合模型设计使得fnr=0得到的保证，复合模型命名为Learned Bloom filter。
![image](https://raw.githubusercontent.com/JingnanJia/jingnanjia.github.io/master/img/11.png)
（2）可设置预期的fpr大小：集合表示Learned Bloom filter训练前的“不属于当前集合”的元素的整体，则我们可以在训练完Model（上图中左半部分）并确定阈值之后计算Model的FPR大小，如下所示：
![image](https://raw.githubusercontent.com/JingnanJia/jingnanjia.github.io/master/img/12.png)
overflow Bloom filter的预期误判率设置为FPRb，则整体的复合模型的预期误判率设置为FPRp，则必须满足：

为了简化结果（右边取等号是为了最小化空间开销），我们取：

则我们可以通过设定阈值来决定整体的预期误判率FPRp（同时也包括决定复合模型的FPR，FPRb）。

## 5. 实验结果

从实验结果我们可以看出，Learned Bloom filter在fpr允许的大范围内节省了原有Bloom filter占据的内存空间开销（图中的W和E是RNN的参数）。

> **Space(Learned Bloom filter) = Space(RNN模型) + Space(overflow Bloom filter)**


## 6. 总结

我们通过以上分析可以看出，Machine Learning现阶段还很难取代传统的数据索引结构，取而代之的是复合型的模型。但是，Bloom filter的一大特点就是性能快，而Learned Bloom filter的查找时间显然要比传统的Bloom filter大上两三个数量级（本人实验已验证），所以文章里也提及了Learned Bloom filter的适用场景，即在用户访问某些例如磁盘时，允许较大的延时。以下为二者之间的性能对比：


### 参考文献：

- [1] Kraska et. al, "The Case for Learned Index Structures", 
  [https://dl.acm.org/citation.cfm?id=3196909, 2018.](https://dl.acm.org/citation.cfm?id=3196909)

