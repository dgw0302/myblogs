---
title: synchronized
categories:
  - 多线程与并发
tags:
  - synchronized
abbrlink: 46153
date: 2022-04-10 19:54:20
---



**必看敖丙文章**：https://mp.weixin.qq.com/s/2ka1cDTRyjsAGk_-ii4ngw

# 前提

**如何解决线程并发安全问题？**

实际上，所有的并发模式在解决线程安全问题时，采用的方案都是**序列化访问临界资源**。即在同一时刻，只能有一个线程访问临界资源，也称作**同步互斥访问**。

**Java 中，提供了两种方式来实现同步互斥访问：synchronized**（内置锁或者叫隐式锁）和**Lock**（显示锁）

**同步器的本质就是加锁**

加锁目的：**序列化访问临界资源**，即同一时刻只能有一个线程访问临界资源(**同步互斥访问**)

不过有一点需要区别的是：当多个线程执行一个方法时，该方法内部的局部变量并不是临界资源，因为这些局部变量是在每个线程的私有栈中，因此不具有共享性，不会导致线程安全问题。



# synchronized

# 三大特性简介

## 保障原子性

保证拿到锁的线程执行完之后才，才能让其他线程执行，保障了拿到锁的线程在执行时候不会受到其他线程的打断。

## 保障可见性

synchronized是可以保证可见性的



**原因**：JMM中关于synchronized有如下规定，线程加锁时，必须清空工作内存中共享变量的值，从而使用共享变量时需要从主内存重新读取；线程在解锁时，需要把工作内存中最新的共享变量的值写入到主存，以此来保证共享变量的可见性





“synchronized在修改了本地内存中的变量后，解锁前会将本地内存修改的内容刷新到主内存中，确保了共享变量的值是最新的，也就保证了可见性

## 不能保障有序性

## 深入

深入synchroized得先🙅‍三个知识：

- CAS(上面已经介绍)
- 用户态与内核态(下面讲)
- markword（下面讲）



`synchronized`是能够保证有序性的。根据as-if-serial语义，无论编译器和处理器怎么优化或指令重排，单线程下的运行结果一定是正确的。而`synchronized`保证了单线程独占CPU，也就保证了有序性。







# synchronized底层原理

**synchronized内置锁是一种对象锁(锁的是对象而非引用)，作用粒度是对象，可以用来实现对临界资源的同步互斥访问，是可重入的。**

**加锁的方式：**

1、同步实例方法，锁是当前实例对象

2、同步类方法，锁是当前类对象

3、同步代码块，锁是括号里面的对象

**synchronized底层原理**

**synchronized是基于JVM**内置锁实现，通过内部对象**Monitor**(监视器锁)实现，基于进入与退出**Monitor**对象实现方法与代码块同步，监视器锁的实现依赖底层操作系统的**Mutex lock**（互斥锁）实现，它是一个重量级锁性能较低。当然，**JVM内置锁在1.5之后版本做了重大的优化，**如锁粗化（Lock Coarsening）、锁消除（Lock Elimination）、轻量级锁（Lightweight Locking）、偏向锁（Biased Locking）、适应性自旋（Adaptive Spinning）等技术来减少锁操作的开销，，内置锁的并发性能已经基本与Lock持平。

synchronized关键字被编译成字节码后会被翻译成monitorenter 和 monitorexit 两条指令分别在同步块逻辑代码的起始位置与结束位置。

​    ![0](../../images/synchroized/2512)

每个同步对象都有一个自己的Monitor(监视器锁)，加锁过程如下图所示：

​    ![0](../../images/synchroized/2528)

**Monitor监视器锁**

​    **任何一个对象都有一个Monitor与之关联，当且一个Monitor被持有后，它将处于锁定状态**。Synchronized在JVM里的实现都是 **基于进入和退出Monitor对象来实现方法同步和代码块同步**，虽然具体实现细节不一样，但是都可以通过成对的MonitorEnter和MonitorExit指令来实现。

- **monitorenter**：每个对象都是一个监视器锁（monitor）。当monitor被占用时就会处于锁定状态，线程执行monitorenter指令时尝试获取monitor的所有权，过程如下：

- 1. **如果monitor的进入数为0**，则该线程进入monitor，然后将进入数设置为1，该线程即为monitor的所有者；
  2. **如果线程已经占有该monitor**，只是重新进入，则进入monitor的进入数加1；
  3. **如果其他线程已经占用了monitor**，则该线程进入阻塞状态，直到monitor的进入数为0，再重新尝试获取monitor的所有权；

