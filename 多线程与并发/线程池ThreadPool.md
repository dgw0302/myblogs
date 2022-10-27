---
title: 线程池
categories:
  - 多线程与并发
tags:
  -  ThreadPool
abbrlink: 56288
date: 2022-02-16 19:54:20

---



[TOC]





# 前言目录

[美团线程池]: https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html



# **线程池体系图**

![image-20220212192431077](../../images/java%E7%BA%BF%E7%A8%8B%E6%B1%A0%E4%BD%93%E7%B3%BB/image-20220212192431077.png)

# 线程池的使用

**在生产环境中，如果为每个任务分配一个线程，会造成许多问题**：

1. **线程生命周期的开销非常高。**线程的创建和销毁都要付出代价。比如，线程的创建需要时间，延迟处理请求。如果请求的到达率非常高并且请求的处理过程都是轻量级的，那么为每个请求创建线程会消耗大量计算机资源。
2. **资源消耗。** 活跃的线程会消耗系统资源，尤其是内存。如果可运行的线程数量多于处理器数量，那么有些线程会闲置。闲置的线程会占用内存，给垃圾回收器带来压力，大量线程在竞争CPU时，还会产生其他的性能开销。
3. **稳定性。** 如果线程数量过大，可能会造成OutOfMemory异常。

## 使用线程池的好处

1. **降低资源消耗**。通过重复利用已创建的线程降低线程创建和销毁造成的消耗

2. **提高响应速度**。当任务到达时，任务可以不需要等到线程创建就能立即执行。

3. **提高线程的可管理性**。线程是稀缺资源，如果无限制地创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一分配、调优和监控。但是，要做到合理利用线程池，必须对其实现原理了如指掌。

   



Java API针对不同需求，利用`Executors`类提供了4种不同的线程池：newCachedThreadPool, newFixedThreadPool, newScheduledThreadPool, newSingleThreadExecutor，



这四个线程池是通过Executors的静态方法创建的，其底层都是创建了ThreadPoolExecutor对象，不过只是彼此参数不同而已。

# 四大线程池分类

（**阿里巴巴开发手册不建议使用Executors类创建的线程池，可能会产生OOM问题，建议自定义线程池**）



## newCachedThreadPool

