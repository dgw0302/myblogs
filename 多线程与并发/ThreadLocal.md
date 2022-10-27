---
title: ThreadLocal
categories:
  - 多线程与并发
tags:
  - ThreadLocal
abbrlink: 55249
date: 2022-05-19
---



本文是ThreadLocal的简单讲解

# ThreadLocal



ThreadLocal是一个**全局对象**，ThreadLocal是线程范围内变量共享的解决方案；**ThreadLocal可以看作是一个map集合，key就是当前线程，value就是要存放的变量**。



**解释**

**ThreadLocal的作用主要是做数据隔离，为每个线程在本地复制了一个共享变量，达到线程隔离，来解决并发问题，复制后的变量是线程隔离的，也就是其他线程干涉不了（防止）**



他可以为每个线程在本地创建一个共享变量的副本，那到底是怎样维护的呢，ThreadLocal类里面有个ThreadLocalMap类（也就是有个Map），

**每个线程类都会维护一个ThreadLocalMap属性，**

**Map的key 是ThreadLocal实例对象，value就是当前线程复制的变量副本。**



## 为什么要有ThreadLocal

ThreadLocal的作用主要是做数据隔离，填充的数据只属于当前线程，变量的数据对别的线程而言是相对隔离的，在多线程环境下，如何防止自己的变量被其它线程篡改。







## spring使用



其实我第一时间想到的就是Spring实现事务隔离级别的源码，

Spring采用Threadlocal的方式，来保证单个线程中的数据库操作使用的是同一个数据库连接，同时，采用这种方式可以使业务层使用事务时不需要感知并管理connection对象，通过传播级别，巧妙地管理多个事务配置之间的切换，挂起和恢复。







## 怎么用

```java
public static final ThreadLocal<String> threadLocal = new ThreadLocal<>();



public static void main(String[] args) {
    

    for( int i = 0; i < 10; i++) {

       new Thread(new Runnable() {
           @Override
           public void run() {
               threadLocal.set(Thread.currentThread().getName());
               System.out.println("线程  " + Thread.currentThread().getName() + "   local       " + threadLocal.get());
           }
       }).start();


    }

}
```

下面是结果

```java
线程  Thread-1   local       Thread-1
线程  Thread-2   local       Thread-2
线程  Thread-3   local       Thread-3
线程  Thread-0   local       Thread-0
线程  Thread-6   local       Thread-6
线程  Thread-5   local       Thread-5
线程  Thread-7   local       Thread-7
线程  Thread-9   local       Thread-9
线程  Thread-4   local       Thread-4
线程  Thread-8   local       Thread-8
```



我们定义了一个ThreadLocal全局对象，每个线程通过set方法设置一个变量副本，通过get方法又可以获取到设置的变量。



## set

```java
public void set(T value) {
    
    //先获得当前线程
    Thread t = Thread.currentThread();
    
    //获得当前线程里面的ThreadLocalMap属性变量
    ThreadLocalMap map = getMap(t);
    if (map != null)
        //如果这个ThreadLocalMap变量不为null,设置value,否则创建ThreadLocalMap变量
        map.set(this, value);
    else
        createMap(t, value);
}
```



## get



```java
public T get() {
    
    //先获取当前线程对象
    Thread t = Thread.currentThread();
       //获得当前线程里面的ThreadLocalMap变量
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        //通过全局ThreadLocal对象来获取线程里面ThreadLocalMap里面ThreadLocal对应的value;
        //ps:在ThreadLocalMap里面ThreadLocal当作key
        
        
        //获取ThreadLocalMap里面ThreadLocal对应的Entry变量
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```