- **monitorexit**：执行monitorexit的线程必须是objectref所对应的monitor的所有者。**指令执行时，monitor的进入数减1，如果减1后进入数为0，那线程退出monitor，不再是这个monitor的所有者**。其他被这个monitor阻塞的线程可以尝试去获取这个 monitor 的所有权。

**monitorexit，指令出现了两次，第1次为同步正常退出释放锁；第2次为发生异步退出释放锁**；

通过上面两段描述，我们应该能很清楚的看出Synchronized的实现原理，**Synchronized的语义底层是通过一个monitor的对象来完成，其实wait/notify等方法也依赖于monitor对象**，这就是为什么只有在同步的块或者方法中才能调用wait/notify等方法，**否则会抛出java.lang.IllegalMonitorStateException的异常的原因**。

看一个同步方法：

​      

```java
package it.yg.juc.sync;
public class SynchronizedMethod {
    public synchronized void method() {
        System.out.println("Hello World!");
    }
}           
```

反编译结果：

​    ![0](../../images/synchroized/14066)

从编译的结果来看，方法的同步并没有通过指令 **monitorenter** 和 **monitorexit** 来完成（理论上其实也可以通过这两条指令来实现），不过相对于普通方法，其常量池中多了 **ACC_SYNCHRONIZED** 标示符。**JVM就是根据该标示符来实现方法的同步的**：

当方法调用时，**调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置**，如果设置了，**执行线程将先获取monitor**，获取成功之后才能执行方法体，**方法执行完后再释放monitor**。在方法执行期间，其他任何线程都无法再获得同一个monitor对象。

两种同步方式本质上没有区别，只是方法的同步是一种隐式的方式来实现，无需通过字节码来完成。**两个指令的执行是JVM通过调用操作系统的互斥原语mutex来实现，被阻塞的线程会被挂起、等待重新调度**，会导致“用户态和内核态”两个态之间来回切换，对性能有较大影响。

**什么是monitor？**

可以把它理解为 **一个同步工具**，也可以描述为 **一种同步机制**，它通常被 **描述为一个对象**。与一切皆对象一样，所有的Java对象是天生的Monitor，每一个Java对象都有成为Monitor的潜质，**因为在Java的设计中 ，每一个Java对象自打娘胎里出来就带了一把看不见的锁，它叫做内部锁或者Monitor锁**。**也就是通常说Synchronized的对象锁，MarkWord锁标识位为10，其中指针指向的是Monitor对象的起始地址**。在Java虚拟机（HotSpot）中，**Monitor是由ObjectMonitor实现的**，其主要数据结构如下（位于HotSpot虚拟机源码ObjectMonitor.hpp文件，C++实现的）：

```c++
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; // 记录个数
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;
    _WaitSet      = NULL; // 处于wait状态的线程，会被加入到_WaitSet
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ; // 处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }

```



ObjectMonitor中有两个队列，**_WaitSet 和 _EntryList**，用来保存ObjectWaiter对象列表（ **每个等待锁的线程都会被封装成ObjectWaiter对象** ），**_owner指向持有ObjectMonitor对象的线程**，当多个线程同时访问一段同步代码时：

1. 首先会进入 _EntryList 集合，**当线程获取到对象的monitor后，进入 _Owner区域并把monitor中的owner变量设置为当前线程，同时monitor中的计数器count加1**；
2. 若线程调用 wait() 方法，**将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 WaitSet集合中等待被唤醒**；
3. 若当前线程执行完毕，**也将释放monitor（锁）并复位count的值，以便其他线程进入获取monitor(锁)**；

同时，**Monitor对象存在于每个Java对象的对象头Mark Word中（存储的指针的指向），Synchronized锁便是通过这种方式获取锁的**，也是为什么Java中任意对象可以作为锁的原因，**同时notify/notifyAll/wait等方法会使用到Monitor锁对象，所以必须在同步代码块中使用**。监视器Monitor有两种同步方式：**互斥与协作**。多线程环境下线程之间如果需要共享数据，需要解决互斥访问数据的问题，**监视器可以确保监视器上的数据在同一时刻只会有一个线程在访问**。

那么有个问题来了，我们知道synchronized加锁加在对象上，对象是如何记录锁状态的呢？答案是锁状态是被记录在每个对象的对象头（Mark Word）中，下面我们一起认识一下**对象的内存布局**



# 补知识

## 用户空间与内核空间

