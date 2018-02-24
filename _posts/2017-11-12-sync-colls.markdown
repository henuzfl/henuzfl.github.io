---
layout:     post
title:      "同步容器与并发容器(一):同步容器"
subtitle:   "同步容器"
date:       2017-11-12 12:00:00
author:     "zfl"
header-img: "img/vitruvian.jpg"
header-mask: 0.6
catalog:    true
tags:
    - java
    - 并发
    - 同步
    - 容器
---  

在java中，同步容器包括两类：  

1. Vector、Stack、Hashtable  

2. Collections类中提供的静态工厂方法创建的类，比如通过Collections.synchronizedList创建一个线程安全的list。  


这些同步容器都是使用**synchronized**进行了同步，Vector类似于ArrayList，可以看成可以自增的数组。Stack继承与Vector，其实就是Vector的特殊实现。Hashtable很奇怪，命名方式都不符合Java的规范，可见有多老，使用同步实现了Map接口。Collections的静态工厂方法就是将线程不安全的List、Set、Map使用**代理设计模式**增加了同步操作。  

### 1 安全性问题  

同步容器是线程安全的这个说法没有问题，add、remove、size等操作都使用synchronized进行了同步。当时对于**复合操作**来说，就需要额外的加锁来保护复合操作。常见的复合操作有：迭代、跳转（根据指定顺序找到当前元素的下一个元素）以及**先检查后执行**操作。

#### 1.1 ConcurrentModificationException异常  

当容器在迭代的过程中被修改，就会抛出这个异常。具体的实现是通过一个容器变化计数器modCount（当add、remove操作时该字段+1），在迭代之前，赋值expectedModCount=modCount，然后迭代过程中如果迭代器外部有修改了容器，则二者的值就会不一样，每次迭代的时候都会检测这两个值是否相当。除此之外，还要注意**隐藏迭代器**，比如容器的toString，retainAll，containsAll，removeAll等操作，实际上就包含了迭代操作。

为了解决同步容器复合操作带来的安全性问题，可以使用对容器加锁，还要特别注意包含隐藏迭代器的操作，这种情况下也需要加锁。同步中迭代操作除了加锁还有一种替代方法是**clone容器**，并在副本上进行迭代，由于副本封闭在线程内，其他线程不会再其迭代的期间对其修改，就会避免抛出ConcurrentModificationException。当然了这种操作会存在显著的性能开销，要综合各个因素来考虑。  

### 2 性能问题  

同步容器的访问都是线性化的，也就是无论什么时候都只有一个线程可以访问容器的状态，以此来实现线程的安全性。另外，为了使用同步容器过程中的复合操作也是线程安全的，就需要在复合操作的过程中在外层对容器加锁，特别是针对于迭代操作，数据量比较大的情况下在迭代的过程中无法访问容器。  
### 3 并发容器
由于同步容器的安全性以及性能问题，在jdk5提供了多种并发容器类用来替代同步容器，可以极大地提高伸缩性并降低风险。接下来几篇文章将介绍一下主要的并发容器。

### 参考资料
* 《Java并发编程实战》