创建一个可缓存的无界线程池，该方法无参数。当线程池中的线程空闲时间超过60s则会自动回收该线程，当任务超过线程池的线程数则创建新线程。线程池的大小上限为Integer.MAX_VALUE，可看做是无限大。

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```



**最大线程最大，会导致CPU100%**



## newFixedThreadPool

创建一个固定大小的线程池，该方法可指定线程池的固定大小，对于超出的线程会在`LinkedBlockingQueue`队列中等待。

**LinkedBlockingQueue是无界队列，因为使用默认容量也就是Integer.MAX_VALUE，极易发生OOM**





## newSingleThreadExecutor

创建一个只有线程的线程池，该方法无参数，所有任务都保存队列LinkedBlockingQueue中，等待唯一的单线程来执行任务，并保证所有任务按照指定顺序(FIFO或优先级)执行



**LinkedBlockingQueue是无界队列，因为使用默认容量也就是Integer.MAX_VALUE，极易发生OOM**





## newScheduledThreadPool

创建一个可定时执行或周期执行任务的线程池，该方法可指定线程池的核心线程个数。



**最大线程最大，会导致CPU100%**





上述4个方法的参数对比，如下：

| 工厂方法                | corePoolSize | maximumPoolSize   | keepAliveTime | workQueue           |
| :---------------------- | :----------- | :---------------- | :------------ | :------------------ |
| newCachedThreadPool     | 0            | Integer.MAX_VALUE | 60s           | SynchronousQueue    |
| newFixedThreadPool      | nThreads     | nThreads          | 0             | LinkedBlockingQueue |
| newSingleThreadExecutor | 1            | 1                 | 0             | LinkedBlockingQueue |
| newScheduledThreadPool  | corePoolSize | Integer.MAX_VALUE | 0             | DelayedWorkQueue    |





## 问题









# 线程池工作流程（重点）

![image-20220215201250894](../../images/java%E7%BA%BF%E7%A8%8B%E6%B1%A0%E4%BD%93%E7%B3%BB/image-20220215201250894.png)

1. **判断核心线程池是否已满，即已创建线程数是否小于corePoolSize？没满则创建一个新的工作线程来执行任务。已满则进入下个流程。**
2. **判断工作队列是否已满？没满则将新提交的任务添加在工作队列，等待执行。已满则进入下个流程。**
3. **判断整个线程池是否已满，即已创建线程数是否小于maximumPoolSize？没满则创建一个新的工作线程来执行任务，已满则交给饱和策略来处理这个任务。**



提交优先级和执行优先级

下面这张图可以清楚理解一些

核心员工就是核心线程，非核心员工就是最大线程数减去核心线程数

![image-20220215182650276](../../images/java%E7%BA%BF%E7%A8%8B%E6%B1%A0%E4%BD%93%E7%B3%BB/image-20220215182650276.png)



# 线程池的状态

![image](../../images/java%E7%BA%BF%E7%A8%8B%E6%B1%A0%E4%BD%93%E7%B3%BB/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDI2ODA5OA==,size_16,color_FFFFFF,t_70.png)

## RUNNING

线程池处在RUNNING状态时，能够接收新任务，以及对已添加的任务进行处理





## ShutDown

- 线程池处在SHUTDOWN状态时，不接收新任务，但能处理已添加的任务。工作队列里面的任务仍能执行
- 调用线程池的shutdown()接口时，线程池由RUNNING -> SHUTDOWN

## Stop

- 程池处在STOP状态时，不接收新任务，不处理已添加的任务，并且会中断正在处理的任务。
- 调用线程池的shutdownNow()接口时，线程池由(RUNNING or SHUTDOWN ) -> STOP。

## Tidying

- 当所有的任务已终止，ctl记录的”任务数量”为0，线程池会变为TIDYING状态。当线程池变为TIDYING状态时，会执行钩子函数terminated()。terminated()在ThreadPoolExecutor类中是空的，若用户想在线程池变为TIDYING时，进行相应的处理；可以通过重载terminated()函数来实现。
- 当线程池在SHUTDOWN状态下，阻塞队列为空并且线程池中执行的任务也为空时，就会由 SHUTDOWN -> TIDYING。
  当线程池在STOP状态下，线程池中执行的任务为空时，就会由STOP -> TIDYING。

## Terminated

- 线程池彻底终止，就变成TERMINATED状态。
- 线程池处在TIDYING状态时，执行完terminated()之后，就会由 TIDYING -> TERMINATED。





# 阻塞队列（工作队列）

## 无界队列

队列大小无限制，常用的为无界的**LinkedBlockingQueue**，使用该队列做为阻塞队列时要尤其当心，当任务耗时较长时可能会导致大量新任务在队列中堆积最终导致**OOM**。阅读代码发现，Executors.newFixedThreadPool 采用就是 LinkedBlockingQueue，比如这个坑，当QPS很高，发送数据很大，大量的任务被添加到这个无界LinkedBlockingQueue 中，导致cpu和内存飙升服务器挂掉。



## 有界队列

常用的有两类，一类是遵循FIFO原则的队列如ArrayBlockingQueue，另一类是优先级队列如PriorityBlockingQueue。PriorityBlockingQueue中的优先级由任务的Comparator决定。
 使用有界队列时队列大小需和线程池大小互相配合，线程池较小有界队列较大时可减少内存消耗，降低cpu使用率和上下文切换，但是可能会限制系统吞吐量。



## 同步移交队列

如果不希望任务在队列中等待而是希望将任务直接移交给工作线程，可使用SynchronousQueue作为等待队列。SynchronousQueue不是一个真正的队列，而是一种线程之间移交的机制。要将一个元素放入SynchronousQueue中，必须有另一个线程正在等待接收这个元素。只有在使用无界线程池或者有饱和策略时才建议使用该队列。





# 拒绝策略

有四种拒绝策略

只有当前线程池线程数量大于总数量并且阻塞队列里面的任务也满了，再提交任务就会触发拒绝策略。



## AbortPolicy(默认)

默认的拒绝策略

```java
public static class AbortPolicy implements RejectedExecutionHandler {
    /**
     * Creates an {@code AbortPolicy}.
     */
    public AbortPolicy() { }

    /**
     * Always throws RejectedExecutionException.
     *
     * @param r the runnable task requested to be executed
     * @param e the executor attempting to execute this task
     * @throws RejectedExecutionException always
     */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() +
                                             " rejected from " +
                                             e.toString());
    }
}
```

**这个拒绝策略是再提交任务就直接抛异常**



## CallerRunsPolicy（重点）

```java
public static class CallerRunsPolicy implements RejectedExecutionHandler {
    /**
     * Creates a {@code CallerRunsPolicy}.
     */
    public CallerRunsPolicy() { }

    /**
     * Executes task r in the caller's thread, unless the executor
     * has been shut down, in which case the task is discarded.
     *
     * @param r the runnable task requested to be executed
     * @param e the executor attempting to execute this task
     */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            r.run();
        }
    }
}
```

**这个策略是提交的任务由提交任务的线程就执行，谁提交的任务谁执行**







## DiscardOldestPolicy（重点）

```java
public static class DiscardOldestPolicy implements RejectedExecutionHandler {
    /**
     * Creates a {@code DiscardOldestPolicy} for the given executor.
     */
    public DiscardOldestPolicy() { }

