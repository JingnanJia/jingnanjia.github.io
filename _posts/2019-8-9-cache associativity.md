---
layout:     post
title:      漫画：Cache Associativity
subtitle:   简明介绍cache的几种映射方法
date:       2019-8-9
author:     Jia
header-img: img/Bloom.jpg
catalog: true
tags:
    - cache
---

## 1. cache功能、映射方法

cache是一种高速缓冲寄存器，是为解决CPU和主存之间速度不匹配而采用的一项重要技术。主存与cache的地址映射方式有直接方式、组相联方式和全相联方式三种。

* 直接映射(directmapping)：将一个主存块存储到唯一的一个Cache行；

* 组相联映射(setassociative mapping)：可以将一个主存块存储到唯一的一个Cache组中任意一个行；

* 全相联映射(fullyassociative mapping)：可以将一个主存块存储到任意一个Cache行。

## 2. 

![image](http://csillustrated.berkeley.edu/PDFs/posters/cache-3-associativity-poster.pdf)
一个虚拟地址分为三个部分 tag. index. offset。
index就像找到书架中编号为index的某一个格子。如果你要找这本书，那么这本书的虚拟地址告诉你只可能出现在这个格子。但是有很多书都可能映射到一个格子中，这个格子中可能没你要的书，所以要用tag来比较下你找到的格子里是不是确实是你要的书。tag通常取自地址的前几位（格子越少，越容易发生地址冲突，需要的tag的位数也就越多，组关联set-associative映射的tag位数就比direct-map位数多）。然后offset就是在得到的书中具体找哪一页。那一页就是CPU真正产生出来要你去找到的东西