操作系统有用户空间与内核空间两个概念，目的也是为了做到程序运行安全隔离与稳定

用户程序运行在用户方式下，而系统调用运行在内核方式下。在这两种方式下所用的堆栈不一样：用户方式下用的是一般的堆栈(用户空间的堆栈)，而内核方式下用的是固定大小的堆栈（内核空间的对战，一般为一个内存页的大小），即每个进程与线程其实有两个堆栈，分别运行与用户态与内核态。

操作系统（OS）将指令分为级别访问，用户态访问和内核态访问，内核态可以访问所有指令，用户态只能访问用户能访问的指令。

CPU有4个运行级别，分别为： ring0、ring1、ring2、ring3，而Linux与Windows只用到了2个级别:ring0、ring3。操作系统内部内部程序指令通常运行在ring0级别，操作系统以外的第三方程序运行在ring3级别（咋们用户就在ring3级别），第三方程序如果要调用操作系统内部函数功能，由于运行安全级别不够，必须切换CPU运行状态，从ring3切换到ring0，然后执行系统函数，说到这里相信同学们明白为什么JVM创建线程，线程阻塞唤醒是重型操作了，因为CPU要切换运行状态。





# 对象的内存布局

问题：

- **这个实例对象是以怎样的形态存在内存中的?**
- **一个Object对象在内存中占用多大?**
- **对象中的属性是如何在内存中分配的?**



## 三部分

HotSpot中，Java的**对象内存模型**分为三部分，分别为**对象头、实例数据和对齐填充**。

而对象头中分为两部分，

- 一部分是“Mark Word”（存储对象自身的运行时数据，数据长度为32b it或64bit，可复用，如哈希码(HashCode)、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等）；

- 另一部分是指向它的类的元数据指针。

  

补充小知识：这个我们平时说的由实例对象找到它对应的class对象，是怎样找到，就是通过它对象头里面的指向它的类的元数据指针找的。



![img](../../images/synchroized/2557-16495704082374)

对象的锁状态记录在markword里面



因为synchronized是Java对象的内置锁，所以其优化策略（即偏向锁等）的信息都包含在Mark Word中。



## **Mark Word的结构**

![image-20220211153557138](../../images/synchroized/image-20220211153557138.png)



其中最重要的是“锁标志位”和“是否偏向锁”，锁标志位代表了当前对象内置锁的状态，不同的锁状态下Mark Word存储的信息是不同的，因此称为可复用。



![img](../../images/synchroized/1162587-20200917170455322-1670500196.png)











# 锁重入

synchroized是可重入锁

什么是可重入锁：假如synchroized修饰了A方法，那么线程在执行A方法前就要拿到当前锁，这个A方法里

面又调用了B方法，B方法又被同一把锁修饰（锁的是相同的对象资源），那么A方法调用B方法并执行B方法那么就要重新获得这个锁。

重入次数必须记录，因为要解锁几次必须得对应。

# 锁膨胀过程

B：https://mp.weixin.qq.com/s/2ka1cDTRyjsAGk_-ii4ngw



jdk1.6之前，使用synchroized就只有一种状态，重量级锁，是在操作系统底层对monitor加锁，并且还有阻塞队列，线程调度器在当前那所线程执行完同步代码块后会调用阻塞队列里面的某个线程，很耗资源。（线程切换）

在1.6之后，对synchroized内部的锁机制进行了改善，引用了几种锁状态

![image-20220211153504253](../../images/synchroized/image-20220211153504253.png)

![image-20220212001256203](../../images/synchroized/image-20220212001256203.png)

new -> 偏向锁 -> 轻量级锁（无锁，自旋锁，自适应自旋）-> 重量级锁

## **偏向锁**

 当一个线程访问同步代码块并获取锁时，**会在对象头和栈帧存储锁偏向的线程id**，以后改线程在进入和退出同步代码块时不需要进行CAS操作来加锁和是否锁，只需要简单测试一下对象头的MARk Word里是否存储着指向当前线程的偏向锁。如果测试成功，表示线程已经获得了锁，如果失败，则需要再测试一下Mark Word中偏向锁的标志是否设置成了1（表示当前是偏向锁）：如果没有设置，则使用CAS竞争锁；如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程。

**偏向锁撤销**



