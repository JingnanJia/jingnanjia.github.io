---
layout:     post
title:      One Slow Memory Access Hash Table
subtitle:   用于多读的hash设计，仅一次内存访问
date:       2020-3-3
author:     Jia
header-img: img/Bloom.jpg
catalog: true
tags:
    - Hash

---

## 1. Background

- **Existing hash tables are read-heavy** :

​       Most of the operations to hash tables are queries.

- **Being critical for the worst-case query time to be well-bounded** : 

​       A large number of works have studied how to reduce the number of memory access per query in the worst case.



### Existing hash table

1. The first kind achieves constant lookup time at the cost of slow update and possibility of update failures.

![image-20200303230233945](C:\Users\JingnanJia\AppData\Roaming\Typora\typora-user-images\image-20200303230233945.png)

- Cuckoo hashing needs two memory accesses per query in the worst case.

Level hashing needs four memory accesses per query in the worst case. 

2. The Second kind of solutions achieve in average around one slow memory access per query by leveraging the fast-slow hierarchical memories.

<img src="C:\Users\JingnanJia\AppData\Roaming\Typora\typora-user-images\image-20200303230408298.png" alt="image-20200303230408298" style="zoom:80%;" />

- The second kind of solutions focus on using auxiliary data structure (e.g., Bloom filters) in the fast memory to reduce the number of accesses of slow memory.

- However, BFs have false positives, which means there could still be multiple slow memory accesses in the worst case.

> `Problem`: In summary, existing works either suffer from slow update and possibility of update failures, or do not improve the worst-case performance.

### **Goal**

```tex
The goal of this paper is to design a hash table that requires at most one memory access per query in the worst case, and supports fast incremental update without update failures.
```

## 2. One memory hash table (OMH)

OMH consists of three components: A main table in slow memory 、A fingerprint table in fast memory、 A stash in fast memory。The main table has some sub-tables with hash functions, and fingerprint table has the same structure and size as the main table, the only difference is that in main table, each bucket has a linked list.

<img src="C:\Users\JingnanJia\AppData\Roaming\Typora\typora-user-images\image-20200304130533476.png" alt="image-20200304130533476" style="zoom: 67%;" />

### OMH-insertion

When we insert <k1, v1>: OMH locates *d* candidate buckets through these hash functions, and being sequential to find the first empty bucket, so OMH inserts <k1, v1> into it, and for the other unchosen candidate buckets, OMH inserts <k1, v1> into the linked lists of them. , and if no empty buckets, insert the K-V into the stash, and if the stash is full, the OMH will add a new sub-table.

After insertion of main table, each bucket that stores a K-V pair, OMH chooses an adequate FP-hash to computer the fingerprint, then records the index of this FP-hash and the fingerprint of this K. 

- [x] How to choose the FP-hash? 

OMH will be sequential to find the adequate FP-hash, If and only if the fingerprint of k1 is different from these fingerprints of key in the linked list.

![image-20200304131246454](C:\Users\JingnanJia\AppData\Roaming\Typora\typora-user-images\image-20200304131246454.png)

### OMH-query

1). query K to fingerprint table( matches >1 or =0 ):

​     ①FP-hash(K) = FP-hash(k1)   and  FP-hash(K) = FP-hash(k2)   

​     ② FP-hash(K) ≠ FP-hash(k1)  and FP-hash(K) ≠ FP-hash(k2) 

  Queried K **doesn’t** exist.

2). query K to fingerprint table( matches = 1 ):

​     FP-hash(K) = FP-hash(k1)   and  FP-hash(K) ≠ FP-hash(k2)    

  Queried K **maybe** exist. *Need one memory access* to check its corresponding bucket in the main table.

![image-20200304131532624](C:\Users\JingnanJia\AppData\Roaming\Typora\typora-user-images\image-20200304131532624.png)



- [x] Why the paper use the linked list to record the conflicting keys? 

Because it is used to choose FP-hash to avoid collision of fingerprint.

### (OMH)-deletion

To delete a KV pair < kl , v1 >:  OMH first **queries** to find it, and then removes it.

## Evaluation

![image-20200304132625920](C:\Users\JingnanJia\AppData\Roaming\Typora\typora-user-images\image-20200304132625920.png)

As shown in Figure, the paper finds that OMH needs only one Average Memory Access (AMA).

## Summary

OMH builds exclusive fingerprints in the fast memory to guide query in the slow memory, and it needs one slow memory access per query in the worst case.

1. Strengths:

- Leveraging the fast-slow hierarchical memories: the access time of fast memory like SRAM is compared negligible to that of slow memory like DRAM.

- The key idea is to compute multiple fingerprints for a KV pair: choose one so that each KV pair has only one matched fingerprint. 

2. Weakness:

- Space overhead.
- Doesn’t discuss fast memory’s size and fingerprint’s size.
- Doesn’t specify the scenario.
- The process of insertion and deletion is not efficient.

### reference

- The paper just has two pages, it is a poster, from APNet2018, and the author’s homepage says that Full paper in submission to NSDI 2019, but I don’t find it. So I guess it maybe be rejected.

- The author *Tong yang*, is a professor from Peking University. He has a lot of work about the data structure, for example : tree, hash, bloom filter, are mainly used for networking. http://net.pku.edu.cn/~yangtong/
