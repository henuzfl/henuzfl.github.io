---
layout:     post
title:      "同步容器与并发容器(二):ConcurrentHashMap"
subtitle:   "ConcurrentHashMap"
date:       2017-11-15 12:00:00
author:     "zfl"
header-img: "img/vitruvian.jpg"
header-mask: 0.6
catalog:    true
tags:
    - java
    - 并发
    - map
    - 容器
--- 

>用来替代Hashtable    

### 1 锁分段
> 对一组独立对象上的锁进行分解，比如ConcurrentHashMap的实现使用了一个包含16个锁的数组，每个锁保护所有散列桶的1/16，其中N个散列桶由N mod 16个锁来保护。假如散列函数合理，并且关键字能够实现均匀分布，那么这大约能把对锁的请求减少到原来的1/16。 

在jdk中，ConcurrentHashMap是由**Segment**数组结构和HashEntry数组结构组成。Segment是一种**可重入锁**ReentrantLock，在ConcurrentHashMap里扮演锁的角色，HashEntry则用于存储键值对数据。一个ConcurrentHashMap里包含一个Segment数组，Segment的结构和HashMap类似，是一种数组和链表结构， 一个Segment里包含一个HashEntry数组，每个HashEntry是一个链表结构的元素， 每个Segment守护者一个HashEntry数组里的元素,当对HashEntry数组的数据进行修改时，必须首先获得它对应的Segment锁。结构如下：
![image.png](http://upload-images.jianshu.io/upload_images/730879-a2d4d712104f6cec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 2 jdk7中的关键源码

segment数组的初始化：
```java
if (concurrencyLevel > MAX_SEGMENTS)
    concurrencyLevel = MAX_SEGMENTS;
int sshift = 0;
int ssize = 1;
while (ssize < concurrencyLevel) {
    ++sshift;
    ssize <<= 1;
}
this.segmentShift = 32 - sshift;
this.segmentMask = ssize - 1;
if (initialCapacity > MAXIMUM_CAPACITY)
    initialCapacity = MAXIMUM_CAPACITY;
int c = initialCapacity / ssize;
if (c * ssize < initialCapacity)
    ++c;
int cap = MIN_SEGMENT_TABLE_CAPACITY;
while (cap < c)
    cap <<= 1;
Segment<K,V> s0 =
    new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                     (HashEntry<K,V>[])new HashEntry[cap]);
Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
```
默认情况下，concurrencyLevel=16，由上面的代码可知segments数组的长度ssize通过concurrencyLevel计算得出。为了能通过按位与的哈希算法来定位segments数组的索引，必须保证segments数组的长度是2的N次方（power-of-two size），所以必须计算出一个是大于或等于concurrencyLevel的最小的2的N次方值来作为segments数组的长度。假如concurrencyLevel等于14，15或16，ssize都会等于16，即容器里锁的个数也是16。
```java
private static int hash(int h) {
    // Spread bits to regularize both segment and index locations,
    // using variant of single-word Wang/Jenkins hash.
    h += (h <<  15) ^ 0xffffcd7d;
    h ^= (h >>> 10);
    h += (h <<   3);
    h ^= (h >>>  6);
    h += (h <<   2) + (h << 14);
    return h ^ (h >>> 16);
}
```
  

可以看到会首先使用Wang/Jenkins hash的变种算法对元素的hashCode进行一次再哈希，其目的是为了减少哈希冲突，使元素能够均匀的分布在不同的Segment上，从而提高容器的存取效率。  

```java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key.hashCode());
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject         
         (segments, (j << SSHIFT) + SBASE)) == null) 
        s = ensureSegment(j);
    return s.put(key, hash, value, false);
}
```
  

ConcurrentHashMap的插入操作，可以看出，首先计算hash，得到segment的编号，然后得到改变好segment，如果segment为空则初始化segment，然后将数据插入到segment中的HashEntry数组中。此类操作只需要获得一个segment上的显式锁即可，同理：remove、containKey、replace等操作也是如此。  

```java
public void clear() {
    final Segment<K,V>[] segments = this.segments;
    for (int j = 0; j < segments.length; ++j) {
        Segment<K,V> s = segmentAt(segments, j);
        if (s != null)
            s.clear();
    }
}
```
  

ConcurrentHashMap的clear操作，看一看出，需要遍历segment数组，然后执行segment的clear操作，也就是这一个操作需要获得segment数组所有的显式锁，同理：size、isEmpty等操作也是如此，其中size操作有一种情况例外，size操作先尝试2次通过不锁住Segment的方式来统计各个Segment大小，如果统计的过程中，容器的count发生了变化，则再采用加锁的方式来统计所有Segment的大小，判断容器是否变化还是采用modCount字段，核心源码如下：  

```java
 int retries = -1;
 for (;;) {
    if (retries++ == RETRIES_BEFORE_LOCK) {
    for (int j = 0; j < segments.length; ++j)
        ensureSegment(j).lock(); 
    }
    sum = 0L;
    size = 0;
    overflow = false;
    for (int j = 0; j < segments.length; ++j) {
        Segment<K,V> seg = segmentAt(segments, j);
        if (seg != null) {
           sum += seg.modCount;
           int c = seg.count;
           if (c < 0 || (size += c) < 0)
               overflow = true;
        }
    }
    if (sum == last)
       break;
    last = sum;
 }
```
### 3 额外的原子操作  

由于ConcurrentHashMap不能被加锁来执行独占访问，因此在客户端无法通过加锁来创建新的原子操作，因此一些常见的复合操作在ConcurrentMap中都已经有声明，比如复合操作：
* 如何没有则添加  

```java
V putIfAbsent(K key, V value)
```
* 只有当key被映射到V才移除
```java
boolean remove(Object key, Object value)
```
* 当key被映射到oldValue时才替换为newValue
```java
boolean replace(K key, V oldValue, V newValue)
```
* 当key被映射到某个值时才替换为newValue。
```java
V replace(K key, V value)
```
  

### 4 参考资料
* 《Java并发编程实战》
*  jdk7 源码