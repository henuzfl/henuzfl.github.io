---
layout:     post
title:      "同步容器与并发容器(五):ConcurrentLinkedQueue"
subtitle:   "ConcurrentLinkedQueue"
date:       2017-12-25 12:00:00
author:     "zfl"
header-img: "img/vitruvian.jpg"
header-mask: 0.6
catalog:    true
tags:
    - java
    - 并发
    - queue 
    - 非阻塞
    - cas 
    - 容器
--- 

### 1 非阻塞算法
> 一个线程的失败和挂起不会引起其他些线程的失败和挂起，这样的算法称为非阻塞算法。非阻塞算法通过使用底层机器级别的原子指令来取代锁，从而保证数据在并发访问下的一致性。  

* 从 **Amdahl 定律**我们可以知道，要想提高并发性，就应该尽量使串行部分达到最大程度的并行；也就是说：最小化串行代码的粒度是提高并发性能的关键。 与锁相比，非阻塞算法在**更细粒度**（机器级别的原子指令）的层面协调多线程间的竞争。它使得多个线程在竞争相同资源时不会发生阻塞，它的并发性与锁相比有了质的提高；同时也大大减少了线程调度的开销。同时，由于几乎所有的同步原语都只能对单个变量进行操作，这个限制导致非阻塞算法的设计和实现非常复杂。  
非阻塞算法的实现需要借助冲突检查机制来判断更新过程中是否存在来自其他线程的干扰，如果存在，这个操作将失败，并且可以重试。在打不多数处理器架构中采用的方法是实现一个**CAS指令**。  

### 2 CAS（Compare and Swap）  
> CAS包含了3个操作数--需要读写的内存位置V、进行比较的值A和要写入的新值B。当且仅当V==A时，CAS指令才会通过原子方式用新值B来更新V的值，否则不进行任何操作。无论位置V的值是否等于A，都将返回V原有的值。其实就是，我认为V的值应该是A，如果是，则将V的值更新为B，如果不是则不修改告诉我V的值是多少。    

CAS的**典型使用模式**：首先从V中读取值A，并根据A计算新值B，然后再通过CAS指令以原子的方式将V中的值由A变成B，如果值更新失败，那么得到CAS指令返回的当前V中的值A1，然后根据A1计算新值B1，再次通过CAS指令执行更新操作，直到CAS指令更新成功。


### 3 队列非阻塞算法的实现
这一块的设计和实现特别复杂，jdk中只要使用了CAS算法的都源码看起来都非常难懂，而且想要设计使用非阻塞算法的数据结构，最好还是由专家来完成（《Java并发编程实战》中的原话）。当时，通过对这个容器核心源码的分析，一方面是因为这一块的设计确实非常的精妙，搞懂之后非常过瘾，另一方面通过学习源码的实现可以在以后的工作中面对并发任务有更多的选择。  

**为什么如此复杂**：因为使用链表实现队列的这种数据结构，需要头节点和尾节点的快速访问，需要两个指针头节点指针和尾节点指针，同时链表结构还需要有节点指针来维持链式结构。比如当增加一个元素时，需要两步操作：    

**1.维持链式结构，也就是将当前尾节点的next指向新的元素节点。**  

**2.更改尾指针的指向，指向新的元素节点**    


同时当出队操作时，也需要两步操作：  

**1. 将头节点置空** 

**2. 头指针指向当前的头节点**  

