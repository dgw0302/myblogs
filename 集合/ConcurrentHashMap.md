---
title: ConcurrentHashMap
categories:
  - 集合
tags:
  - ConcurrentHashMap
abbrlink: 46153
date: 2022-04-14
cover : https://pic3.zhimg.com/v2-e883a05318653c2b5569ba41a08ca160_1440w.jpg?source=172ae18b
---

[TOC]



**数据结构**

ConcurrentHashMap的数据结构与HashMap基本类似，区别在于：

1、内部在数据写入时加了同步机制(分段锁)保证线程安全，读操作是无锁操作；

2、扩容时老数据的转移是并发执行的，这样扩容的效率更高。



**并发安全控制**

# jdk1.7



Java7 ConcurrentHashMap基于ReentrantLock实现分段锁





​    ![0](../../images/ConcurrentHashMap/16139)











# jdk1.8

## 相比于HashMap,通过什么提高效率与保证并发



synchronized + CAS







Java8中 ConcurrentHashMap基于分段锁+CAS保证线程安全，分段锁基于**synchronized**关键字实现；

​    ![0](../../images/ConcurrentHashMap/16135)

**源码原理分析**

**重要成员变量**

ConcurrentHashMap拥有出色的性能, 在真正掌握内部结构时, 先要掌握比较重要的成员:

- LOAD_FACTOR: 负载因子, 默认75%, 当table使用率达到75%时, 为减少table的hash碰撞, tabel长度将扩容一倍。负载因子计算: 元素总个数%table.lengh
- TREEIFY_THRESHOLD: 默认8, 当链表长度达到8时, 将结构转变为红黑树。
- UNTREEIFY_THRESHOLD: 默认6, 红黑树转变为链表的阈值。
- MIN_TRANSFER_STRIDE: 默认16, table扩容时, 每个线程最少迁移table的槽位个数。
- MOVED: 值为-1, 当Node.hash为MOVED时, 代表着table正在扩容
- TREEBIN, 置为-2, 代表此元素后接红黑树。
- nextTable: table迁移过程临时变量, 在迁移过程中将元素全部迁移到nextTable上。
- sizeCtl: 用来标志table初始化和扩容的,不同的取值代表着不同的含义:

- - - 0: table还没有被初始化
    - -1: table正在初始化
    - 小于-1: 实际值为resizeStamp(n)<
    - 大于0: 初始化完成后, 代表table最大存放元素的个数, 默认为0.75*n

- transferIndex: table容量从n扩到2n时, 是从索引n->1的元素开始迁移, transferIndex代表当前已经迁移的元素下标
- ForwardingNode: 一个特殊的Node节点, 其hashcode=MOVED, 代表着此时table正在做扩容操作。扩容期间, 若table某个元素为null, 那么该元素设置为ForwardingNode, 当下个线程向这个元素插入数据时, 检查hashcode=MOVED, 就会帮着扩容。

​    ConcurrentHashMap由三部分构成, table+链表+红黑树, 其中table是一个数组, 既然是数组, 必须要在使用时确定数组的大小, 当table存放的元素过多时, 就需要扩容, 以减少碰撞发生次数, 本文就讲解扩容的过程。扩容检查主要发生在插入元素(putVal())的过程:

- 一个线程插完元素后, 检查table使用率, 若超过阈值, 调用transfer进行扩容
- 一个线程插入数据时, 发现table对应元素的hash=MOVED, 那么调用helpTransfer()协助扩容。





















# synchronized加锁加在了哪里



**加在了哈希桶的第一个元素，也就是锁住了哈希桶**





直接看put源码



```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
/* put 调用 putVal 方法*/
final V putVal(K key, V value, boolean onlyIfAbsent) {
    //如果有空值或者空键，直接抛异常
    if (key == null || value == null) throw new NullPointerException();
    //基于key计算hash值，并进行一定的扰动
    int hash = spread(key.hashCode());
    //记录某个桶上元素的个数，如果超过8个，会转成红黑树
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //如果数组还未初始化，先对数组进行初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
	    //如果hash计算得到的桶位置没有元素，利用cas将元素添加
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //cas+自旋（和外侧的for构成自旋循环），保证元素添加安全
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        //如果hash计算得到的桶位置元素的hash值为MOVED，证明正在扩容，那么协助扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            //hash计算的桶位置元素不为空，且当前没有处于扩容操作，进行元素添加
            V oldVal = null;
            //对当前桶进行加锁，保证线程安全，执行元素添加操作
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    //普通链表节点
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    //树节点，将元素添加到红黑树中
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                //链表长度大于/等于8，将链表转成红黑树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                //如果是重复键，直接将旧值返回
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    //添加的是新元素，维护集合长度，并判断是否要进行扩容操作
    addCount(1L, binCount);
    return null;
}

/* 初始化底层数组 */
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    //cas+自旋，保证线程安全，对数组进行初始化操作
    while ((tab = table) == null || tab.length == 0) {
        //如果sizeCtl的值（-1）小于0，说明此时正在初始化， 让出cpu
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        //cas修改sizeCtl的值为-1，修改成功，进行数组初始化，失败，继续自旋
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    //sizeCtl为0，取默认长度16，否则去sizeCtl的值
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    //基于初始长度，构建数组对象
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    //计算扩容阈值，并赋值给sc
                    sc = n - (n >>> 2);
                }
            } finally {
                //将扩容阈值，赋值给sizeCtl
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}

```