    /**
     * Obtains and ignores the next task that the executor
     * would otherwise execute, if one is immediately available,
     * and then retries execution of task r, unless the executor
     * is shut down, in which case task r is instead discarded.
     *
     * @param r the runnable task requested to be executed
     * @param e the executor attempting to execute this task
     */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            e.getQueue().poll();
            e.execute(r);
        }
    }
}
```

**这个是将阻塞队列对头的任务弹出，然后提高当前的任务到阻塞队列里面去。**



## DiscardPolicy

```java
public static class DiscardPolicy implements RejectedExecutionHandler {
    /**
     * Creates a {@code DiscardPolicy}.
     */
    public DiscardPolicy() { }

    /**
     * Does nothing, which has the effect of discarding task r.
     *
     * @param r the runnable task requested to be executed
     * @param e the executor attempting to execute this task
     */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    }
}
```



**什么都不干**





# **execute和submit的区别**

- execute只能提交Runnable类型的任务，无返回值。**submit既可以提交Runnable类型的任务，也可以提交Callable类型的任务，会有一个类型为Future的返回值，但当任务类型为Runnable时，返回值为null。**
- **execute在执行任务时，如果遇到异常会直接抛出，**
- **而submit不会直接抛出，只有在使用Future的get方法获取返回值时，才会抛出异常。**submit方便异常处理
- submit内部调用了execute方法 

![image-20220916164801323](../../images/%E7%BA%BF%E7%A8%8B%E6%B1%A0ThreadPool/image-20220916164801323.png)











# ThreadPoolExecutor(重点)

这个**ThreadPoolExecutor**类有四个构造方法

```java
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
        BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler);
```

但其实前面三个构造器都是调用的第四个构造器进行的初始化工作。

下面是第四个构造器（最全的）

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
```

## 构造器参数

构造器有**7**个参数：

- **corePoolSize**：核心线程池的大小，在创建了线程池后，默认情况下，线程池中并没有任何线程，而是等待有任务到来才创建线程去执行任务，除非调用了**prestartAllCoreThreads()**或者**prestartCoreThread()**方法，从这2个方法的名字就可以看出，是预创建线程的意思，即在没有任务到来之前就创建corePoolSize个线程或者一个线程。

- **maximumPoolSize**:线程池最大线程数，它表示在线程池中最多能创建多少线程。

- **keepAliveTime**：表示非核心线程没有任务时执行时最多保持多久时间会终止。默认情况下，只有当线程池中的线程数大于corePoolSize时，keepAliveTime才会起作用，直到线程池中的线程数不大于corePoolSize：即当线程池中的线程数大于corePoolSize时，如果一个线程空闲的时间达到keepAliveTime，则会终止，直到线程池中的线程数不超过corePoolSize；但是如果调用了**allowCoreThreadTimeOut(boolean)**方法，在线程池中的线程数不大于corePoolSize时，keepAliveTime参数也会起作用，直到线程池中的线程数为0；

- **workQueue**：一个阻塞队列，用来存储等待执行的任务，这个参数的选择会对线程池的运行过程产生重大影响，一般来说，这里的阻塞队列有以下几种选择：

- ```
  ArrayBlockingQueue;
  LinkedBlockingQueue;
  SynchronousQueue;
  ```

- **threadFactory**：线程工厂，主要用来创建线程。

- **handler**：表示当拒绝处理任务时的策略



**如果在自定义线程池的时候不指定线程工厂，会有默认的线程工厂。**

```java
static class DefaultThreadFactory implements ThreadFactory {
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;

    DefaultThreadFactory() {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() :
                              Thread.currentThread().getThreadGroup();
        namePrefix = "pool-" +
                      poolNumber.getAndIncrement() +
                     "-thread-";
    }

    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,
                              namePrefix + threadNumber.getAndIncrement(),
                              0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
```

**也会有默认的拒绝策略**



```java
public static class AbortPolicy implements RejectedExecutionHandler {
    /**
     * Creates an {@code AbortPolicy}.
     */
    public AbortPolicy() { }

    /**
     * Always throws RejectedExecutionException.
     *
     * @param r the runnable task requested to be executed
     * @param e the executor attempting to execute this task
     * @throws RejectedExecutionException always
     */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() +
                                             " rejected from " +
                                             e.toString());
    }
}
```

## 线程工厂

```java
public interface ThreadFactory {

    /**
     * Constructs a new {@code Thread}.  Implementations may also initialize
     * priority, name, daemon status, {@code ThreadGroup}, etc.
     *
     * @param r a runnable to be executed by new thread instance
     * @return constructed thread, or {@code null} if the request to
     *         create a thread is rejected
     */
    Thread newThread(Runnable r);
}
```

线程工厂就简单的创建线程

## 拒绝策略（看上面）



**讲完前面的，接下来我们来看具体的方法源码**



![image-20220215201426973](../../images/java%E7%BA%BF%E7%A8%8B%E6%B1%A0%E4%BD%93%E7%B3%BB/image-20220215201426973.png)









## 内部核心类与数据结构

Worker是线程池ThreadPoolExecutor的一个内部类，其有一个成员变量thread（线程），所以我们可以把一个Worker假以理解为一个线程。