偏向锁使用了一种等到竞争出现才释放锁的机制，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁。偏向锁的撤销需要等待拥有偏向锁的线程到达全局安全点（在这个时间点上没有字节码正在执行），会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着，如果线程不处于活动状态，则将锁的对象的对象头设置成无锁状态，如果线程仍然活着，拥有偏向锁的栈会被执行**(判断是否需要持有锁)，遍历偏向对象的锁记录，查看使用情况，如果还需要持有偏向锁，则偏向锁升级为轻量级锁**，如果不需要持有偏向锁了，则将锁对象恢复成无锁状态，最后唤醒暂停的线程。



## **轻量级锁**

倘若偏向锁失败，虚拟机并不会立即升级为重量级锁，它还会尝试使用一种称为轻量级锁的优化手段(1.6之后加入的)，此时Mark Word 的结构也变为轻量级锁的结构。轻量级锁能够提升程序性能的依据是“对绝大部分的锁，在整个同步周期内都不存在竞争”，注意这是经验数据。需要了解的是，轻量级锁所适应的场景是线程交替执行同步块的场合，如果存在同一时间访问同一锁的场合，就会导致轻量级锁膨胀为重量级锁。



关于轻量级锁加锁过程下面这个总结的比较好



> **轻量级解锁过程总结：**
>
> - 在代码即将进入同步块的时候，如果此同步对象没有被锁定（锁状态位是01），如果设置了偏向模式，检查偏向的线程是否时当前线程，如果不是，难么将膨胀为轻量级锁。**虚拟机在当前线程的栈帧中创建lockRecord， 用于存储对象MarkWord的拷贝（加锁成功后存储markword）和对象的引用地址（用于锁住之后完成对象的访问定位）。**
>
> 
>
> - **创建完LockRecord之后，虚拟机将使用CAS操作尝试把锁对象的Mark Word更新为指向当前线程中LockRecord的指针。**这里CAS的比较方法是：锁标志位是否为01，如果是则更新为LockRecord地址并将标志位置位00。
>
> 
>
> -  如果更新成功,此时LockRecord中存放了对象的原来的markword信息，同时将对象的markword锁标志位置为00，而对象的markword则存放了持有锁的线程的LockRecord地址，如果更新失败，则表示该对象的锁已经被持有了，持有锁的线程可能是他自己，也可能是其他线程。然后虚拟机先检查对象的MarkWord是否指向当前线程的栈帧，如果是，说明当前线程已经拥有了这个对象的锁，那直接进入同步块继续执行就可以了；否则就说明这个锁对象已经被其他线程抢占了，当前线程开始自旋重试。这里自旋重试次数可以是0，也就意味着发生竞争就会膨胀到重量级锁。当然也可以是某个数，取决于jdk设计者如何考量。所以有的文章说会自旋，有的不会自旋，无非是次数是否为0的差别。
>
> 
>
> 
>
> - 为什么更新失败后仍要检查对象MarkWord是否指向当前栈帧呢？原因是锁的重入。CAS更新失败有两种可能，1.它自己已经持有了该对象的锁，现在要重入。 2.其他线程持有了对象的锁。若是当前线程CAS更新了MarkWord，那么当前线程再次想要持有对象的锁时，它应该要能重入。锁重入的时候，又创建了新的LockRecord，但由于CAS更新失败，它内部并没有对象原来MarkWord的拷贝。







## **重量级锁**

**视器锁本质又是依赖于底层的操作系统的Mutex Lock（互斥锁）来实现的。而操作系统实现线程之间的切换需要从用户态转换到核心态，这个成本非常高，状态之间的转换需要相对比较长的时间，这就是为什么Synchronized效率低的原因。**



其实本质是底层对monitor对象加锁



**如果在自旋一定次数后线程获得锁，那么轻量级锁将会升级成重量级锁**

1.6之前，有线程超过十次自旋，或者自选线程数超过CPU核数的一半,就会有轻量级锁到重量级锁



**如果使用 synchronized 给对象上锁（重量级）之后，该对象头的Mark Word 中就被设置指向 Monitor 对象的指针**



**重量级锁有等待队列，所有拿不到锁的进入等待队列，不需要消耗CPU资源。**





所有线程的竞争转变成了monitor对象的竞争

monitor对象有维护一个队列









**问题：**

**为什么有自旋锁还需要重量级锁**，毕竟自旋锁只需要再用户态，不需要转换消耗时间，因为同步代码块执行时间如果过长，其他线程不断自旋，或者自旋线程多，这是消耗CPU资源的，不划算的，所有会生成重量级锁。

这一篇写的很好



# 







**是不是偏向锁一定比自旋锁效率高**？