- 每个线程都有自己的ThreadLocalMap对象；(ThreadLocal 多线程下资源隔离的根本原因)。
- **各个线程在调用同一个ThreadLocal对象的`set(value)`设置值的时候，是往各自的ThreadLocalMap对象数组中设置值。**
- 至于当前值放置在数组中的下标位置，则是通过ThreadLocal对象的`threadLocalHashCode`计算而来。即多线程环境下ThreadLocal对象的`threadLocalHashCode`是共享的。
- ThreadLocal对象的threadLocalHashCode是一个原子自增的变量，通过类方法initValue初始化值。
  即：当实例化ThreadLocal对象ThreadLocal local = new ThreadLocal();时，就会初始化threadLocalHashCode的值，这个值不会再变。所以，同一个线程在同一个ThreadLocal对象中set()值，只能保存最后一次set的值。
- 为什么每个线程都有自己的ThreadLocalMap对象，且是一个数组呢？
  答：**根据以上的分析，多个线程操作一个ThreadLocal对象就能达到线程之间资源隔离**。**而采用数组是因为可能一个线程需要通过多个ThreadLocal对象达到多个资源隔离。每个不同的ThreadLocal对象的threadLocalHashCode都不一样，也就映射到ThreadLocalMap对象数组下的不同下标。**
- **每个线程的ThreadLocalMap对象是通过偏移位置（线性探测法）的方式解决hash碰撞。**
- 每个线程都有自己的ThreadLocalMap对象也有扩容机制，且是天然线程安全的。



# ThreadLocalMap

**ThreadLocalMap是ThreadLocal里面一个静态内部类，同时也是Thread类里面的一个属性，说明每个线程都有一个ThreadLocalMap，底层是Entry[]数组，每个ThreadLocal的threadLocalHashCode值经过计算算出其在Entry[]数组的下标,value值保存在Entry对象里面。**



```java
int i = key.threadLocalHashCode & (len-1);
```



```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}

/**
 * The initial capacity -- MUST be a power of two.
 */
private static final int INITIAL_CAPACITY = 16;

/**
 * The table, resized as necessary.
 * table.length MUST always be a power of two.
 */
private Entry[] table;

/**
 * The number of entries in the table.
 */
private int size = 0;

/**
 * The next size value at which to resize.
 */
private int threshold; // Default to 0
```







**ThreadLocalMap中解决Hash冲突的方式并非链表的方式，而是采用线性探测的方式**









# 内存泄漏



```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```



**作为Entry的key的ThreadLocal就是使用的弱引用**

内部类Entry继承了弱引用WeakReference：



> **弱引用**，JVM进行垃圾回收时，无论内存是否充足，都会回收被弱引用关联的对象。



## 解答

`ThreadLocalMap`使用`ThreadLocal`的弱引用作为`key`，如果一个`ThreadLocal`没有外部强引用来引用它，那么系统 GC 的时候，这个`ThreadLocal`势必会被回收，这样一来，`ThreadLocalMap`中就会出现`key`为`null`的`Entry`，**就没有办法访问这些`key`为`null`的`Entry`的`value`**，如果当前线程再迟迟不结束的话，这些`key`为`null`的`Entry`的`value`就会一直存在一条强引用链：`Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value`永远无法回收，造成内存泄漏。



> 其实，`ThreadLocalMap`的设计中已经考虑到这种情况，也加上了一些防护措施：**在`ThreadLocal`的`get()`,`set()`,`remove()`的时候都会清除线程`ThreadLocalMap`里所有`key`为`null`的`value`。**



**例如**

`cleanSomeSlots(int i, int n)`方法通过遍历桶位，也会将 `key == null` 过期数据清理掉



## 为什么使用弱引用

- **key 使用强引用**：当**ThreadLocalMap**的key为强引用回收ThreadLocal时，因为ThreadLocalMap还持有ThreadLocal的强引用，如果没有手动删除，ThreadLocal不会被回收，导致Entry内存泄漏。 譬如 设置：**ThreadLocal=null** 以后，应该会被回收的，但实际情况是ThreadLocalMap还有一个强引用，导致无法回收
- **key 使用弱引用**：引用的`ThreadLocal`的对象被回收了，由于`ThreadLocalMap`持有`ThreadLocal`的弱引用，即使没有手动删除，`ThreadLocal`也会被回收。`value`在下一次`ThreadLocalMap`调用`set`,`get`，`remove`的时候会被清除。









