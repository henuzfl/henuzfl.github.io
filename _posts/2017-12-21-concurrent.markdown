---
layout:     post
title:      "同步容器与并发容器"
subtitle:   "同步容器与并发容器"
date:       2017-12-21 12:00:00
author:     "zfl"
header-img: "img/vitruvian.jpg"
header-mask: 0.6
catalog:    true
tags:
    - java
    - 并发
    - 容器
    - CAS
---

## 1.同步容器  

在java中，同步容器包括两类：  

1. Vector、Stack、Hashtable  

2. Collections类中提供的静态工厂方法创建的类，比如通过Collections.synchronizedList创建一个线程安全的list。  


这些同步容器都是使用**synchronized**进行了同步，Vector类似于ArrayList，可以看成可以自增的数组。Stack继承与Vector，其实就是Vector的特殊实现。Hashtable很奇怪，命名方式都不符合Java的规范，可见有多老，使用同步实现了Map接口。Collections的静态工厂方法就是将线程不安全的List、Set、Map使用**代理设计模式**增加了同步操作。  

### 1.1 安全性问题  

同步容器是线程安全的这个说法没有问题，add、remove、size等操作都使用synchronized进行了同步。当时对于**复合操作**来说，就需要额外的加锁来保护复合操作。常见的复合操作有：迭代、跳转（根据指定顺序找到当前元素的下一个元素）以及**先检查后执行**操作。

#### ConcurrentModificationException异常  

当容器在迭代的过程中被修改，就会抛出这个异常。具体的实现是通过一个容器变化计数器modCount（当add、remove操作时该字段+1），在迭代之前，赋值expectedModCount=modCount，然后迭代过程中如果迭代器外部有修改了容器，则二者的值就会不一样，每次迭代的时候都会检测这两个值是否相当。除此之外，还要注意**隐藏迭代器**，比如容器的toString，retainAll，containsAll，removeAll等操作，实际上就包含了迭代操作。

为了解决同步容器复合操作带来的安全性问题，可以使用对容器加锁，还要特别注意包含隐藏迭代器的操作，这种情况下也需要加锁。同步中迭代操作除了加锁还有一种替代方法是**clone容器**，并在副本上进行迭代，由于副本封闭在线程内，其他线程不会再其迭代的期间对其修改，就会避免抛出ConcurrentModificationException。当然了这种操作会存在显著的性能开销，要综合各个因素来考虑。  

### 1.2 性能问题  

同步容器的访问都是线性化的，也就是无论什么时候都只有一个线程可以访问容器的状态，以此来实现线程的安全性。另外，为了使用同步容器过程中的复合操作也是线程安全的，就需要在复合操作的过程中在外层对容器加锁，特别是针对于迭代操作，数据量比较大的情况下在迭代的过程中无法访问容器。

## 2.并发容器  

由于同步容器的安全性以及性能问题，在jdk5提供了多种并发容器类用来替代同步容器，可以极大地提高伸缩性并降低风险。
### 2.1 ConcurrentHashMap  

>用来替代Hashtable    

#### 锁分段
> 对一组独立对象上的锁进行分解，比如ConcurrentHashMap的实现使用了一个包含16个锁的数组，每个锁保护所有散列桶的1/16，其中N个散列桶由N mod 16个锁来保护。假如散列函数合理，并且关键字能够实现均匀分布，那么这大约能把对锁的请求减少到原来的1/16。 

