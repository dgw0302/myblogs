---
title: AQS
categories:
  - 多线程与并发
tags:
  - AQS
abbrlink: 46153
date: 2022-04-12 19:54:20be
---



[TOC]

本文来讲讲AQS

AQS是AbstractQueuedSynchronizer的缩写，这个AbstractQueuedSynchronizer是一个类，也叫抽象队列同步器，它是一个可以用来实现线程同步的基础框架。



像一些我们平常锁用的**ReentrantLock/Semaphore/CountDownLatch**底层都是基于AQS;



本文主要基于**ReentrantLock**来看看这个AQS。



# 美团技术文章

https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html

# ****





# 锁分类





![image-20220822155533159](../../images/AQS/image-20220822155533159.png)





 









# 简单介绍

**美团文章**：https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html



**AQS是一种提供了原子式管理同步状态、阻塞和唤醒线程功能以及队列模型的简单框架。**



# AbstractQueuedSynchronizer

AQS维护了一个int型变量**state**（volatile修饰）和一个**同步队列**（CLH)

**抽象的队列式同步器**



**AQS具备的特性：**

- 阻塞等待队列
- 共享/独占
- 公平/非公平
- 可重入
- 允许中断

**这几点贯穿AQS全文**

## JUC应用场景



| 同步工具               | 同步工具与AQS的关联                                          |
| :--------------------- | :----------------------------------------------------------- |
| ReentrantLock          | 使用AQS保存锁重复持有的次数。当一个线程获取锁时，ReentrantLock记录当前获得锁的线程标识，用于检测是否重复获取，以及错误线程试图解锁操作时异常情况的处理。 |
| Semaphore              | 使用AQS同步状态来保存信号量的当前计数。tryRelease会增加计数，acquireShared会减少计数。 |
| CountDownLatch         | 使用AQS同步状态来表示计数。计数为0时，所有的Acquire操作（CountDownLatch的await方法）才可以通过。 |
| ReentrantReadWriteLock | 使用AQS同步状态中的16位保存写锁持有的次数，剩下的16位用于保存读锁的持有次数。 |
| ThreadPoolExecutor     | Worker利用AQS同步状态实现对独占线程变量的设置（tryAcquire和tryRelease）。 |









## state

- state:来记录当前的同步状态，也可以理解为锁的状态，根据state的值的不同，可以判断当前锁是否已经被获取。一个线程修改state成功，则表示它成功获取了锁。

  ![image-20220411221249820](../../images/AQS/image-20220411221249820.png)

  

  

  **对于独占锁，这个属性每次重入就会加一**

  **对于共享锁，他就是资源的个数，当它减到0，后续的线程就不能获取到锁了。**

  

  




## 同步队列

- CLH：通过这个同步队列来维护当前获取锁失败而进入阻塞状态的线程。这个同步队列是一个双向链表，获取锁失败的线程会被封装成一个Node结点，加入链表的尾部排队，而AQS保存了链表的头结点的引用head以及链表的为结点的引用。

![image-20220411221241245](../../images/AQS/image-20220411221241245.png)

**数据结构：双向链表**



AQS可以用来实现独占锁，也可以用来实现共享锁



- **独占锁**：也叫排他锁，即锁只能由一个线程获取，若一个线程获取了锁，则其他想要获取锁的线程只能等待，直到锁被释放。比如说写锁，对于写操作，每次只能由一个线程进行，若多个线程同时进行写操作，将很可能出现线程安全问题；
- **共享锁**：锁可以由多个线程同时获取，锁被获取一次，则锁的计数器+1。比较典型的就是读锁，读操作并不会产生副作用，所以可以允许多个线程同时对数据进行读操作，而不会有线程安全问题，当然，前提是这个过程中没有线程在进行写操作；



**ReentrantLock是独占锁**



**PS:只有同步队列里面的线程才能获得锁**

## 条件队列

数据结构：担心链表

调用await（）的时候会释放锁，然后线程会加入到条件队列，调用signal()唤醒的时候会把从条件队列中的线程结点移动到同步队列中，等待再次获得锁