在[Threadlocal](https://so.csdn.net/so/search?q=threadlocal&spm=1001.2101.3001.7020)的生命周期中,都存在这些引用. 看下图: 实线代表强引用,虚线代表弱引用.

![在这里插入图片描述](../../images/ThreadLocal/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyODYyODgy,size_16,color_FFFFFF,t_70.png)\









> 每个thread中都存在一个map, map的类型是ThreadLocal.ThreadLocalMap.
>
> Map中的key为一个threadlocal实例. 这个Map的确使用了弱引用,不过弱引用只是针对key.
>
> 每个key都弱引用指向threadlocal.
>
> 所以当把threadlocal实例置为null以后,没有任何强引用指向threadlocal实例,所以threadlocal就可以顺利被gc回收
>
> **注意！假如每个key都强引用指向threadlocal，也就是上图虚线那里是个强引用，那么这个threadlocal就会因为和entry存在强引用无法被回收！造成内存泄漏** ，除非线程结束，线程被回收了，map也跟着回收。
>
> https://www.cnblogs.com/onlywujun/p/3524675.html





## 强引用产生的内存泄漏



![image-20220822193442222](../../images/ThreadLocal/image-20220822193442222.png)



 当业务代码中Thread使用完ThreadLocal，threadLocalRef被回收。因ThreadLocalMap持有threadLocal Strong Ref,造成threadLocal无法被回收。但在没有手动删除Entry及CurrentThread依然运行的前提下，又存在强引用链threadRef->currentThread->threadLocalMap->entry。entry不会被回收了，产生内存泄漏。











## 弱引用产生的内存泄漏

![image-20220822193510541](../../images/ThreadLocal/image-20220822193510541.png)





 当业务代码中Thread使用完ThreadLocal，threadLocalRef被回收，因ThreadLocalMap仅仅持有threadLocal Weak Ref,没有任何引用指向threadlocal实例所以threadlocal可被gc回收，此时Entry中的key=null。但在没有手动删除Entry及CurrentThread依然运行的前提下，又存在强引用链threadRef->currentThread->threadLocalMap->entry->value。因entry的key是null,这块value永远不会被回收了，产生内存泄漏。

### 解决：

`value`在下一次`ThreadLocalMap`调用`set`,`get`，`remove`的时候会被清除key为null的entry。























**重点：**

**要知道，ThreadlocalMap是和线程绑定在一起的，如果这样线程没有被销毁，而我们又已经不会再某个threadlocal引用，那么key-value的键值对就会一直在map中存在，这对于程序来说，就出现了内存泄漏。**

**为了避免这种情况，只要将key设置为弱引用，那么当发生GC的时候，就会自动将弱引用给清理掉，也就是说：假如某个用户A执行方法时产生了一份threadlocalA，然后在很长一段时间都用不到threadlocalA时，作为弱引用，它会在下次垃圾回收时被清理掉。**



**而且ThreadLocalMap在内部的set，get和扩容时都会清理掉泄漏的Entry，内存泄漏完全没必要过于担心。**







https://csp1999.blog.csdn.net/article/details/117380154



**详解**：https://www.cnblogs.com/cy0628/p/15086201.html、







# ThreadLocal使用场景

## 全局存储用户信息



**在我们的SpringBoot应用中，通常代码都是分层的，接口层、服务层、持久层，大部分的接口请求中我们都需要使用当前登录的用户信息，比如用户名、用户编码、当前组织等信息。通常如果是使用session、header、token来存储当前登录信息，不管用哪种方式，都需要每一层代码传递下去，这样很麻烦，代码看起来也不够简洁。**



> 在现在的系统设计中，前后端分离已基本成为常态，分离之后如何获取用户信息就成了一件麻烦事，通常在用户登录后， 用户信息会保存在Session或者Token中。这个时候，我们如果使用常规的手段去获取用户信息会很费劲，拿Session来说，我们要在接口参数中加上HttpServletRequest对象，然后调用 getSession方法，且每一个需要用户信息的接口都要加上这个参数，才能获取Session，这样实现就很麻烦了。
>
> 在实际的系统设计中，我们肯定不会采用上面所说的这种方式，而是使用ThreadLocal，我们会选择在拦截器的业务中， 获取到保存的用户信息，然后存入ThreadLocal，那么当前线程在任何地方如果需要拿到用户信息都可以使用ThreadLocal的get()方法 (异步程序中ThreadLocal是不可靠的)
>
> 对于笔者而言，这个场景使用的比较多，当用户登录后，会将用户信息存入Token中返回前端，当用户调用需要授权的接口时，需要在header中携带 Token，然后拦截器中解析Token，获取用户信息，调用自定义的类(AuthNHolder)存入ThreadLocal中，当请求结束的时候，将ThreadLocal存储数据清空， 中间的过程无需在关注如何获取用户信息，只需要使用工具类的get方法即可。





多用于存取用户的token



web容器里面，每一个http请求都会开辟一个线程

这样每个线程都有自己对应的用户token



我们平常获取token只能在controller层里面，通过request获取token

但我们用threadlocal就可以在全局使用用户传过来的token







## 解决线程安全问题



在Spring的Web项目中，我们通常会将业务分为Controller层，Service层，Dao层， 我们都知道@Autowired注解默认使用单例模式，那么不同请求线程进来之后，由于Dao层使用单例，那么负责[数据库](https://cloud.tencent.com/solution/database?from=10680)连接的Connection也只有一个， 如果每个请求线程都去连接数据库，那么就会造成线程不安全的问题，Spring是如何解决这个问题的呢？

在Spring项目中Dao层中装配的Connection肯定是线程安全的，其解决方案就是采用ThreadLocal方法，当每个请求线程使用Connection的时候， 都会从ThreadLocal获取一次，如果为null，说明没有进行过数据库连接，连接后存入ThreadLocal中，如此一来，每一个请求线程都保存有一份 自己的Connection。于是便解决了线程安全问题

ThreadLocal在设计之初就是为解决并发问题而提供一种方案，每个线程维护一份自己的数据，达到线程隔离的效果。











## 代替参数的显式传递



## spring



**解决数据库连接**connection





数据库连接、Session管理





## 总结

[^知乎文章借鉴]: https://zhuanlan.zhihu.com/p/351750316

在我们的SpringBoot应用中，通常代码都是分层的，接口层、服务层、持久层，大部分的接口请求中我们都需要使用当前登录的用户信息，比如用户名、用户编码、当前组织等信息。通常如果是使用session、header、token来存储当前登录信息，不管用哪种方式，都需要每一层代码传递下去，这样很麻烦，代码看起来也不够简洁。



我们知道前前端发起的每个请求都会对应一个线程，我们只要在这一个线程里定义一个共享变量来存储当前登录信息，这样就可以方便的再任何地方取得当前登录信息了。java 为我们提供了一个ThreadLocal类，本地线程，其实它的对象是本地线程的局部变量。该变量为其所属线程所有，各个线程互不影响，我们可以将需要的数据保存到该对象中。**但是要注意线程结束要释放该对象中的数据，不然会出现内存泄露的问题**。





我们在拦截器中先进行Token 的验证，如果验证成功，就取出Token中的登录信息，这里你可以根据你的实际情况调用缓存、数据库查询等方式获取当前登录信息，然后保存到UserContext的ThreadLocal对象中。

然后在其他地方，需要用到登录信息的地方直接 调用即可，调用完必须清除