任务队列是BlockingQueue，都说精通线程池了，如果熟悉这个队列的话，能不熟悉它的几个方法吗？线程池正是利用poll方法的超时时间来决定要不要回收空闲线程的。





## execute

直接上源码

```java
// int是4个字节，有32位，这里的ctl前3位表示线程池的状态，后29位标识工作线程的数量
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
// Integer.SIZE - 3 = 29
private static final int COUNT_BITS = Integer.SIZE - 3;
// 容量 000 11111111111111111111111111111
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;//二进制29个1        

// runState is stored in the high-order bits
// 运行中状态 111 00000000000000000000000000000 （-536870912）  括号内为十进制的
private static final int RUNNING    = -1 << COUNT_BITS;
// 关闭状态 000 00000000000000000000000000000   （0）
private static final int SHUTDOWN   =  0 << COUNT_BITS;
// 停止状态 001 00000000000000000000000000000   （536870912）
private static final int STOP       =  1 << COUNT_BITS;
// 整理状态 010 00000000000000000000000000000   （1073741824）
private static final int TIDYING    =  2 << COUNT_BITS;
// 终结状态 011 00000000000000000000000000000   （1610612736）
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
// 先非然后位与运算符获取线程池运行的状态，也就是前3位
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// 位与运算符获取工作线程数量，也就是后29位
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

**核心函数**

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
   
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         * 有以下3个步骤
         *
         * 1.如果少于corePoolSize的线程在运行，那么试着启动一个新线程，其中用给定指令作为first task。
         * 这会调用addWorker去原子性得检查runState和workerCoune，因此可以防止错误报警，在错误报警不应该时通过返回false来添加线程
         * 2.如果任务被成功排队，我们任然应该第二次检查是否添加一个新线程（因为可能存在在最后一次检查后挂掉的情况）
         * 或者在进入这个方法期间线程池shutdown。所以我们再次检查状态，如果已关闭和有必要则退出队列，或者如果没有的话就开始一个新的线程。
         * 3.如果我们无法将task入队，那么我们试图添加新线程。如果失败，那么知道我们shutdown或者是饱和的并拒绝task。
         */
    int c = ctl.get();//这个ctl存有运行状态和线程数（前高三位是运行状态，后29位是线程数）
    if (workerCountOf(c) < corePoolSize) {//当前线程池里面活着的线程数（工作线程的数量）小于核心线程数时，直接调用addworker来创建新线程来执行任务 
        if (addWorker(command, true))//true或false代表核心与非核心
            return;
        c = ctl.get();//
    }
    if (isRunning(c) && workQueue.offer(command)) {//如果线程池在运行，并且核心线程满了，能把任务放在任务队列里面
        int recheck = ctl.get();
        // 这里重新检查是为了以下2种情况
         // 1.当offer方法执行之后，线程池关闭了，回滚之前放入队列的操作并拒绝任务
        if (! isRunning(recheck) && remove(command))//这里进行再次检查，如果线程池没在运行并且成功删除task后，使用拒绝策略拒绝该task
            reject(command);
        //2.线程池里没有可用的消费线程，比如现在核心线程数就1个，前一个任务抛异常了
        // 那么现在就没有可用的消费线程了，所以要判断还有没有Worker，这步很关键
        else if (workerCountOf(recheck) == 0)　//如果已经将task添加到队列中，而此时没有worker的话，那么新建一个worker。稍后这个空闲的worker就会自动去队列里面取任务来执行
            addWorker(null, false);
    }
    else if (!addWorker(command, false))//如果添加非核心线程失败，调用拒绝函数
        reject(command);
}
```

　　Worker实现了Runnable接口，说明可以当做一个可执行的任务

## addWorker