> 可见，条件队列为单向列表，只有指向下一个节点的引用；没有被唤醒的节点全部存储在条件队列上。上图描述的是一个长度为 5 的条件队列，即有5个线程执行了`await()`方法；与阻塞队列不同，条件队列没有常驻内存的“head结点”，且一个处于正常状态节点的`waitStatus`为 -2 。当有新节点加入时，将会追加至队列尾部，等调用signal()唤醒的时候会把从条件队列中的线程结点移动到同步队列中，等待再次获得锁





## waitStatus队列结点状态

**waitStatus**



| 0              | 初始化状态，表示当前结点在sync队列中，等待着获取锁           |
| -------------- | ------------------------------------------------------------ |
| CANCELLED = 1  | 表示当前线程被取消                                           |
| SIGNAL = -1    | 值为-1，后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者被取消，那么就会通知后继节点，让后继节点的线程能够运行，表示后继结点在等待当前结点唤醒。后继结点入队时，会将前继结点的状态更新为SIGNAL。 |
| CONDITION = -2 | 值为-2，节点在条件队列中，节点线程等待在Condition上，不过当其他的线程对Condition调用了signal()方法后，该节点就会从条件队列转移到同步队列中，然后开始尝试对同步状态的获取 |
| PROPAGATE = -3 | 值为-3，表示下一次的共享式同步状态获取将会无条件的被传播下去，共享模式下，前继结点不仅会唤醒其后继结点，同时也可能会唤醒后继的后继结点。 |



重要标志：**SIGNAL —— -1，表示：当当前节点释放锁的时候，需要唤醒下一个节点。**



**注意，负值表示结点处于有效等待状态，而正值表示结点已被取消。所以源码中很多地方用>0、<0来判断结点的状态是否正常**



## Condition

pS:只有持有锁的线程才可以await和signal





### **await**

Condition的作用用一句话概括就是为了实现线程的等待（await）和唤醒（signal），多线程情况下为什么需要等待唤醒机制？原因是有些线程执行到某个阶段需要等待符合某个条件才可以继续执行，在之前学习操作系统的时候，有一个经典的场景就是在容量有限的缓冲区实现生产者消费者模型，如果缓冲区满了，这个时候生产者就不能再生产了，就要阻塞等待消费者消费，当缓冲区为空了，消费者就要阻塞等待生产者生产，这就是一个很典型的使用condition实现条件状态的场景。



> 当线程执行await，意味着当前线程一定是持有锁的，首先会把当前线程放入到条件队列队尾，之后把当前线程的锁释放掉（还会阻塞该线程），当当前线程释放锁之后，阻塞队列的第二个节点会获取到锁（正常情况下），当前持有锁的节点是首节点，当释放锁之后，首节点会被干掉，也就是说执行await的线程会从阻塞队列中干掉。



**主要操作：、将当前持有锁的线程加入条件队列、并释放锁资源、然后调用park()方法使线程进入等待状态**



AQS有一个阻塞队列，把没有获取到锁的线程都放到这个队列中，但AQS中其实还有别的队列，那就是等待队列，就是放执行await之后的线程，大家看上面的例子可以发现，执行了这么一段代码：

```java
final ReentrantLock lock = new ReentrantLock();
final` `Condition condition = lock.newCondition();
```

这里是new了一个Condition，这段代码就会在AQS中创建一个条件队列，那如果多次执行上面的代码，就会在AQS中创建多个条件队列



**PS:await底层用到LockSupport.park()方法**



### signal

移入到同步队列后才有机会使得等待线程被唤醒

**signal只把条件队列第一个结点放进同步队列中，而signalAll是一个一个的把条件队列的结点放进同步队列中**

signalAll和signal的区别就是signalAll会执行一个循环，把等待队列中的所有节点都执行一遍signal，就是说把所有等待队列中的节点全部加入到阻塞队列中，之后还是要一个节点一个节点的去慢慢获取锁。





**主要操作：signal唤醒条件队列队头的第一个线程，然后将该线程从等待队列中转移到同步队列，去等待尝试获取锁**









**PS:signal底层用到LockSupport.unpark()方法**



调用condition的signal的前提条件是 当前线程已经获取了lock，该方法会使得等待队列中的头节点(等待时间最长的那个节点)移入到同步队列， 而移入到同步队列后才有机会使得等待线程被唤醒， 即从await方法中的LockSupport.park(this)方法中返回，从而才有机会使得调用await方法的线程成功退出。

![img](../../images/AQS/1620.jpeg)







