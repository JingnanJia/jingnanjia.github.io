---
layout:     post
title:      探索：解决Bloom filter删除问题的一种可行思路
subtitle:   有关Bloom filter不支持删除的问题
date:       2019-7-25
author:     Jia
header-img: img/Bloom_delete.jpg
catalog: true
tags:
    - Bloom filter
    - index
    - exploratory
---


## 前言

目前解决Bloom filter删除问题有很多种方法，比如Bloom filter的变种Counting Bloom filter、d-left Counting Bloom filter，还有Cuckoo filter等，但是这些方法基本上都是对Bloom filter本身算法做出或大或小的改变，同样的也牺牲了一部分性能。以上三种方法的原理及细节均可以通过阅读论文或查找相关资料进行学习（这里不做过多的介绍）。接下来，介绍不同于以上方法的另一种可行思路！

>在计算机系统结构中，存在这样一句话： **加上一个缓存层可以解决体系结构大部分问题！**

## 1. 探索

在解决Bloom filter不支持删除的问题上，我们借鉴这种思路，添加一个缓存层用于存储删除的元素。添加的缓存层应满足的需求：①在查找时应具备与Bloom filter相匹配的性能，以便并发操作时可以给出结果（即当前元素是否已被删除）②空间开销应尽可能的小。

满足上述两个条件，我的第一反应是Bloom filter索引结构本身，即我们将需要删除的元素插入另一个Bloom filter当中。

## 2. 将Bloom filter视作“缓存”

基于以上探索，将Bloom filter需要删除的元素插入另一个Bloom filter(我们分别称之为insert Bloom filter，insert Bloom filter)。如下图所示，我们需要建立映射关系的元素插入insert Bloom filter，将需要从集合中删除的元素插入delete Bloom filter，其基本思想是将需要删除的元素转而插入另一个Bloom filter当中。

**插入和删除操作**：insert Bloom filter支持插入操作，delete Bloom filter支持删除操作（其实也是插入操作，相对于insert Bloom filter是删除操作）。
![image](https://raw.githubusercontent.com/JingnanJia/jingnanjia.github.io/master/img/32.png)
**查询操作**：查询操作的演示如下图所示（通过乘法和异或运算得出最终结果）：
![image](https://raw.githubusercontent.com/JingnanJia/jingnanjia.github.io/master/img/31.png)
**But！Non false negative无法保证**：将删除元素插入另外一个Bloom filter存在的问题——Non false negative无法保证！例如：当我首先向insert Bloom filter发起写操作时，即插入一个a元素后，这时紧接着发起了查询操作，然后发现delete Bloom filter返回的结果显示元素a已被删除，实际上并未执行过删除操作。产生结果的原因是Bloom filter的hash冲突！！！
![image](https://raw.githubusercontent.com/JingnanJia/jingnanjia.github.io/master/img/33.png)
如下所示，经过进一步的测试分析得知:
在0~40%的删除范围内，false positive rate几乎保持不变，false negative rate几乎为零；
在40%~100%的删除范围内，false positive rate逐渐下降，与此同时false negative rate逐渐上升；
![image](https://raw.githubusercontent.com/JingnanJia/jingnanjia.github.io/master/img/34.png)

## 3. 进一步探索

将需要从集合中删除的元素插入delete Bloom filter中，虽然解决了Bloom filter的删除问题，但是却无法保证Non false negative！其原因是hash的冲突！所以，我们需要避开hash的冲突，即将需要从集合中删除的元素插入一个不包含hash冲突的数据索引结构当中，且同时满足我们上述提到的要求，例如：tree、linklist等，而且如果需要的话，原Bloom filter和插入的另一个数据索引结构在系统空闲时间可以rebuild，以减少空间开销和时间开销等。

## 4. 带来的好处

**优势**：这种插入、删除操作分离的复合结构的好处在于支持高并发，即当处于一个multiple write、multiple read、multiple update的应用当中，可以很好的支持高并发以提高系统性能！

***