**addworker创建新线程**

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        //获取线程池状态和线程数
        int c = ctl.get();
        //rs是线程池状态信息
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        //以下几种情况会返回false
        // 1.线程池状态为STOP，TIDYING，TERMINATED
        // 2.线程池状态为SHUTDOWN且工作线程的firstTask不为空
        // 3.线程池状态为SHUTDOWN且队列为空
        if (rs >= SHUTDOWN &&
            
            //这里主要是要保证SHUTDOWN状态下，不接收外部任务，但是要消费工作队列任务。
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            //获取工作线程数
            int wc = workerCountOf(c);
              // 如果工作线程大于最大容量或者核心线程大于核心线程数(核心与非核心三目运算符里面判断)返回false
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            //添加工作线程+1,然后结束外循环
            if (compareAndIncrementWorkerCount(c))
                break retry;
            // 重新获取线程池状态和线程数
            c = ctl.get();  // Re-read ctl
             // 如果线程池状态变了，那么重新走外循环
            if (runStateOf(c) != rs)
                continue retry;
            
        }
    }
	// 线程是否开始工作
    boolean workerStarted = false;
    // 线程是否添加到工作线程集合
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
             // 利用显式锁加锁添加Worker
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                int rs = runStateOf(ctl.get());
   // 如果线程池状态是RUNNING或者是SHUTDOWN&&第一个任务为空
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                       // 检查这个线程是否处于活动状态 - RUNNABLE或者RUNNING
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    //添加到工作线程集合
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
              // 如果添加到工作线程集合则开始工作
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
           // 如果线程没有开始工作，那么工作线程数量-1
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```





## runWorker

当Worker里的线程启动时，就会调用该方法。

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    // 这里为什么要先unlock一下呢？到这一行代码为止，我们没有进行任何的任务处理
     // Worker的构造函数中，setState(-1);这一行代码抑制了线程中断，所以这里需要unlock从而允许中断
    // 初始化worker的state = 0, exclusiveOwnerThread = null 解锁
    w.unlock(); // 允许中断
     // 是否是异常终止的标识，默认为true。有2中情况为true
    // 1.执行任务抛出了异常
    // 2.worker被中断
    boolean completedAbruptly = true;
    try {
        //为什么线程可以重用
        // 获取任务，如果getTask()方法返回null，那么随之worker也要-1，之后有getTask()方法分析
        // 只有在等待从workQueue队列里获取任务的时候才能中断。
        // 第一次执行传入的任务，之后从workQueue队列里获取任务，如果队列为空则等待keepAliveTime这么久         
        while (task != null || (task = getTask()) != null) {
            // 加锁的目的在于防止在执行任务的时候，中断当前worker

            w.lock();
            // 这个方法比较重要，当线程池正在关闭，确保worker被中断
           //        3个判断
//        1、runStateAtLeast(ctl.get(), STOP)为真说明当前状态大于等于STOP 此时需要给他一个中断信号
//        2、wt.isInterrupted()查看当前是否设置中断状态如果为false则说明为设置中断状态
//        3、Thread.interrupted() && runStateAtLeast(ctl.get(), STOP) 获取当前中断状态且清除中断状态
//          这个判断为真的话说明当前被设置了中断状态(有可能是线程池执行的业务代码设置的，然后重置了)且当前状态变成了大于等于STOP的状态了
//                 
//     整个判断为真的两种情况
//                 1、如果当前线程大于等于STOP 且未设置中断状态 整个判断为true 第一个runStateAtLeast(ctl.get(), STOP)为true !wt.isInterrupted()为true
//                 2、第一次判断的时候不大于STOP 且当前设置了中断状态(Thread.interrupted()把中断状态又刷新了) 且设置完了之后线程池状态大于等于STOP了
//                    Thread.interrupted() && runStateAtLeast(ctl.get(), STOP) 为true !wt.isInterrupted()为true
//                
//     结合if判断里面的代码来看
//                 也就是说如果线程池状态大于等于STOP则设置当前线程的中断状态
//                 如果线程池状态小于STOP则清除中断状态
            if ((runStateAtLeast(ctl.get(), STOP) ||(Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) && !wt.isInterrupted() )
                wt.interrupt();
            try {
                   // 执行任务之前的操作，如统计日志等，子类自己实现
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                       // 将异常包装成Error抛出
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                   // 解锁，一次任务的执行结束
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
                // 结束worker的清理工作
        processWorkerExit(w, completedAbruptly);
    }
}
```

