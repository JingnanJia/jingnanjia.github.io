---
layout:     post
title:      漫画：Cache Associativity
subtitle:   简明介绍cache的几种映射方法
date:       2019-8-9
author:     Jia
header-img: img/cache_header.jpg
catalog: true
tags:
    - cache
---

## 1. cache功能、映射方法

Cache是一种高速缓冲寄存器，是为解决CPU和主存之间读写速度不匹配而采用的一项重要技术。主存与cache之间的地址映射方式有直接方式、组相联方式和全相联方式三种。

* 直接映射(direct mapping)：将一个主存块存储到唯一的一个Cache行；

* 组相联映射(set associative mapping)：可以将一个主存块存储到唯一的一个Cache组中任意一个行；

* 全相联映射(fully associative mapping)：可以将一个主存块存储到任意一个Cache行。

## 2. 漫画：三种映射方式
![image](https://raw.githubusercontent.com/JingnanJia/jingnanjia.github.io/master/img/cache_associativity.png)

一个虚拟地址分为三个部分 Tag、Index、Offset。     

如上图漫画所示：    

①index就像找到书架中编号为index的某一个格子，如果你要找某本书，那么这本书的虚拟地址告诉你只可能出现在这个index格子中。但是，有很多书都可能映射到一个格子中，这个格子中也可能没你要的书，所以要用tag来比较下你找到的格子里是不是确实是你要找的书；    

②tag通常取自地址的前几位（格子越少，越容易发生地址冲突，需要的tag的位数也就越多，组关联set-associative映射的tag位数就比direct-map位数多）；    

③offset就是在得到的书中具体找哪一页，那一页就是CPU真正要你去找到的东西。    

### reference
[1] Cache Associativity: [http://csillustrated.berkeley.edu/PDFs/posters/cache-3-associativity-poster.pdf](http://csillustrated.berkeley.edu/PDFs/posters/cache-3-associativity-poster.pdf)

***
