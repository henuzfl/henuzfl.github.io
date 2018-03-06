---
layout:     post
title:      "同步容器与并发容器(七):ConcurrentSkipListMap"
subtitle:   "ConcurrentSkipListMap"
date:       2018-01-10 12:00:00
author:     "zfl"
header-img: "img/vitruvian.jpg"
header-mask: 0.6
catalog:    true
tags:
    - java
    - 并发
    - skipList 
    - 容器
--- 

>用来替代ConcurrentTreeMap  
> **排序平衡树**很好的替代品。但是，这样一种重要的数据结构在学校的算法课以及《算法导论》中都没有提及（据说在第五版会有），反而都用很大的篇幅去讲解实现起来特别复杂、对并发支持不是很友好的红黑树。   
 
在java的concurrent中ConcurrentSkipListMap和ConcurrentSkipListSet是用SkipList实现的，而且通过CAS模式实现了线程安全，是ConcurrentTreeMap和ConcurrentTreeSet的替代品。  
之前关于CAS这种非阻塞算法已经通过ConcurrentLinkedQueue做了介绍，遂本文将核心放到SkipList算法的实现上去。  
## SkipList
Skiplist（跳跃表）是一种可以替代平衡树的数据结构，**redis**、**LevelDB**等都是用Skiplist作为核心的数据结构，虽然SkipList在最坏的情况下退化为单链表时间复杂度为O(n)，大部分情况下和排序平衡树有着相同的效率，主要的是作为替代品相较于排序平衡树有**实现简单**、**内存占用量小**，特别是在面对并发开发时，由于平衡树在执行插入、删除、更新时需要rebalance的操作，这个操作时需要锁定整个表的，锁的粒度是比较大的，这就很影响性能，而Skiplist的使用链表为底层实现的，插入、删除操作都只影响前置节点、后置节点，很便于使用CAS这种方式实现线程安全（详情参见JDK中ConcurrentSkipListMap和ConcurrentSkipListSet的具体实现）。  
SkipList说白了是在链表上加入索引结构的一种数据结构，典型**空间换时间**的策略。链表是最简单的一种数据结构，采用何种方式的索引加持就可以成为一种高效、实用的SkipList呢？我们接下来一探究竟。  
## 实现原理  
下图是传统意义上的有序链表：  
![list1.png](http://upload-images.jianshu.io/upload_images/730879-b00b891f3836d271.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
查找的时间复杂度都是O(n),插入、删除依赖查找，时间复杂度O(n)，下图所展示的就是一种跳跃表，在一些节点上面增加了索引，通过查找索引就可以将查找的时间复杂度降为O(n/2)，下图演示了查找数字12的过程，第一步先和6比较，大于6则第二步和9比较，大于9第三步和17比较，小于17说明12在9和17之间，第四步退到9，第五步再和12比较，可以看出这次查找过程跳过了3,7的比较；
![list2.png](http://upload-images.jianshu.io/upload_images/730879-7fd0e411dfdcc98a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
其实，上面基本上就是跳跃表的思想，每一个节点不单单只包含指向下一个节点的指针，可能包含很**多个指向后续节点**的指针，这样就可以跳过一些不必要的节点，从而加快查找、删除等操作。  
至此，原理是不是很简单的，链表上节点随机的增加一些指针指向的后续节点，下面我们重点看一下jdk是如何实现的。  
## 源码解析    
这篇文章主要是为了讲明白实现算法的，jdk源码实现中的CAS部分有很大的干扰作用，因此，在下面的源码解析中我都去掉了控制CAS这些代码，可以参照源码看。  

* jdk数据结构示意图：
    ![image.png](http://upload-images.jianshu.io/upload_images/730879-0d99c2f112743403.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
### 初始化
```java
    private void initialize() {
        keySet = null;
        entrySet = null;
        values = null;
        descendingMap = null;
        head = new HeadIndex<K,V>(new Node<K,V>(null, BASE_HEADER, null),
                                  null, null, 1);
    }
```  

初始化就很简单，相当于创建一个特殊的头节点，然后创建一个层级为1的头索引。   
### findPredecessor操作  
在第一层的索引中找到小于给定key的最右的一个索引对应的节点
```java
    private Node<K,V> findPredecessor(Object key, Comparator<? super K> cmp) {
        if (key == null)
            throw new NullPointerException(); 
        // 从头索引节点开始，查找的方向是先右后下，如果存在右索引，判断右索引节点的key与给定key的大小，如果比给定key小，继续向右，否则向下。直到到达第一层的索引，返回该索引对应的节点。     
        for(Index<K,V> q = head, r = q.right, d;;){
            if (r != null) {
                Node<K,V> n = r.node;
                K k = n.key;
                if (cpr(cmp, key, k) > 0) {
                    q = r;
                    r = r.right;
                    continue;
                }
            }
            if ((d = q.down) == null)
                return q.node;
            q = d;
            r = d.right;
        }
    }
```  
上述示例中，我们要找到key=8的processor步骤如下图：
![捕获.PNG](http://upload-images.jianshu.io/upload_images/730879-a5d17c07230febc2.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 查询操作
```java
   private V doGet(Object key) {
        if (key == null)
            throw new NullPointerException();
        Comparator<? super K> cmp = comparator;
        // 第一步找到Predecessor节点
        for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
                Object v; int c;
                if (n == null)
                    break outer;
                // 实现cas代码段，判断数据是否由其他线程修改
                Node<K,V> f = n.next;
                // 如果找到该节点，则返回该节点的value值
                if ((c = cpr(cmp, key, n.key)) == 0) {
                    @SuppressWarnings("unchecked") V vv = (V)v;
                    return vv;
                }
                // 如果比给定的key值还要小，则说明链表中不存在该数据，返回null
                if (c < 0)
                    break;
                // 如果比给定key值大，那么继续向后续节点搜索
                b = n;
                n = f;
        }
        return null;
    }
```
OK，了解完findPredecessor操作之后再看doGet操作就比较清晰明了了，从找到的这个节点出发，向后跟给定的key作对比，如果比较结果相等则返回结果，如果比给定的key值大，则继续向后执行，如果比给定的key值小，说明不存在该节点。比较令人困惑的就是上述代码中cas的那部分代码了，确实很是干扰，如果不看那部分，还是很清晰的。
### 插入操作
这段代码看起来特别复杂，主要是掺杂了CAS的实现，其实主要分成了三个部分：
1. 将新节点插入到链表中  

2. 新节点开始创建索引节点  

3. 索引节点的连接  

现在开始一部分一部分的分析。 
* **第一步：将新节点插入到链表中**
```java
        Node<K,V> z;             // 新增的节点
        if (key == null)
            throw new NullPointerException();
        Comparator<? super K> cmp = comparator;
        for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
            if (n != null) {
                Object v; int c;
                Node<K,V> f = n.next;
                // 如果比给定的key值大，则继续向后检索    
                if ((c = cpr(cmp, key, n.key)) > 0) {
                    b = n;
                    n = f;
                    continue;
                }
                // 如果与给定的key值相等，如果不存在才插入，直接返回结果；如果存在也插入，那么将value赋值给节点n，返回结果。插入操作结束。
                if (c == 0) {
                    if (onlyIfAbsent || n.casValue(v, value)) {
                        @SuppressWarnings("unchecked") V vv = (V)v;
                        return vv;
                    }
                }
            }

            // 将新节点插入到b与n之间    
            z = new Node<K,V>(key, value, n);
            if (!b.casNext(n, z))
                break;
        }
```  
* **第二步：新节点开始创建索引节点**  
```java
        // 生成一个随机的level
        int rnd = ThreadLocalRandom.nextSecondarySeed();
        if ((rnd & 0x80000001) == 0) { 
            int level = 1, max;
            while (((rnd >>>= 1) & 1) != 0)
                ++level;
            Index<K,V> idx = null;
            HeadIndex<K,V> h = head;
            // 如果level小于等于目前的层级数，那么从level从第一层向上开始创建新建节点的索引节点
            if (level <= (max = h.level)) {
                for (int i = 1; i <= level; ++i)
                    idx = new Index<K,V>(z, idx, null);
            }
            // 如果level大于目前的层级数，那么每次整体层级数增加一层
            else { 
                level = max + 1;
                // 从level从第一层向上开始创建新建节点的索引
                @SuppressWarnings("unchecked")Index<K,V>[] idxs =
                    (Index<K,V>[])new Index<?,?>[level+1];
                for (int i = 1; i <= level; ++i)
                    idxs[i] = idx = new Index<K,V>(z, idx, null);
                // 更新头索引
                h = head;
                int oldLevel = h.level;
                if (level > oldLevel){
                    // 创建新头索引节点
                    HeadIndex<K,V> newh = h;
                    Node<K,V> oldbase = h.node;
                    for (int j = oldLevel+1; j <= level; ++j)
                        newh = new HeadIndex<K,V>(oldbase, newh, idxs[j], j);
                    // 新的头索引赋值给之前的head
                    if (casHead(h, newh)) {
                        h = newh;
                        idx = idxs[level = oldLevel];
                        break;
                    }
                }
            }
```
* **第三步：索引节点的连接**
```java
            int insertionLevel = level;  // 新建索引的层级
            int j = h.level; //目前的整体层级
            // 从头索引节点开始，建立到新加入节点的索引节点的连接
            for (Index<K,V> q = h, r = q.right, t = idx;;) {
                // 如果当前索引节点的右索引节点不为空，那么判断是否需要向右继续查找，找到距离当前节点索引节点最接近的索引节点
                if (r != null) {
                    Node<K,V> n = r.node;
                    int c = cpr(cmp, key, n.key);
                    //如果给定key大于当前右索引节点的值，那么继续向右查找 
                    if (c > 0) {
                        q = r;
                        r = r.right;
                        continue;
                    }
                }
                // 如果当前左边的索引与当前节点的索引在同一层级上，则进行关联
                if (j == insertionLevel) {
                    q.link(r, t)  // 之前是q->r,现在link之后变成q->t->r
                    // 新加入索引降一层，如果当前是第一层，那么退出循环
                    if (--insertionLevel == 0)
                        break;
                }
                // 当前新加入索引以及当前需要连接的左边的索引都降一层进行循环
                if (--j >= insertionLevel && j < level)
                    t = t.down;
                q = q.down;
                r = q.right;
            }
```  


### 5 参考资料
* 《Java并发编程实战》
*  jdk7 源码 
*  Pugh W. Skip lists: A probabilistic alternative to balanced trees[C]// The Workshop on Algorithms and Data Structures. 1989:668-676.