## getTask()

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

           // 当线程池状态是STOP或者SHUTDOWN并且workQueue队列是空的，返回null
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // timed用来判断该工作线程是否有超时控制？
        // allowCoreThreadTimeOut参数是是否允许核心线程也有keepAliveTime这么一个属性
        // 核心线程默认是没有超时限制
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
		// 条件1：如果工作线程大于最大线程数或者超时了
        // 条件2：如果工作线程大于1或者workQueue队列为空
        // 满足以上2个条件则返回null
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            // 一个是阻塞方法，一个是非阻塞方法，关键还是看timed这个变量，见上
            //这个poll如果队列为null,会阻塞，阻塞到一定时间就会返回null
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```



## shutdown

将线程池状态置为`SHUTDOWN`,并不会立即停止：

- 停止接收外部submit的任务
- 内部正在跑的任务和队列里等待的任务，会执行完
- 等到第二步完成后，才真正停止

```java
public void shutdown() {
    // 获取显式锁
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 检查shutdown权限
        checkShutdownAccess();
        // 将线程池状态改为SHUTDOWN
        advanceRunState(SHUTDOWN);
        // 中断空闲worker
        // 如果该线程正在工作，则不中断
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    // 保证workQueue里的剩余任务可以执行完
    tryTerminate();
}
```



## shutdownNow

将线程池状态置为`STOP`。**企图**立即停止，事实上不一定：

- 跟shutdown()一样，先停止接收外部提交的任务

- 忽略队列里等待的任务

- 尝试将正在跑的任务`interrupt`中断

- 返回未执行的任务列表

  它试图终止线程的方法是通过调用Thread.interrupt()方法来实现的，但是大家知道，这种方法的作用有限，如果线程中没有sleep 、wait、Condition、定时锁等应用, interrupt()方法是无法中断当前的线程的。所以，ShutdownNow()并不代表线程池就一定立即就能退出，**它也可能必须要等待所有正在执行的任务都执行完成了才能退出。**

```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        // 将线程池状态改为STOP
        advanceRunState(STOP);
        interruptWorkers();
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```





## awaitTermination()

当前线程阻塞，直到

- 等所有已提交的任务（包括正在跑的和队列中等待的）执行完
- 或者等超时时间到
- 或者线程被中断，抛出`InterruptedException`

然后返回true（shutdown请求后所有任务执行完毕）或false（已超时）







# 线程池线程回收





## 非核心线程怎么回收

如果经过 keepAliveTime 时间后，超过核心线程数的线程还没有接受到新的任务，就会被回收。





### 解释



**循环会结束，Worker会主动消除自身在线程池内的引用。最后由jvm回收**



这个方法里会做非核心线程的判断(满足以下任意一个条件就会被判定为非核心线程: 1.allowCoreThreadTimeOut为真 2.当前线程数＞核心线程数)，如果当前线程被判定为非核心线程，那么**从阻塞队列获取任务的时候会设置一个超时时间**，如果在规定时间内没有获得任务，getTask方法向runWorker方法返回null，进而从while循环跳出，执行终结线程的逻辑



**有个timed标志：如果allowCoreThreadTimeOut为true或者工作线程大于核心线程，那么timed也为true；**

**接下来的主要逻辑里面，如果线程发生了空闲时间超时，getTask方法就会向runWorker()方法返回null ,这样runWorker里面的while()循环就会停掉，然后执行终结线程的逻辑**，最后由垃圾回收器回收。





**看下面的**

**在getTask里面，如果是非核心线程（当前线程数大于核心线程）并且超时了就会返回Null,或者是核心线程但标志allowCoreThreadOut为true并且也超时了,也就是开启核心线程回收了也会返回null.**



**超时的逻辑**

> 阻塞队列poll(long timeout, TimeUnit unit)   ：**返回值：**此方法从此LinkedBlockingQueue的头部检索并删除元素，如果在元素可用之前经过了指定的等待时间，则为null。



**怎样判断超时，是靠阻塞队列的poll方法和标志位，会传入超时时间，如果poll返回null则为超时，标志位timeOut为false，如果不是则没有超时,那么timeOut还是true.**



**返回值：**此方法从此LinkedBlockingQueue的头部检索并删除元素，如果在元素可用之前经过了指定的等待时间，则为null。

BlockingQueue接口的poll(long timeout，TimeUnit unit)方法通过从队列中删除该元素来返回BlockingQueue的头部。可以说此方法从此LinkedBlockingQueue的头部检索并删除了元素。如果队列为空，则poll()方法将等待直到指定时间元素可用。





### getTask里面的主逻辑



```java
private Runnable getTask() { 


/*********/

        // 当前线程是否允许超时销毁的标志
        // 允许超时销毁：当线程池允许核心线程超时 或 工作线程数>核心线程数
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        // 如果(当前线程数大于最大线程数 或 (允许超时销毁 且 当前发生了空闲时间超时))
        // 且(当前线程数大于1 或 阻塞队列为空)
        // 则减少worker计数并返回null
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

/*********/

}
```



### **runWorker主要逻辑**

```java
final void runWorker(Worker w) {


        try{
             while (task != null || (task = getTask()) != null) {

                    task.run(); // 执行线程的run方法

          }
        }   finally {
 		 processWorkerExit(w, completedAbruptly);//获取不到任务时，主动回收自己	
	}


}
```



keepAliveTime:具体判断有没有超时是在blokingqueue的poll()方法参数提交，并在poll方法里面具体有判断。

### processWorkerExit

**线程回收的工作是在processWorkerExit方法完成的。**

![图10 线程销毁流程](../../images/%E7%BA%BF%E7%A8%8B%E6%B1%A0ThreadPool/90ea093549782945f2c968403fdc39d415386.png)





事实上，在这个方法中，将线程引用移出线程池就已经结束了线程销毁的部分。但由于引起线程销毁的可能性有很多，线程池还要判断是什么引发了这次销毁，是否要改变线程池的现阶段状态，是否要根据新状态，重新分配线程。



## 核心线程会被回收吗

核心线程通常不会回收，java核心线程池的回收由**allowCoreThreadTimeOut**方法的参数控制，默认为false，若开启为true，则此时线程池中不论核心线程还是非核心线程，只要其空闲时间达到keepAliveTime都会被回收。





下面为具体回收核心线程的操作

```java
static ThreadPoolExecutor executor=new ThreadPoolExecutor(8,16,0,TimeUnit.SECONDS,new LinkedBlockingQueue<>(10));