不一定，在明确知道会有多线程竞争的情况下，偏向锁会涉及锁撤销，这时候直接使用轻量级锁（自旋锁）

jvm启动时候，会有很多线程竞争，默认不打开偏向锁，等一段时间再打开。



**轻量级锁一定比重量级锁性能高吗**

不一定，大量线程加锁的话，大量线程在自旋

锁升级过程，下面这篇博客比较好。

https://www.cnblogs.com/myseries/p/12213997.html





# synchroized锁原理一图总结

![synchronized锁实现与升级过程](../../images/synchroized/synchronized%E9%94%81%E5%AE%9E%E7%8E%B0%E4%B8%8E%E5%8D%87%E7%BA%A7%E8%BF%87%E7%A8%8B.png)



# 锁膨胀升级一图总结

![JVM锁的膨胀升级](../../images/synchroized/JVM%E9%94%81%E7%9A%84%E8%86%A8%E8%83%80%E5%8D%87%E7%BA%A7.jpg)



# Monitor重量级锁













## synchronized线程什么时候释放锁

1、当前线程的同步方法、代码块执行结束的时候释放

2、当前线程在同步方法、同步代码块中遇到break 、 return 终于该代码块或者方法的时候释放。

3、出现未处理的error或者exception导致异常结束的时候释放

4、程序执行了 同步对象 wait 方法 ，当前线程暂停，释放锁



##  Monitor对象

我们经常说synchronized关键字获得的是一个对象锁，那这个对象锁到底是什么？

每一个对象的对象头会关联一个Monitor对象，这个Monitor对象的实现底层是用C++写的，对应在虚拟机里的ObjectMonitor.hpp文件中。

Monitor对象由以下3部分组成：

（1）**EntryList队列**

当多个线程同时访问一个Monitor对象时，**这些线程会先被放进EntryList队列，此时这些线程处于Blocked状态；**

（2）**Owner**

当一个线程获取到了这个Monitor对象时，Owner会指向这个线程，当线程释放掉了Monitor对象时，Owner会置为null；

（3）**WaitSet队列**

**当线程调用wait方法时，当前线程会释放对象锁，同时该线程进入WaitSet队列。**

Monitor对象还有一个计数器count的概念，这个count是属于Monitor对象的，而不属于某个获得了Monitor对象的线程，当Monitor对象被某个线程获取时，++count，当Monitor对象被某个线程释放时，--count。









## monitor加锁（官方）



https://tech.youzan.com/javasuo-yu-xian-cheng-de-na-xie-shi/



> 这个尝试竞争锁失败是指CAS monitor对象里面的owner属性
>
> owner：指向当前获得锁的线程

![image-20220903160155296](../../images/synchroized/image-20220903160155296.png)





**PS:线程竞争锁时候会先CAS替换owner来竞争锁，没有竞争成功才会被加入cxq队列。当前拥有锁的线程被释放后会唤醒entrylist里面的线程跟同时到来的线程一起竞争owner（这也是非公平的一种体现，对已经进入队列的线程是不公平的）。**





### 核心属性介绍

> 1. owner 持有锁的线程
> 2. recursions 线程的重入次数
> 3. waitSet 调用wait方法后线程被放入的队列
> 4. cxq 请求锁的线程首先被放入的队列
> 5. EntryList cxq队列中有资格能获取到锁的线程被移动到的队列（策略迁移）





**核心逻辑**

![image.png](../../images/synchroized/388acdf1038149a396d97a4e5b45ce79tplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp)









**Cxq 竞争队列**

每次新加入 Node 会在 Cxq 的队头进行，通过 CAS 改 变第一个节点的指针为新增节点，同时设置新增节点的 next 指向后续节点;从 Cxq 取得元素时，会从队尾获取。因为只有 Owner 线程才能从队尾取元素，也即线程出列操作无争用，当然也就避免了 CAS 的 ABA 问题。线程进入 Cxq 前，抢锁线程会先尝试通过 CAS 自旋获取锁，如果获取不到，才就进入 Cxq 队列，这明显对于已经进入 Cxq 队列的线程是不公平的。所以，synchronized 同步块所使用的重量级锁是不公平锁。

**EntryList**

EntryList 与 Cxq 在逻辑上都属于等待队列。Cxq 会被线程并发访问，为了降低对 Cxq 队尾的 争用，而建立 EntryList。在 Owner 线程释放锁时，JVM 会从 Cxq 中迁移线程到 EntryList，并会 指定 EntryList 中的某个线程(一般为 Head)为 OnDeck Thread(Ready Thread)。EntryList 中的线程，作为候选竞争线程而存在。