```java
public final void signal() {
    if (!isHeldExclusively()) // 抽象方法，有子类实现，用于判断当前线程是否持有锁
        throw new IllegalMonitorStateException();  // 只有持有锁的线程才能操作唤醒
    Node first = firstWaiter;  // 获取等待队列的头结点
    if (first != null) 
        doSignal(first); // 执行唤醒操作
}
```



```java
private void doSignal(Node first) {
    do {
        // t1
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&  // t2
             (first = firstWaiter) != null); // t3
}
```



 transferForSignal ，将该 Node 加入 AQS 队列尾部；

**等到AQS被唤醒并拿锁**







### await与signal和signalAll的结合



await和signal和signalAll方法就像一个开关控制着线程A（等待方）和线程B（通知方）。 它们之间的关系可以用下面一个图来表现得更加贴切：

![img](../../images/AQS/1620-16611586524173.jpeg)





线程awaitThread先通过lock.lock()方法获取锁成功后调用了condition.await方法进入等待队列， 而另一个线程signalThread通过lock.lock()方法获取锁成功后调用了condition.signal或者signalAll方法， 使得线程awaitThread能够有机会移入到同步队列中， 当其他线程释放lock后使得线程awaitThread能够有机会获取lock， 从而使得线程awaitThread能够从await方法中退出，然后执行后续操作。 如果awaitThread获取lock失败会直接进入到同步队列。

































# ReentrantLock

比如在ReentrantLock中就是定义了一个抽象内部类Sync`，继承`AQS，然后定义了一个FairSync（公平锁类）和NonfairSync（非公平锁类）



## 特点

- 可中断
- 可以设置超时时间
- 可以设置公平锁或者非公平锁
- 支持多个条件变量
- 与synchronized一样，都支持可重入



如图在ReentrantLock类里面

![image-20220411215927960](../../images/AQS/image-20220411215927960.png)

**结构图**

![image-20220411215909135](../../images/AQS/image-20220411215909135.png)



**公平锁与非公平锁**

- **公平锁**：多个线程按照申请锁的顺序去获得锁，后申请锁的线程需要排队，等它之前的线程获得锁并释放后，它才能获得锁；
- **非公平锁**：线程获得锁的顺序于申请锁的顺序无关，申请锁的线程可以直接尝试获得锁，谁抢到就是谁的；





```java
public static final Lock lock = new ReentrantLock();//括号里面true是公平锁，false是非公平锁，默认是非公平锁
```

这个lock锁平时就是这样用的

```java
lock.lock();//加锁
//具体逻辑
lock.unlock();//解锁
//补充一点：这个lock锁的可重入锁，它的对手synchronized也是可重入锁
```

接下来我们就看看lock锁加锁和解锁是底层是怎样基于AQS的、



# 加锁(lock)

点进去这个lock()会进入syn.lock,接下来会让选择是公平锁还是非公平类里面的加锁

![image-20220411222057654](../../images/AQS/image-20220411222057654.png)



**加锁主要是acquire里面的两个方法**

## 公平锁与非公平锁(重点)

1. **非公平锁在调用 lock 后，首先就会调用 CAS 进行一次抢锁，如果这个时候恰巧锁没有被占用，那么直接就获取到锁返回了。**

   

2. **非公平锁在 第一次CAS 失败后，和公平锁一样都会进入到 tryAcquire 方法，在 tryAcquire 方法中，如果发现锁这个时候被释放了（state == 0）也就是无线程持有锁，非公平锁会直接 CAS 抢锁。**

2. 无论是公平锁还是非公平锁，如果此时有其他线程持有锁，公平锁和非公平锁都会进入队列

3. **但是如果在无线程持有锁的状态下（state = =0）情况下，公平锁会判断同步队列里面是否有线程在等待，如果有线程在队列里面等待则不去抢锁，然后进同步队列，没有线程才会去抢锁。而非公平锁无论同步队列里面有没有线程等待，都会直接去抢锁，没有抢到才会进入同步队列**

2. 如果state值不等于0，进入下一个逻辑，公平锁和非公平锁锁都会去判断的当前持有锁的线程是不是自己，来判断线程是否重入锁。









**总结：**

**公平锁和非公平锁就这前三点区别，如果lock和tryAcquire的这两次 CAS 都不成功，那么后面非公平锁和公平锁是一样的，都要进入到阻塞队列等待唤醒。**





