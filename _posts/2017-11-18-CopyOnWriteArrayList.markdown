---
layout:     post
title:      "同步容器与并发容器(三):CopyOnWriteArrayList"
subtitle:   "CopyOnWriteArrayList"
date:       2017-11-18 12:00:00
author:     "zfl"
header-img: "img/vitruvian.jpg"
header-mask: 0.6
catalog:    true
tags:
    - java
    - 并发
    - list 
    - 容器
--- 

>在遍历操作为主要操作的情况下替代同步的List，类似的CopyOnWriteArraySet来替代同步的Set，  

> CopyOnWrite即写入时复制，当然不止字面意义上的写入操作了，删除、更新等需要复制，容器的底层是一个数组，这个数组本身是不会修改的，也就时当容器需要修改时，会创建和发布这个数组的一个副本，然后在这个副本上进行更新操作，更新完成之后再赋值给本身的数组。多个线程可以同时对其迭代，而且不会抛出ConcurrentModificationException，返回的元素与创建迭代器的时候的元素一致。   
 
### 1 适用场景  

每次修改容器都会复制数组，会造成一定的开销，特别是当容器规模非常大的时候。仅当迭代操作远远多于修改操作的时候，才使用CopyOnWrite容器，比如说事件通知系统，需要迭代已经注册监听器链表，并且调用每一个监听器，注册和注销的操作远远少于接受事件通知的操作。  

### 2 存在的问题
* **内存占用率**：在容器进行更新操作操作的时候，内存里会同时驻扎两个底层数据对象的内存，旧的对象和新更新的对象。如果对象规模比较大，可能会造成频繁的垃圾回收，影响性能。
* **数据一致性**：CopyOnWrite容器只能保证数据的最终一致性，不能保证数据的实时一致性。所以如果你希望写入的的数据，马上能读到，请不要使用CopyOnWrite容器。
#### 关键源码解析
```java
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```
  

可以看出，容器在增加一个容器的时候，会新建一个比之前底层数组长度+1的新的数组，然后把之前数据元素复制给新数组，最后把新的数组赋值给之前的数组。其他更新操作与之类似。    
### 3 参考资料
* 《Java并发编程实战》
*  jdk7 源码