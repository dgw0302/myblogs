---
title: Semaphore&&CountDownLatch
categories:
  - 多线程与并发
tags:
  - 多线程与并发
abbrlink: 56288
date: 2022-04-13 19:54:20
---













线程间通信的工具 **Semaphore**&&**CountDownLatch**在JUC包下

上一篇的BlockingQueue也是线程通信的工具



# Semaphore

## 什么是Semaphore

Semaphore 字面意思是信号量的意思，它的作用是控制同时访问特定资源的线程数目，底层依赖AQS的状态State，是在生产当中比较常用的一个工具类。

感觉很像操作系统信号量机制

## **怎么使用 Semaphore？**

**构造方法**

```java
public Semaphore(int permits)

public Semaphore(int permits, boolean fair)              
```

- permits 表示许可线程的数量
- fair 表示公平性，如果这个设为 true 的话，下次执行的线程会是等待最久的线程

**重要方法**

```java
public void acquire() throws InterruptedException 

public void release() t

ryAcquire（int args,long timeout, TimeUnit unit)
```

 

- acquire() 表示阻塞并获取许可
- release() 表示释放许可

**基本使用**

​            ①**需求场景**

  资源访问，服务限流(Hystrix里限流就有基于信号量方式)。

​           ②**代码实现**

```java
public class SemaphoreRunner {
    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(2);
        for (int i=0;i<5;i++){
            new Thread(new Task(semaphore,"yangguo+"+i)).start();
        }
    }

    static class Task extends Thread{
        Semaphore semaphore;

        public Task(Semaphore semaphore,String tname){
            this.semaphore = semaphore;
            this.setName(tname);
        }

        public void run() {
            try {
                semaphore.acquire();
                System.out.println(Thread.currentThread().getName()+":aquire() at time:"+System.currentTimeMillis());
                Thread.sleep(1000);
                semaphore.release();
                System.out.println(Thread.currentThread().getName()+":aquire() at time:"+System.currentTimeMillis());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        }
    }
}
```



​      ③ **打印结果：**

> Thread-3:aquire() at time:1563096128901
>
> Thread-1:aquire() at time:1563096128901
>
> Thread-1:aquire() at time:1563096129903
>
> Thread-7:aquire() at time:1563096129903
>
> Thread-5:aquire() at time:1563096129903
>
> Thread-3:aquire() at time:1563096129903
>
> Thread-7:aquire() at time:1563096130903
>
> Thread-5:aquire() at time:1563096130903
>
> Thread-9:aquire() at time:1563096130903
>
> Thread-9:aquire() at time:1563096131903

从打印结果可以看出，一次只有两个线程执行 acquire()，只有线程进行 release() 方法后才会有别的线程执行 acquire()。













**Semaphore**&&**CountDownLatch**是基于AQS里面的共享锁









```java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```









```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus either before
     *     or after setHead) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     *
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     */
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```











# CountDownLatch



# 什么是CountDownLatch

 CountDownLatch这个类能够使一个线程等待其他线程完成各自的工作后再执行。例如，应用程序的主线程希望在负责启动框架服务的线程已经启动所有的框架服务之后再执行。

**使用场景：**

Zookeeper分布式锁,Jmeter模拟高并发等

**CountDownLatch如何工作？**

CountDownLatch是通过一个计数器来实现的，计数器的初始值为线程的数量。每当一个线程完成了自己的任务后，计数器的值就会减1。当计数器值到达0时，它表示所有的线程已经完成了任务，然后在闭锁上等待的线程就可以恢复执行任务。

**API**

```java
CountDownLatch.countDown() 

CountDownLatch.await();    
```

​          

**CountDownLatch应用场景例子**

比如陪媳妇去看病。

医院里边排队的人很多，如果一个人的话，要先看大夫，看完大夫再去排队交钱取药。

现在我们是双核，可以同时做这两个事（多线程）。

假设看大夫花3秒钟，排队交费取药花5秒钟。我们同时搞的话，5秒钟我们就能完成，然后一起回家（回到主线程）。

代码如下：

```java
/**
 * 看大夫任务
 */
public class SeeDoctorTask implements Runnable {
    private CountDownLatch countDownLatch;

    public SeeDoctorTask(CountDownLatch countDownLatch){
        this.countDownLatch = countDownLatch;
    }

    public void run() {
        try {
            System.out.println("开始看医生");
            Thread.sleep(3000);
            System.out.println("看医生结束，准备离开病房");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            if (countDownLatch != null)
                countDownLatch.countDown();
        }
    }

}

/**
 * 排队的任务
 */
public class QueueTask implements Runnable {

    private CountDownLatch countDownLatch;

    public QueueTask(CountDownLatch countDownLatch){
        this.countDownLatch = countDownLatch;
    }
    public void run() {
        try {
            System.out.println("开始在医院药房排队买药....");
            Thread.sleep(5000);
            System.out.println("排队成功，可以开始缴费买药");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            if (countDownLatch != null)
                countDownLatch.countDown();
        }
    }
}

/**
 * 配媳妇去看病，轮到媳妇看大夫时
 * 我就开始去排队准备交钱了。
 */
public class CountDownLaunchRunner {

    public static void main(String[] args) throws InterruptedException {
        long now = System.currentTimeMillis();
        CountDownLatch countDownLatch = new CountDownLatch(2);

        new Thread(new SeeDoctorTask(countDownLatch)).start();
        new Thread(new QueueTask(countDownLatch)).start();
        //等待线程池中的2个任务执行完毕，否则一直
        countDownLatch.await();
        System.out.println("over，回家 cost:"+(System.currentTimeMillis()-now));
    }
}
```

​        

























