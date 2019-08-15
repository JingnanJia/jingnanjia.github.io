---
layout:     post
title:      探索：Linear_search Practically Better Than Binary_search
subtitle:   SIMD线性查找算法 
date:       2019-8-14
author:     Jia
header-img: img/linearVSbinary.jpg
catalog: true
tags:
    - SIMD
    - Search 
    - exploratory
---

## 1. Background
假设我们有一个非常小的数组，我们希望能够在其中尽快找到满足相应条件的信息元素，现有的算法大致分为两类：无序查找和有序查找。
无序查找指线性的顺序查找，有序查找包括二分查找、基于树结构的查找（需要构造树，相对复杂）等，二分查找的时间复杂度为O(logn)。

因此，我们使用SIMD优化线性查找算法。

## 2. Binary_search
我们期望在长度为n的数组arr中返回首个大于或等于key的元素，std::lower_bound()已解决，其中arr为已排序数组。
```c
int binary_search_std (const int *arr, int n, int key) {
    return std::lower_bound(arr, arr + n, key) - arr;
}
```
上述中std::lower_bound()的实现包括if…else…分支跳转，不可预测的分支非常慢，因此应尽可能避免使用分支。下面的binary_search_branchless()实现了无分支跳转
，其满足的条件是n = 2^k - 1，通过使用步长step =2^logstep，这样无论比较结果如何，下一个搜索间隔都会有长度2^logstep。

```c
intptr_t MINUS_ONE = -1;
int binary_search_branchless (const int *arr, int n, int key) {
    assert((n & (n+1)) == 0);            
    intptr_t pos = MINUS_ONE;       
    intptr_t logstep = bsr(n);      
    intptr_t step = intptr_t(1) << logstep;
    while (step > 0) {
        pos = (arr[pos + step] < key ? pos + step : pos);
        step >>= 1; 
    }
    return pos + 1;
}
```
## 2. Linear_search

使用SIMD-AVX指令集进行优化线性查找，其核心思想是通过一次比较16个数据，大于等于key的返回0，否则返回1，然后通过对所有的0/1求和来得到结果的索引位置。
此代码了循环终止时的最后一次迭代之外，不存在其他的减慢算法速度的分支。
```c
int linear_search_avx (const int *arr, int n, int key) {
    assert(size_t(arr) % 32 == 0);
    __m256i vkey = _mm256_set1_epi32(key);
    __m256i cnt = _mm256_setzero_si256();
    for (int i = 0; i < n; i += 16) {
        __m256i mask0 = _mm256_cmpgt_epi32(vkey, _mm256_load_si256((__m256i *)&arr[i+0]));
        __m256i mask1 = _mm256_cmpgt_epi32(vkey, _mm256_load_si256((__m256i *)&arr[i+8]));

        __m256i sum = _mm256_add_epi32(mask0, mask1);
        cnt = _mm256_sub_epi32(cnt, sum);
    }
    __m128i xcnt = _mm_add_epi32(_mm256_extracti128_si256(cnt, 1), _mm256_castsi256_si128(cnt));
    xcnt = _mm_add_epi32(xcnt, _mm_shuffle_epi32(xcnt, SHUF(2, 3, 0, 1)));
    xcnt = _mm_add_epi32(xcnt, _mm_shuffle_epi32(xcnt, SHUF(1, 0, 3, 2)));
    return _mm_cvtsi128_si32(xcnt);
}
```
## 3. Comparison

以下是搜索实现的名称：
* binary_search_std：直接调用std :: lower_bound
* binary_search_branchless：无分支二分搜索
* linear_search_avx：线性搜索（AVX）

下面是三种算法的latency比较：
![image](https://raw.githubusercontent.com/JingnanJia/jingnanjia.github.io/master/img/linear_vs_binary.png)
结果分析：    
1. 对于二分搜索，无分支实现比分支实现快得多：延迟快两倍。我认为即使在无分支实现中增加了对非二次幂阵列长度的支持，结果仍然会如此。
2. 对于使用SIMD优化的线性搜索，在N <= 64时明显更快.   

## 4. Conclusion
对于长度小的数组，建议使用SIMD优化的Linear_search算法，其明显快于无分支的二分搜索算法binary_search_branchless。而对于更大长度的数组，binary_search_branchless相对更快。SIMD优化的Linear_search仅在少数场景（在已知数组长度非常小且搜索性能非常重要）下值得使用。此外，与Binary_search不同，SIMD优化的Linear_search算法很容易嵌入到一些矢量计算当中。

***
