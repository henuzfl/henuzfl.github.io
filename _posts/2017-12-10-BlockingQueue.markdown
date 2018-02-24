---
layout:     post
title:      "同步容器与并发容器(四):BlockingQueue"
subtitle:   "BlockingQueue"
date:       2017-12-10 12:00:00
author:     "zfl"
header-img: "img/vitruvian.jpg"
header-mask: 0.6
catalog:    true
tags:
    - java
    - 并发
    - queue 
    - 阻塞 
    - 容器
--- 

队列这种数据结构就不多说了，FIFO，BlockingQueue是一种阻塞队列，BlockingQueue提供了可阻塞的put和take方法，以及支持定时的offer和poll方法，所谓阻塞，如果队列为空，那么take方法将会阻塞知道有元素可用，如果队列满了，那么put方法将阻塞知道有空间可用。  

### 1 生产者-消费者模式  

生产者生产完数据之后不用等待消费者处理，而是将生产的数据放入一个“待处理”的列表等待消费者处理，生产者无需关心消费者，消费者也无需关心生产者，这种模式简化了开发过程，能够实现生产者和消费者的解耦，双方不再直接进行通信而，而是通过一个共享工作队列而实现的。  
BlockingQueue简化了生产者-消费者模式设计的实现过程，它支持任意数量的生产者和消费者，BlockingQueue平衡了生产者消费者之间的处理能力，当生产数据能力较弱时，阻塞队列中没有数据，消费者take操作会一直阻塞知道队列中有元素；当消费能力较弱时，如果使用的是有界队列，那么队列很快会满，生产者的put操作就会一直阻塞直到有空间可用。
其中，LinkedBlockingQueue和ArrayBlockingQueue是FIFO队列，二者与LinkedList和ArrayLIst类似，分别通过链表、顺序表实现，当时比同步list有更好的并发性能。
另外还有PriorityBlockingQueue，可以按照设定的某种顺序而不是FIFO的按优先级排序的队列。

### 2 关键的源码
```java
    private final ReentrantLock takeLock = new ReentrantLock();
    
    private final Condition notEmpty = takeLock.newCondition();

    private final ReentrantLock putLock = new ReentrantLock();

    private final Condition notFull = putLock.newCondition();
```
使用两个Condition，分别为notFoll和notEmpty。用于表示“非满”和非空
两个条件谓词。当队列为空时，take将阻塞并等待notEmpty，此时put向notEmpty发送信息，可以解除任何在take中阻塞的线程。同理，当队列的已满的时候，put将阻塞并等待notFull，此时take向notFull发送消息，可以解除任何在put中阻塞的线程。
```java
    public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) {
                notEmpty.await();
            }
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }

    private void signalNotFull() {
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            notFull.signal();
        } finally {
            putLock.unlock();
        }
    }
```
                                              
  
  ```java
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            while (count.get() == capacity) {
                notFull.await();
            }
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
    }

    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }
```   
### 3 参考资料
* 《Java并发编程实战》
*  jdk7 源码