在jdk中，ConcurrentHashMap是由**Segment**数组结构和HashEntry数组结构组成。Segment是一种可重入锁ReentrantLock，在ConcurrentHashMap里扮演锁的角色，HashEntry则用于存储键值对数据。一个ConcurrentHashMap里包含一个Segment数组，Segment的结构和HashMap类似，是一种数组和链表结构， 一个Segment里包含一个HashEntry数组，每个HashEntry是一个链表结构的元素， 每个Segment守护者一个HashEntry数组里的元素,当对HashEntry数组的数据进行修改时，必须首先获得它对应的Segment锁。结构如下：
![image.png](http://upload-images.jianshu.io/upload_images/730879-a2d4d712104f6cec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### jdk7中的关键源码

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
#### 额外的原子操作  

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



### 2.2 CopyOnWriteArrayList
>在遍历操作为主要操作的情况下替代同步的List，类似的CopyOnWriteArraySet来替代同步的Set，  

> CopyOnWrite即写入时复制，当然不止字面意义上的写入操作了，删除、更新等需要复制，容器的底层是一个数组，这个数组本身是不会修改的，也就时当容器需要修改时，会创建和发布这个数组的一个副本，然后在这个副本上进行更新操作，更新完成之后再赋值给本身的数组。多个线程可以同时对其迭代，而且不会抛出ConcurrentModificationException，返回的元素与创建迭代器的时候的元素一致。   
 

* **适用场景**：每次修改容器都会复制数组，会造成一定的开销，特别是当容器规模非常大的时候。仅当迭代操作远远多于修改操作的时候，才使用CopyOnWrite容器，比如说事件通知系统，需要迭代已经注册监听器链表，并且调用每一个监听器，注册和注销的操作远远少于接受事件通知的操作。  

#### 存在的问题
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

### 2.3 BlockingQueue  

队列这种数据结构就不多说了，FIFO，BlockingQueue是一种阻塞队列，BlockingQueue提供了可阻塞的put和take方法，以及支持定时的offer和poll方法，所谓阻塞，如果队列为空，那么take方法将会阻塞知道有元素可用，如果队列满了，那么put方法将阻塞知道有空间可用。  

#### 生产者-消费者模式  

生产者生产完数据之后不用等待消费者处理，而是将生产的数据放入一个“待处理”的列表等待消费者处理，生产者无需关心消费者，消费者也无需关心生产者，这种模式简化了开发过程，能够实现生产者和消费者的解耦，双方不再直接进行通信而，而是通过一个共享工作队列而实现的。  
BlockingQueue简化了生产者-消费者模式设计的实现过程，它支持任意数量的生产者和消费者，BlockingQueue平衡了生产者消费者之间的处理能力，当生产数据能力较弱时，阻塞队列中没有数据，消费者take操作会一直阻塞知道队列中有元素；当消费能力较弱时，如果使用的是有界队列，那么队列很快会满，生产者的put操作就会一直阻塞直到有空间可用。
其中，LinkedBlockingQueue和ArrayBlockingQueue是FIFO队列，二者与LinkedList和ArrayLIst类似，分别通过链表、顺序表实现，当时比同步list有更好的并发性能。
另外还有PriorityBlockingQueue，可以按照设定的某种顺序而不是FIFO的按优先级排序的队列。

#### 关键的源码
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
### 2.4 ConcurrentLinkedQueue
#### 非阻塞算法
> 一个线程的失败和挂起不会引起其他些线程的失败和挂起，这样的算法称为非阻塞算法。非阻塞算法通过使用底层机器级别的原子指令来取代锁，从而保证数据在并发访问下的一致性。  

* 从 **Amdahl 定律**我们可以知道，要想提高并发性，就应该尽量使串行部分达到最大程度的并行；也就是说：最小化串行代码的粒度是提高并发性能的关键。 与锁相比，非阻塞算法在**更细粒度**（机器级别的原子指令）的层面协调多线程间的竞争。它使得多个线程在竞争相同资源时不会发生阻塞，它的并发性与锁相比有了质的提高；同时也大大减少了线程调度的开销。同时，由于几乎所有的同步原语都只能对单个变量进行操作，这个限制导致非阻塞算法的设计和实现非常复杂。  
> CAS（Compare and Swap）

#### 队列非阻塞算法的实现
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

#### 源码分析

##### 入队操作
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

##### 出队操作
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

### 2.5 ConcurrentSkipListMap
>用来替代ConcurrentTreeMap

### 参考资料
* 《Java并发编程实战》
*  jdk7 源码