concurrentHashmap完全基本与HashMap的逻辑一样，只不过通过锁保证了线程安全。

cmap有个for循环，多个线程会不断循环，直到拿到锁



ConcurrentHashmap的put方法里面有用的乐观锁和悲观锁

## CAS

ConcurrentHashmap的put方法里面

如果映射到的数组下标没有元素，说明还没有发生哈希冲突，此时用CAS操作来插入结点。



## 锁在了哪里

它锁的是桶，也就是当前线程要加到键值对的key经过计算后对应的数组下标当前的结点

**也可以说它锁的是映射到数组下标的第一个结点元素**



## for循环作用

**外层有个大的循环，`就是不断的尝试`**，自旋。

为什么要自旋？

当前CAS操作casTabAt可能会失败，所以需要不断自旋来重试。















# synchronized代替ReentranLock



**之前是分段锁，现在是锁桶，粒度更细**



JDK1.8为什么使用内置锁synchronized来代替重入锁ReentrantLock，有以下几点：

- 因为粒度降低了，在相对而言的低粒度加锁方式，synchronized并不比ReentrantLock差，在粗粒度加锁中ReentrantLock可能通过Condition来控制各个低粒度的边界，更加的灵活，而在低粒度中，Condition的优势就没有了。
- JVM的开发团队从来都没有放弃synchronized，而且基于JVM的synchronized优化空间更大，使用内嵌的关键字比使用API更加自然。
- 在大量的数据操作下，对于JVM的内存压力，基于API的ReentrantLock会开销更多的内存，虽然不是瓶颈，但是也是一个选择依据。
  







# ConcurrentHashMap如何使用CAS来提高效率





**第一个CAS：当通过key的哈希值取模数组长度算出元素下标后，如果当前下标位置没有元素，HashMap是直接插入新元素，而ConcurrentHashMap是CAS插入**







# ConcurrentHashMap如何保存其结点数量



直接看博客

https://www.cnblogs.com/gunduzi/p/13653505.html



```java
    private transient volatile long baseCount;

    private transient volatile CounterCell[] counterCells;

    @sun.misc.Contended static final class CounterCell {
        volatile long value;
        CounterCell(long x) { value = x; }
    }
```



ConcurrentHashMap就是依托上面三个东东进行计数的，那下面就详细解释一下这三个东东。

- **baseCount：最基础的计数，比如只有一个线程put操作，只需要通过CAS修改baseCount就可以了。**
- **counterCells：这是一个数组，里面放着CounterCell对象，这个类里面就一个属性，其使用方法是，在高并发的时候，多个线程都要进行计数，每个线程有一个探针hash值，通过这个hash值定位到数组桶的位置，如果这个位置有值就通过CAS修改CounterCell的value（如果修改失败，就换一个再试）,如果没有，就创建一个CounterCell对象。**
- **最后通过把桶中的所有对象的value值和baseCount求和得到总值，代码如下。**





```java
    final long sumCount() {
        CounterCell[] as = counterCells; CounterCell a;
        //baseCount作为基础值
        long sum = baseCount;
        if (as != null) {
            //遍历数组
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    //对每个value累加
                    sum += a.value;
            }
        }
        return sum;
    }
```

































## 总结

 ConcurrentHashMap计数思路是通过引入一个类似HashMap中用来存储节点的数组，利用数组来减少多线程对同一变量写操作的竞争。ConcurrentHashMap利用和线程绑定的Probe的值来快速计算对应的数组下标，如果下标处对象为null，通过了乐观锁+双重检查的形式对对象或者整个数组进行初始化。一般线程会对应counterCells中的某个数组下标对象进行累加，如果不存在别的线程的竞争，cas往往都会执行成功。当别的线程计算出的下标值是同一个，就存在对counterCells中的对象的竞争，此时，执行cas操作失败的线程会重新计算一个新的下标然后继续累加，如果还是存在竞争，继续更换下标，当多次失败的话，ConcurrentHashMap就认为当前数组中的大部分对象都有对应的线程在执行，会对整个数组进行扩容，原数组下标处的对象不变（可以认为counterCells中的对象往往只有一个线程对应，如果有多个线程，各个线程的cas操作也能高效的执行，否则，其中部分线程就会更换数组下标）。
























# HashTable



简单介绍



- 底层数组+链表实现，无论key还是value都**不能为null**，线程**安全**，实现线程安全的方式是在修改数据时锁住整个HashTable，效率低，ConcurrentHashMap做了相关优化
- 初始size为**11**，扩容：newsize = olesize*2+1
- 计算index的方法：index = (hash & 0x7FFFFFFF) % tab.length







**ConcurrentHashMap的效率要高于HashTable，因为HashTable是使用一把锁锁住整个链表结构从而实现线程安全。而ConcurrentHashMap的锁粒度更低，在JDK1.7中采用分段锁实现线程安全，在JDK1.8中采用CAS（无锁算法）+Synchronized实现线程安全。**





# concurrentHashMap扩容机制

