**非公平锁效率能高一点，先去CAS地加一次锁，然后再acquire**



**FairSync**

```java
final void lock() {
    acquire(1);
}
```



**NonfairSync**

```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

**可以清晰的看到，非公平锁是先之间去CAS去改变state来抢锁。**



![image-20220410201311198](../../images/AQS/image-20220410201311198.png)

## acquire()

两者最后都会进入这个方法

```java
acquire(1);//这个参数为1很重要，与state有关，看下去就知道了
```

发现这个方法在AQS里面

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&//第一个方法
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))//第二个方法
        selfInterrupt();//自我中断，如果当前线程是中断唤醒的，可能拿到锁之后的逻辑需要用到中断标志位，而     acquireQueued里面的parkAndCheckInterrupt在由中断唤醒后会清除标志位。
}
```



**基于上面的acquire来看看里面的tryAcquire方法（这个方法在FairSync和NonfairSync里面）和acquireQueued方法**



这个tryAcquire方法在公平锁类和非公平类里面实现不一样，逻辑是尝试去取锁。



### ①tryAcquire（尝试加锁）

**公平锁里面的**

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();//获取当前进入这个方法的线程
    int c = getState();//获取当前AQS维护的state值
    if (c == 0) {//如果state是0，说明这个锁还没有其他线程持有，那就接着再判断
        if (!hasQueuedPredecessors() &&//公平锁里面先判断，同步队列里面是否有线程再等待，而非公平锁就不用判断
            compareAndSetState(0, acquires)) {//如果没有才能去CAS尝试改变state值来获取锁（如果CAS成功并且是第一次持有那么state变成1）。
            setExclusiveOwnerThread(current);//设置AQS里面当前获取锁的线程为current
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {//如果c != 1,但是当前持有锁的线程是自己，那么自己就可以重新获取锁（返回true）
        int nextc = c + acquires;//此时再重入的话要改变一下state值
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;//重入了锁，返回true
    }
    return false;
}
```

**非公平锁里面的**

```java
里面又调用了这个方法
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {//看，与公平锁对比，这个就不用判断同步阻塞队列里面是否有阻塞线程再等待
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```









### ②acquireQueued(addWaiter(Node.EXCLUSIVE)) （两个逻辑） 

记得这个acquire方法里面的逻辑，如果当前尝试锁失败，那就执行**acquireQueued(addWaiter(Node.EXCLUSIVE), arg)**

这个要先执行acquireQueued方法里面的**addWaiter(Node.EXCLUSIVE)**，这个方法就是将未获取锁的线程加入**同步队列队尾**，Node.EXCLUSIVE说明是独占。



#### addWaiter（加入队列（尾插））



点进这个addWaiter一探究竟

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);//为获取锁失败的线程创建应该Node结点
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail; 
    if (pred != null) { //判断如果当前队列不为null,执行这里面的逻辑
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    
    enq(node);//这个方法是队列为null才进入到这个方法，进行初始化，注意一下，可能有多个线程抢锁失败然后进入阻塞队列，而同时发现这个队列还没有开始真正建立，所以都进入到了这个方法里面，
    return node;
}
```



##### enq（）（初始化结点）

```java
private Node enq(final Node node) {
    for (;;) {//循环，线程会不断自旋（自旋的目的是要让每个进入到这个方法的线程入队）
        Node t = tail;//获取尾结点
        if (t == null) { // 说明队列还没有初始化，先初始化
            if (compareAndSetHead(new Node()))//CAS地创建一个结点，不能让多个线程创建了多个这样的结点，（因为这是初始化队列啊）
                tail = head;//头和尾指针指向这个被创建好的默认结点，（不存放线程值等一些其他值），主要是为了方便操作
        } else {//如果某个线程初始化好了队列
            
            node.prev = t;//当前线程的node结点前驱指向尾结点
            if (compareAndSetTail(t, node)) {//多个线程CAS地加入队列，一定要让进入这个方法的多个线程都要加入队列，因为进入这个方法的多个线程都是在没有初始化队列的时候进来的，就算线程们没有初始化成功，那也要加入初始化好的队列里面啊，因为进入这个方法的线程们都是抢锁失败而要被阻塞的线程们啊。这里为什么要CAS因为入队也在竞争
                t.next = node;
                return t;
            }
            
        }
    }
}
```



**注意这个addWaiter方法只是入队了，还没有真正的阻塞，阻塞还得看acquireQueued方法。**



#### acquireQueued（park阻塞线程）

阻塞线程加入队列后开始执行**acquireQueued**逻辑

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {//进入到这里的线程会自旋，很重要。可以结合后面的图看
            final Node p = node.predecessor();//获得当前线程组成的结点入队后的前驱结点
            if (p == head && tryAcquire(arg)) {
                //p == head判断这个结点是不是head.next所指向的结点（也就是判断是不是进入队列的第一个结点），如果是，那么再尝试一下去获取锁tryAcquire
                //如果成功获取锁，那么进入下面的逻辑
                setHead(node);//把head结点出队，并且当前结点作为head结点，但当前结点的pre和thread值已经被置null，相当于是新的head结点了
                p.next = null; // help GC GC会释放这个刚才被抛弃的前head结点
                failed = false;
                return interrupted;// 返回中断标志，acquire方法可以根据返回的中断标志，判断当前线程是否被中断
                
           //   这个if逻辑总结：再次尝试锁，如果能够获取到，节点出队，并且把head往后挪一个节点，新的头结点就是当前节点
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                
                parkAndCheckInterrupt() )// 直到被其他线程唤醒，或者被中断后返回，返回时将返回一个boolean值，                                                 // 表示这个线程是否被中断，若为true，则将执行下面一行代码，将中断标志置为true 
               	//补充：线程在被park()阻塞后，unpark()和中断（interrupt（））都可以唤醒线程											
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```



补充：

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);//当park没有参数的时候，中断唤醒之后再调用park不会被阻塞，而有参数的会再次被阻塞
    return Thread.interrupted();
}
```





##### shouldParkAfterFailedAcquire（设置前置结点为singal(-1)）



**并删除前置结点中处于CANCELLED状态的结点**





**这个方法主要是将新插入的结点的前置结点waitstatus设置为singal(-1)，表示后继节点的线程处于等待状态**



1. 如果当前线程的前驱节点状态为SINNAL，则表明当前线程需要被阻塞，调用unpark()方法唤醒，直接返回true，当前线程阻塞

2. 如果当前线程的前驱节点状态为CANCELLED（ws > 0），则表明该线程的前驱节点已经等待超时或者被中断了，则需要从CLH队列中将该前驱节点删除掉，直到回溯到前驱节点状态 <= 0 ，返回false

3. 如果前驱节点非SINNAL，非CANCELLED，则通过CAS的方式将其前驱节点设置为SINNAL，返回false

   ![image-20220412000026233](../../images/AQS/image-20220412000026233.png)

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)

        return true;
    if (ws > 0) {

        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {

        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

​		



##### parkAndCheckInterrupt



调用parkAndCheckInterrupt方法，当前线程在这个方法中等待，

 这个方法就是阻塞线程，线程到这就阻塞了也自旋不了。等待被唤醒。

这个方法又调用了LockSupport.park(this)，

```java
private final boolean parkAndCheckInterrupt() {
            //调用park()使线程进入waiting状态
    LockSupport.park(this);
     //如果被唤醒，查看自己是不是被中断的
    return Thread.interrupted();//返回当前线程是否阻塞
}
```

## lock总结图

![image-20220412002620445](../../images/AQS/image-20220412002620445.png)

加锁的逻辑就是上面这些了，接下来看看解锁的逻辑。

## lock加锁逻辑（重点）





下面为lock加锁的主要逻辑





非公平锁相比与公平锁先CAS地去加一次锁

然后公平锁和非公平锁都进入acquire逻辑

**![image-20220822173545804](../../images/AQS/image-20220822173545804.png)**



**tryAcquire**先尝试加锁，公平锁与非公平锁略微不同，具体不同看上面逻辑



**addWaiter**将抢锁失败的线程加入队列



**acquireQueued**阻塞线程 {



​	当前线程是否排在队首，如果是的话还会去tryAcquire

}	















# 解锁(unlock)

lock.unlock()会调用release方法，在ReentrantLock里面



**一般解锁unlock在finally代码块里面，避免异常。**





## release（解锁主逻辑）

```java
public final boolean release(int arg) {//arg是1
    if (tryRelease(arg)) {//// 调用tryRelease尝试修改state释放锁，若成功，将返回true，否则false
        Node h = head;//拿到队列head结点
        // 头结点不为空并且头结点的waitStatus不是初始化节点情况，解除线程挂起状态
        if (h != null && h.waitStatus != 0)// != 0 说明刚才在阻塞方法里面第一个阻塞结点的前驱结点是-1,也就是那个head结点是-1，head.next是要等待被唤醒的线程，这个h.waitStatus != 0就是唤醒做判断的，之前那个为什么要循环两次改waitstatues值，就是这块来做判断的，确保唤醒的是head,next对应的结点指向的线程
            
        // 若head不是null，且waitStatus不为0，表示它是一个装有线程的正常节点，
        // 在之前提到的addWaiter方法中，若同步队列为空，则会创建一个默认的节点放入head
        // 这个默认的节点不包含线程，它的waitStatus就是0，所以不能释放锁
            unparkSuccessor(h);//唤醒的方法
        return true;
    }
    return false;
}
```

> h == null Head还没初始化。初始情况下，head == null，第一个节点入队，Head会被初始化一个虚拟节点。所以说，这里如果还没来得及入队，就会出现head == null 的情况。
>
> h != null && waitStatus == 0 表明后继节点对应的线程仍在运行中，不需要唤醒。
>
> h != null && waitStatus < 0 表明后继节点可能被阻塞了，需要唤醒。



**双向链表中，第一个节点为虚节点，其实并不存储任何信息，只是占位。真正的第一个有数据的节点，是在第二个节点开始的。**

## tryRelease（尝试解锁）

看看尝试释放锁的逻辑

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;//当前状态减1
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {//说明加上重入加的state值已经减完了
        free = true;
        setExclusiveOwnerThread(null);//AQS里面持有锁的线程置null
    }
    setState(c);//修改state值，不一定是置0，说不定是释放的重入之后的锁而减的state值。
    return free;
}
```







## unparkSuccessor（唤醒队列结点）     

**重要：从尾节点开始往前遍历，寻找离头节点最近的等待状态正常的节点**



```java
private void unparkSuccessor(Node node) {
		// 获取头结点waitStatus
    int ws = node.waitStatus;//把这个head结点状态拿出来判断
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);//正常情况下一开始肯定ws是-1，重新改回0
// 获取当前节点的下一个节点
    Node s = node.next;//head.next指向的结点的thread值就是要被唤醒的下次你
    // 如果下个节点是null或者下个节点被cancelled，就找到队列最开始的非cancelled的节点
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 就从尾部节点开始找，到队首，找到队列第一个waitStatus<0的节点。
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    
    if (s != null)
        LockSupport.unpark(s.thread);//唤醒，之前那个阻塞也是调用这个类的方法
}
```







这个head的waitStatus之前由0改-1是为了阻塞线程，而又由-1改为0是为了唤醒线程，为什么？为什么？



一定要记得，这个要被唤醒等待线程还正在之前acquireQueued那里阻塞着，如果被唤醒了，就会自旋重新进入那个for循环逻辑，自旋的话，肯定第一个if要去尝试获取锁，但是如果在公平锁里面就会执行成功，会获取锁然后出队，但是在非公平锁里面可能尝试获取锁失败，然后又会执行shouldParkAfterFailedAcquire修改head.waitStatus为-1，然后又会调用parkAndCheckInterrupt阻塞住了





### 为什么要从后往前找第一个非Cancelled的节点呢？（重点）

**唤醒结点的逻辑**：**从尾节点开始往前遍历，寻找离头节点最近的等待状态正常的节点**





**可能是队列尾部加入结点得的操作不是原子操作，如果此时从头到尾遍历，可能会发生断链，不能遍历到所有的结点。**





















**之前的addWaiter方法：**

```java
private Node addWaiter(Node mode) {
	Node node = new Node(Thread.currentThread(), mode);
	// Try the fast path of enq; backup to full enq on failure
	Node pred = tail;
	if (pred != null) {
		node.prev = pred;
		if (compareAndSetTail(pred, node)) {
			pred.next = node;
			return node;
		}
	}
	enq(node);
	return node;
}
```

**原因**

- 我们从这里可以看到，节点入队并不是原子操作，也就是说，node.prev = pred; compareAndSetTail(pred, node) 这两个地方可以看作Tail入队的原子操作，但是此时pred.next = node;还没执行，如果这个时候执行了unparkSuccessor方法，就没办法从前往后找了，所以需要从后往前找。
- 还有一点原因，在产生CANCELLED状态节点的时候，先断开的是Next指针，Prev指针并未断开，因此也是必须要从后往前遍历才能够遍历完全部的Node。

> 关于上面第一个原因，我自己的理解是，因为之前的尾插进新节点的逻辑是，先tail.next = node,然后CAS操作当前结点node为尾结点，最后之前的tail结点.next = node,这几个操作是非原子性，如果最后一步tail.next = node还没执行的话，就执行unparkSuccessor（唤醒队列结点）方法，而又采取从前往后遍历（找waitstatus<=0的结点，最后找的是最靠近head的有效结点），是遍历不到当前插入的新结点，**不能遍历全部的结点**。





**综上所述，如果是从前往后找，由于极端情况下入队的非原子操作和CANCELLED节点产生过程中断开Next指针的操作，可能会导致无法遍历所有的节点。所以，唤醒对应的线程后，对应的线程就会继续往下执行。继续执行acquireQueued方法以后，中断如何处理？**









![image-20220919202939673](../../images/AQS/image-20220919202939673.png)



































## 中断恢复后的执行流程

也就是唤醒队列中阻塞线程后的逻辑



唤醒后，会执行return Thread.interrupted();，这个函数返回的是当前执行线程的中断状态，并清除。



```java
private final boolean parkAndCheckInterrupt() {
	LockSupport.park(this);
	return Thread.interrupted();
}
```







**再回到acquireQueued代码，当parkAndCheckInterrupt返回True或者False的时候，interrupted的值不同，但都会执行下次循环。如果这个时候获取锁成功，就会把当前interrupted返回。**







```java
final boolean acquireQueued(final Node node, int arg) {
	boolean failed = true;
	try {
		boolean interrupted = false;
		for (;;) {
			final Node p = node.predecessor();
			if (p == head && tryAcquire(arg)) {
				setHead(node);
				p.next = null; // help GC
				failed = false;
				return interrupted;
			}
			if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
				interrupted = true;
			}
	} finally {
		if (failed)
			cancelAcquire(node);
	}
}
```



如果acquireQueued为True，就会执行selfInterrupt方法。



```java
static void selfInterrupt() {
	Thread.currentThread().interrupt();
}
```

**该方法其实是为了中断线程。但为什么获取了锁以后还要中断线程呢？这部分属于Java提供的协作式中断知识内容，感兴趣同学可以查阅一下。这里简单介绍一下：**

1. 当中断线程被唤醒时，并不知道被唤醒的原因，可能是当前线程在等待中被中断，也可能是释放了锁以后被唤醒。因此我们通过Thread.interrupted()方法检查中断标记（该方法返回了当前线程的中断状态，并将当前线程的中断标识设置为False），并记录下来，如果发现该线程被中断过，就再中断一次。
2. 线程在等待资源的过程中被唤醒，唤醒后还是会不断地去尝试获取锁，直到抢到锁为止。也就是说，在整个流程中，并不响应中断，只是记录中断记录。最后抢到锁返回了，那么如果被中断过的话，就需要补充一次中断。

这里的处理方式主要是运用线程池中基本运作单元Worder中的runWorker，通过Thread.interrupted()进行额外的判断处理，感兴趣的同学可以看下ThreadPoolExecutor源码。











## 唤醒之后线程还是还是在lock加锁逻辑，去尝试获取锁







































## unlock总结图

![image-20220412002701663](../../images/AQS/image-20220412002701663.png)





Park阻塞线程唤醒有两种方式：

1.中断

2.unlock() -> release()

## unlock逻辑



释放锁资源后，还得找到第一个满足条件的线程唤醒





















# lockInterruptibly 

所谓的**中断锁指的是锁在执行时可被中断，也就是在执行时可以接收 interrupt 的通知，从而中断锁执行**。 





使用中断锁



**通过显示锁 Lock 的 lockInterruptibly 方法来完成，它和 lock 方法作用类似，但是 lockInterruptibly 可以优先接收到中断的通知，而 lock 方法只能“死等”锁资源的释放**



```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class InterruptiblyExample {
    public static void main(String[] args) throws InterruptedException {
        Lock lock = new ReentrantLock();

        // 创建线程 1
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    // 加锁操作
                    lock.lock();
                    System.out.println("线程 1:获取到锁.");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                // 线程 1 未释放锁
            }
        });
        t1.start();

        // 创建线程 2
        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                // 先休眠 0.5s，让线程 1 先执行
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                // 获取锁
                try {
                    System.out.println("线程 2:尝试获取锁.");
                    lock.lockInterruptibly(); // 可中断锁
                    System.out.println("线程 2:获取锁成功.");
                } catch (InterruptedException e) {
                    System.out.println("线程 2:执行已被中断.");
                }
            }
        });
        t2.start();a

        // 等待 2s 后,终止线程 2
        Thread.sleep(2000);
        if (t2.isAlive()) { // 线程 2 还在执行
            System.out.println("执行线程的中断.");
            t2.interrupt();
        } else {
            System.out.println("线程 2:执行完成.");
        }
    }
}
```

![img](../../images/AQS/v2-5889cb4741cb91b5a328c8cd8df60c4f_720w.jpg)



**从上述结果可以看出，当我们使用了 lockInterruptibly 方法就可以在一段时间之后，判断它是否还在阻塞等待，如果结果为真，就可以直接将他中断，如上图效果所示。** 



但当我们尝试将 lockInterruptibly 方法换成 lock 方法之后（其他代码都不变），执行的结果就完全不一样了，实现代码如下：

**从上图可以看出，当使用 lock 方法时，即使调用了 interrupt 方法依然不能将线程 2 进行中断。**





## 中断锁原理



除了lock锁还有一种加锁方式，接下来就看看这个lockInterruptibly加锁有什么与lock不同点把。

```java
lock.lockInterruptibly();
```



这个方法会调用acquireInterruptibly方法

```java
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))//如果加锁失败的话会进入下面这个方法
        doAcquireInterruptibly(arg);
}
```





```java
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);//入队
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();//不同点主要在这，如果是由中断唤醒的线程，会抛出异常
        }
    } finally {
        if (failed)
            cancelAcquire(node);//在抛出异常前执行，将这个线程的waitStatus修改为1
                                //标注当前结点的生命状态为canceld，并且剔除这个结点
    }
}
```



Node结点的**waitStatus**状态：

- SIGNAL = -1 // 可被唤醒
- CANCELLED = 1 //代表出现异常，中断引起的，需要废弃结束
- CONDITION = -2 //条件等待
- PROPAGATE = -3 //传播
- 0 - 初始状态Init状态 //



cancelAcquire方法就是将当前线程Node结点的waitStatus状态改为1，并且队列里面剔除这个结点。



下面是这个方法的主要逻辑

```java
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;

    node.thread = null;

    // Skip cancelled predecessors
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    // predNext is the apparent node to unsplice. CASes below will
    // fail if not, in which case, we lost race vs another cancel
    // or signal, so no further action is necessary.
    Node predNext = pred.next;

    // Can use unconditional write instead of CAS here.
    // After this atomic step, other Nodes can skip past us.
    // Before, we are free of interference from other threads.
    node.waitStatus = Node.CANCELLED;

    // If we are the tail, remove ourselves.
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```





# AQS总结图

![图](../../images/AQS/%E5%9B%BE.png)





# 面试篇

## 为什么 AQS 需要一个虚拟 head 节点



每个节点都必须设置前置节点的 ws 状态为 SIGNAL（-1），因为每个节点在休眠前，都需要将前置节点的 ws 设置成 SIGNAL。否则自己永远无法被唤醒，所以必须要一个前置节点，而这个前置节点，实际上就是当前持有锁的节点。
由于第一个节点他是没有前置节点的，就创建一个假的。
总结下来就是：每个节点都需要设置前置节点的 ws 状态（这个状态为是为了保证数据一致性），而第一个节点是没有前置节点的，所以需要创建一个虚拟节点。







## 如果队列中有多个线程存活，有可能会唤醒第一个线程之后的线程吗（共享模式）

应该是可以的



**在共享模式下**

当队列中的等待线程被唤醒以后就重新尝试获取锁资源，如果成功则**唤醒后面还在等待的共享节点并把该唤醒事件传递下去，即会依次唤醒在该节点后面的所有共享节点**，然后进入临界区，否则继续挂起等待。