static {
    //如果设置为true,当任务执行完后，所有的线程在指定的空闲时间后，poolSize会为0
    //如果不设置，或者设置为false，那么，poolSize会保留为核心线程的数量
    executor.allowCoreThreadTimeOut(true);
}
```

# **线程池被创建后有没有线程**



看线程池有没有预热



**如果没有预热:是没有线程的**

**如果预热了：如果预热了，线程池里面有所有核心线程或者一个核心线程**（prestartAllCoreThreads()或prestartCoreThread() 
）。

**线程池的预热仅仅针对核心线程，使用方法也会因为场景不用有所不同，大家按需选择，无需千篇一律。**



# 线程池参数设置





## 核心线程数设置

CPU密集型：cpu核 + 1

IO密集型： cpu核数 * 2





## 最大线程数

一般是核心线程数的2倍











# 线程池优化











# 线程池抛出异常处理（※）





外部：https://www.sohu.com/a/328522906_100109711



必看：https://www.cnblogs.com/fanguangdexiaoyuer/p/12332082.html



**如果线程出现异常，则会将该线程从线程池中移除销毁，然后再新创建一个线程加入到线程池中，也就是说在任务发生异常的时候，会终结掉运行它的线程。**



## 当一个线程池里面的线程异常后:

**1、当执行方式是execute时,可以看到堆栈异常的输出**

原因：ThreadPoolExecutor.runWorker()方法中，task.run()，即执行我们的方法，如果异常的话会throw x;所以可以看到异常。



**2、当执行方式是submit时,堆栈异常没有输出。但是调用Future.get()方法时，可以捕获到异常**

原因：ThreadPoolExecutor.runWorker()方法中，task.run()，其实还会继续执行FutureTask.run()方法，再在此方法中c.call()调用我们的方法，
如果报错是setException()，并没有抛出异常。当我们去get()时，会将异常抛出。



**3、不会影响线程池里面其他线程的正常执行**



**4、线程池会把这个线程移除掉，并创建一个新的线程放到线程池中**

当线程异常，会调用ThreadPoolExecutor.runWorker()方法最后面的finally中的processWorkerExit()，会将此线程remove，并重新addworker()一个线程。







## 异常解决（下面两种）



## try catch

直接在业务代码，也就是run方法里面try catch住代码







缺点：



1)所有的不同任务类型都要trycatch，增加了代码量。

2)不存在checkedexception的地方也需要都trycatch起来，代码丑陋。



## 线程池实现





**java中的线程池用的是ThreadPoolExecutor，真正执行代码的部分是runWorker方法：final void runWorker(Worker w)**





程序会捕获包括Error在内的所有异常，并且在程序最后，将出现过的异常和当前任务传递给afterExecute方法。**

**而ThreadPoolExecutor中的afterExecute方法是没有任何实现的。**

**protected void afterExecute(Runnable r, Throwable t) { }**





### 自定义线程池

> 程序会捕获包括Error在内的所有异常，并且在程序最后，将出现过的异常和当前任务传递给afterExecute方法。**
>
> **而ThreadPoolExecutor中的afterExecute方法是没有任何实现的。**
>
> **protected void afterExecute(Runnable r, Throwable t) { }**





**自定义线程池，继承ThreadPoolExecutor并复写其afterExecute(Runnable r, Throwable t)方法。**



实现Thread.UncaughtExceptionHandler接口

实现Thread.UncaughtExceptionHandler接口，

实现void uncaughtException(Thread t, Throwable e);方法，

并将该handler传递给线程池的ThreadFactory





### 继承ThreadGroup

覆盖其uncaughtException方法。(与第二种方式类似，因为ThreadGroup类本身就实现了Thread.UncaughtExceptionHandler接口)







### 采用Future模式

尤其注意：上面三种方式针对的都是通过execute(xx)的方式提交任务，如果你提交任务用的是submit()方法，那么上面的三种方式都将不起作用,而应该使用下面的方式









**如果提交任务的时候使用的方法是submit，那么该方法将返回一个Future对象，所有的异常以及处理结果都可以通过future对象获取。**

**采用Future模式，将返回结果以及异常放到Future中，在Future中处理**





![image-20220822001811297](../../images/%E7%BA%BF%E7%A8%8B%E6%B1%A0ThreadPool/image-20220822001811297.png)









# 线程池刚启动





线程池启动的时候不会创建任何线程，假设请求进来6个，则会创建5个核心线程来处理五个请求，另一个没被处理到的进入到队列。这时候有进来99个请求，线程池发现核心线程满了，队列还在空着99个位置，所以会进入到队列里99个，加上刚才的1个正好100个。这时候再次进来5个请求，线程池会再次开辟五个非核心线程来处理这五个请求。目前的情况是线程池里线程数是10个RUNNING状态的，队列里100个也满了。如果这时候又进来1个请求，则直接走拒绝策略。



# 监控线程池



第一篇

csdn:https://blog.csdn.net/f80407515/article/details/115916821



第二篇

https://blog.csdn.net/xupeiyan/article/details/122666245







# 动态线程池

https://blog.csdn.net/weixin_41757663/article/details/106902902





https://juejin.cn/post/7008082134673915934





原理：通过线程池可视化管理平台对线程池参数进行更改，推送到nacos配置中心，dynamic-thread-pool-autoconfigure模块会对nacos监听，实时感应线程池参数变化，然后设置响应参数，实时生效。ThreadPoolExecutor提供了setCorePoolSize()、setMaximumPoolSzie()、setRejectedExecutionHandle()等方法，可对线程池进行动态参数配置。但是阻塞队列本身没有对外提供修改队列大小的方法，那我们需要继承AbstractQueue然后增加一个setCapacity()方法即可实现改功能。





> **动态调参**：支持线程池参数动态调整、界面化操作；包括修改线程池核心大小、最大核心大小、队列长度等；参数修改后及时生效。 **任务监控**：支持应用粒度、线程池粒度、任务粒度的Transaction监控；可以看到线程池的任务执行情况、最大任务执行时间、平均任务执行时间、95/99线等。 **负载告警**：线程池队列任务积压到一定值的时候会通过大象（美团内部通讯工具）告知应用开发负责人；当线程池负载数达到一定阈值的时候会通过大象告知应用开发负责人。 **操作监控**：创建/修改和删除线程池都会通知到应用的开发负责人。 **操作日志**：可以查看线程池参数的修改记录，谁在什么时候修改了线程池参数、修改前的参数值是什么。 **权限校验**：只有应用开发负责人才能够修改应用的线程池参数。











**观察 CPU 利用率、系统负载、GC、内存、RT、吞吐量 等各种综合指标数据，来找到一个相对比较合理的值。**

要对线程池的运行状况有感知，比如当前线程池的负载是怎么样的？分配的资源够不够用？任务的执行情况是怎么样的？是长任务还是短任务？











# 面试篇





## 线程池的任务队列怎么保证一个任务不被多个线程拿到





## 线程池的使用场景，解决了什么问题



本来有的缺点：

> 1、首先频繁的创建、销毁对象是一个很消耗性能的事情；
>
> 2、如果用户量比较大，导致占用过多的资源，可能会导致我们的服务由于资源不足而宕机；
>
> 3、综上所述，在实际的开发中，这种操作其实是不可取的一种方式。





**线程池的改进**

线程池中线程的使用率提升，减少对象的创建、销毁；

线程池可以控制线程数，有效的提升[服务器](https://cloud.tencent.com/product/cvm?from=10680)的使用资源，避免由于资源不足而发生宕机等问题；



频繁的创建和销毁线程很浪费资源，线程池可以达到线程资源复用

提高线程的可管理性，能安全有效的管理线程资源，避免不加限制无限申请造成资源耗尽风险

**提高响应速度**：当任务到达时，任务可以不需要等到线程创建就能立即执行。

线程池异步处理任务







## 线程池的任务队列可以设置成非共享的吗，比如每个线程都有一个私有队列。

**如果不可以，为什么；如果可以，你怎么去实现？**





是可以设计成每个工作线程有私有任务队列的，如果这样设计，在提交新任务的时候可以将一些负载均衡的策略加入，优先将任务提交给任务少的线程，但是，想要判断哪个线程的任务少，实现起来有两种方式：1.轮询一遍所有任务队列，判断出一个目标队列，但是轮询的过程存在**并发修改**，所以结果是不准确的。2.使用一个共享堆来记录所有线程的任务数，然而堆的修改仍然需要**加锁**，所以思来想去还不如直接使用共享队列了呢。



消耗内存



## 实现一个阻塞队列

```java