**OnDeck Thread 与 Owner Thread**

JVM 不直接把锁传递给 Owner Thread，而是把锁竞争的权利交给 OnDeck Thread，OnDeck 需要重新竞争锁。这样虽然牺牲了一些公平性，但是能极大的提升系统的吞吐量，在 JVM 中，也把这种选择行为称之为“竞争切换”。在 OnDeck Thread 成为 Owner 的过程中，还有一个不公平的事情，就是后来的新抢锁线程可能直接通过 CAS 自旋成为 Owner 而抢到锁

**WaitSet**

Owner 线程被 Object.wait()方法阻塞，则转移到 WaitSet 队列中，直到某个时刻通过 Object.notify()或者 Object.notifyAll()唤醒，在线程会重新进去 EntryList 中。

















































# synchoronized公平吗

![image-20220820222208489](../../images/synchroized/image-20220820222208489.png)



当synchronized锁升级为重量级锁后，是不公平的









 

**公平锁与非公平锁**

**公平锁是按照锁申请的顺序来获取锁，线程直接进入同步队列中排队，队列中的第一个线程才能获得到锁。**



**非公平锁是线程申请锁时，直接尝试加锁，获取不到才会进入到同步队列排队。如果此时该线程刚好获取到了锁，那么它不需要因为队列中有其他线程在排队而阻塞，省去了CPU唤醒该线程的开销。而对于已经在同步队列中的线程，仍然是按照先进先出的公平规则获取锁～**









## 管程



![image-20220821184110184](../../images/synchroized/image-20220821184110184.png)























# lock与synchronized区别



## 总结



**synchronized是关键字，lock是接口。**



**synchronized和lock都可重入。**



**synchronized是非公平锁，lock提供了公平和非公平锁。**



**synchronized不可中断，除非出现异常和线程正常执行完毕，lock可以响应中断，**可通过trylock(long timeout,TimeUnit unit)设置超时方法或者将lockInterruptibly()放到代码块中，调用interrupt方法进行中断。就比如这个lockInterruptibly锁，可以线程调用interrupt中断，就会释放锁，而lock方法只会死等当前线程释放锁资源，不会相应当前线程的调用interrupt方法释放锁。

> tryLock(long time, TimeUnit unit)方法和tryLock()方法是类似的，只不过区别在于这个方法在拿不到锁时会等待一定的时间，在时间期限之内如果还拿不到锁，就返回false。如果一开始拿到锁或者在等待期间内拿到了锁，则返回true。

























## 锁的类型

**synchronized:** 可重入 不可中断 非公平    --关键字

**Lock:** 可重入 可判断 可公平（两者皆可） --接口





## **锁的释放（死锁产生）**

**synchronized:** 在发生异常时候会自动释放占有的锁，因此不会出现死锁

**Lock:** 发生异常时候，不会主动释放占有的锁，必须手动unlock来释放锁，可能引起死锁的发生





## **调度**

**synchronized:** 使用Object对象本身的wait 、notify、notifyAll调度机制

**Lock:** 可以使用Condition进行线程之间的调度



## **用法**

**synchronized:** 在需要同步的对象中加入此控制，synchronized可以加在方法上，也可以加在特定代码块中，括号中表示需要锁的对象。

**Lock:** 一般使用ReentrantLock类做为锁。在加锁和解锁处需要通过lock()和unlock()显示指出。所以一般会在finally块中写unlock()以防死锁。



## **底层实现**

**synchronized:** 底层使用指令码方式来控制锁的，映射成字节码指令就是增加来两个指令：monitorenter和monitorexit。当线程执行遇到monitorenter指令时会尝试获取内置锁，如果获取锁则锁计数器+1，如果没有获取锁则阻塞；当遇到monitorexit指令时锁计数器-1，如果计数器为0则释放锁。

**Lock:** 底层是CAS乐观锁，依赖AbstractQueuedSynchronizer类，把所有的请求线程构成一个CLH队列。而对该队列的操作均通过Lock-Free（CAS）操作。



## 是否可以中断

synchronized是不可中断类型的锁，除非加锁的代码中出现异常或正常执行完成； ReentrantLock则可以中断，可通过trylock(long timeout,TimeUnit unit)设置超时方法或者将lockInterruptibly()放到代码块中，调用interrupt方法进行中断。