初看起来，这两个操作是无法原子执行的，因为这两个操作都需要两个不同的CAS操作，并且第一个CAS成功了没有办法保证第二个CAS也成功，就算两个CAS操作都执行成功了也没有办法保证两个CAS操作之间没有其他线程访问这个队列，就会造成数据不一致的情况。
为了解决上述问题，一个技巧就是如果当B到达时发现A正在修改数据结构，那么在数据结构中应用足够多的信息，使得B能完成A的更新操作。如果B帮助A完成了更新操作，那么B可以执行自己的操作，而不用等待A的操作完成，当A恢复后再师徒完成其操作时，会发现B已经替它完成了。
比如当前是一个队列的示意图：
![image.png](http://upload-images.jianshu.io/upload_images/730879-3f7f71566d39c438.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如果有两个线程（线程1、线程2）同时进行入队操作，分别入队元素D、E，假如线程1首先访问队列，需要执行两个步骤：
![image.png](http://upload-images.jianshu.io/upload_images/730879-17c97c79163e3d79.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
假如线程1在执行步骤1执行成功之后线程挂掉了，此时线程2访问队列时就会先执行线程1剩余的步骤2，然后在执行线程2自己的两步操作。
![image.png](http://upload-images.jianshu.io/upload_images/730879-eb6a4e81ac7104e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
入队操作类似，不再赘述。这只是算法的描述，下面就看看jdk源码的具体实现。

### 4 源码分析

#### 4.1 入队操作
```java
    public boolean offer(E e) {
        checkNotNull(e);
        final Node<E> newNode = new Node<E>(e);
        
        // (1)
        for (Node<E> t = tail, p = t;;) {
            Node<E> q = p.next;
            // (2)
            // (2.1)
            if (q == null) {
                // (2.1.1) 
                if (p.casNext(null, newNode)) {
                    // (2.1.2) 
                    if (p != t)  
                        casTail(t, newNode); // 将tail指针指向新元素的节点  
                    return true;
                }
            }
            // （2.2）
            else if (p == q)
                p = (t != (t = tail)) ? t : head;
            // (2.3)
            else
                p = (p != t && t != (t = tail)) ? t : q;
        }
    }
```
  
(1) 入队操作，只有当链表上加入新元素的节点才会跳出循环  
(2) 有三个条件分支，分别代表不同的处理方式。  
(2.1)检查是否有其他线程正在修改数据结构，如果q == null 说明p是最后一个节点
(2.1.1) 将新节点通过cas操作加入到链式结构中去，直到成功为止。  
(2.1.2) 更新tail指针，不论是否更新tail指针成功，都返回结果。如果此时p和t都指向最后一个元素，说明有其他线程已经做过caseTail操作了，则不需要再进行此操作了。  
(2.2) 这个步骤很疑惑，q是p的next，怎么可能相等呢,在找资料的过程中有两种说法，一说是因为都是空节点，这很明显是错误的，q==null条件不满足才跳到这里的；还有一说是因为tail的next会指向自己，这一点不确定，因为看源码之中没有将tail的next指向自己的操作。这段暂且**略过**。  
(2.3) p不是最后一个节点，此时判断tail是否已经更新，如果已经更新，那么将p指向最新的tail，如果没有更新，则指向q节点，其实就是尽可能保证下次迭代过程中p指向最后的节点。

#### 4.2 出队操作
```java
    public E poll() {
        restartFromHead:
        // (1)
        for (;;) {
            for (Node<E> h = head, p = h, q;;) {
                E item = p.item;
                // (2)
                // (2.1)
                if (item != null && p.casItem(item, null)) {
                    // (2.1.1)
                    if (p != h) 
                        updateHead(h, ((q = p.next) != null) ? q : p);
                    return item;
                }
                // (2.2)
                else if ((q = p.next) == null) {
                    updateHead(h, p);
                    return null;
                }
                // (2.3)
                else if (p == q)
                    continue restartFromHead;
                // (2.4)
                else
                    p = q;
            }
        }
    }

    final void updateHead(Node<E> h, Node<E> p) {
        if (h != p && casHead(h, p))
            h.lazySetNext(h);
    }
```
  
  （1）出队操作，直到返回结果才跳出循环，要么返回头节点的数据，要么返回空。  
  （2）有四个条件分支，分表代表不同条件下的处理方式。  
   (2.1) 如果头节点的item不为空，并且成功的将头节点的item设置为空，如果不能成功设置为空，那么将继续外侧for循环，重新获得head并执行。   
   (2.1.1) 更新head指针，如果有next为空则直接指向null节点，如果不为空则指向next节点。如果p==h，说明已经有其他线程执行过该操作，则不需要再执行。  
   (2.2) head的item为null说明此时头节点已经执行了上述算法描述中的步骤1或者队列为空，此时判断head的next为空说明队列为空，则头节点指向空节点并返回null。   
   (2.3) 和入队的疑惑是一样的，本来是不应该相当的，**略过**。  
   (2.4) 这个判断条件成立说明头节点的item已经被其他线程置空，当时还没有来得及更新头指针指向或者更新失败，此时返回下一个节点再次进行出队操作。 

### 5 参考资料
* 《Java并发编程实战》
*  jdk7 源码 