class ArrayBlockingQueue{
  int size = 10;
  Deque queue = new ArrayDeque();
  ReetrantLock lock = new ReetrantLock();
  Condition notEmpty = lock.newCondition(), notFull = lock.newCondition();
    
  //不空存消费者
  //不满存生产者
    
  Object poll() {
    lock.lock();
    try {
      while (queue.isEmpty()) {
          //当队列为空了，那么不空的condition就阻塞消费者
        notEmpty.await();
      }
      Object res = queue.pollFirst();
      //队列里面有数据了，唤醒不满condition，生产者
      notFull.signal();
      return res;
    } catch (Exception e) {
      //
    } finally {
      lock.unlock();
    }
  }
  void offer(Object object) {
    lock.lock();
    try {
      while (queue.size() == size) {
          //当队列满了，阻塞不满condition，阻塞x'z
          
        notFull.await();
      }
      queue.offerLas(object);
        
        //队列消费完元素了
        //唤醒不空condition
      notEmpty.signal();
      return;
    } catch (Exception e) {
      //
    } finally {
      lock.unlock();
    }
  }
}
```





## shutdown()与shutdownNow()区别



**shutdown只是将线程池的状态设置为SHUTWDOWN状态，正在执行的任务会继续执行下去，没有被执行的则中断。**

**而shutdownNow则是将线程池的状态设置为STOP，正在执行的任务则被停止，没被执行任务的则返回。**















