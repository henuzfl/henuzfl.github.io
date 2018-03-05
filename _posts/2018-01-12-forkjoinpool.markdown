---
layout:     post
title:      "ForkJoinPool框架学习"
subtitle:   "ForkJoinPool"
date:       2018-01-12 12:00:00
author:     "zfl"
header-img: "img/vitruvian.jpg"
header-mask: 0.6
catalog:    true
tags:
    - java
    - 并发
    - ForkJoinPool
    - 分治 
--- 

### 1 分治算法  
> 《算法导论》中介绍的第一个算法，字面上就是“分而治之”的意思，就是把一个复杂的问题分成两个或更多的相同的子问题，知道最后子问题可以很简单的直接求解，原问题的解即子问题解的合并。  
> 很多高效的算法都使用了分治算法的思想，如快速排序、归并排序、快速傅里叶变换等。   
分治算法一般的步骤：  

1. 分解：将原问题分解为若干个规模较小，相互独立，与原问题形式相同的子问题；
2. 解决：若子问题规模较小而容易被解决则直接解，否则递归地解各个子问题
3. 合并：将各个子问题的解合并为原问题的解。  
归并排序的伪码：
```java
    // A是要排序的数据，p起始位置，r结束位置
    void mergeSort(A,p,r)
        // 如果任务不够小，将任务分解为两个任务
        if p < r:
            q = (p + r) / 2
            mergeSort(A,p,q)
            mergeSort(A,q+1,r)
            // 归并结果
            merge(A,p,q,r)

    void merge(A,p,q,r):
        L[0...(q-p)] = A[p...q]
        L[0...(r-q)] = A[q...r]
        L[q-p+1] = MAX
        R[r-q+1] = MAX
        i = 0
        j = 0
        for k = p to r
            if L[i] <= R[j]
                A[k] = L[i]
                i++
            else
                A[k] = R[j]
                j++      
```

 
### 2 Frok/Join框架  
分治算法提出了一种问题的解决思路，并没有带了性能上的提升，在《算法导论》27.3多线程归并排序已经将归并排序的并行化做了实现和分析。  
jdk7中新提供了Fork/Join框架的实现，得益于大神 Doug Lea 的论文——《A Java Fork/Join Framework》，得以一窥其貌。  

Fork/Join算法，如同其他分治算法一样，总是会递归的、反复的划分子任务，直到这些子任务可以用足够简单的、短小的顺序方法来执行。
```java
    Result solve(Problem problem) {
    if (problem is small)
        directly solve problem
        else {
            split problem into independent parts
            fork new subtasks to solve each part
            join all subtasks
            compose result from subresults
        }
}
```  


图示：
![image.png](http://upload-images.jianshu.io/upload_images/730879-d522ef64578f1727.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

