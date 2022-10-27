---
title: HashMap
categories:
  - 集合
tags:
  - HashMap
abbrlink: 46153
date: 2022-04-14 19:54:20
cover : https://pic3.zhimg.com/v2-e883a05318653c2b5569ba41a08ca160_1440w.jpg?source=172ae18b
---



本篇写的是我们平常很常用的集合类HashMap，来根据它的源码来探究它的原理



[TOC]









HashMap有两个分隔时代，一个是jdk1.7一个

是jdk1.8





# HashMap数据结构

- **jdk1.7 数组+链表**
- **jdk1.8 数组+链表+红黑树**







## 重要成员变量

| DEFAULT_INITIAL_CAPACITY = 1 << 4; | Hash表默认初始容量                                 |
| ---------------------------------- | -------------------------------------------------- |
| MAXIMUM_CAPACITY = 1 << 30;        | 最大Hash表容量                                     |
| DEFAULT_LOAD_FACTOR = 0.75f        | 默认加载因子                                       |
| TREEIFY_THRESHOLD = 8；            | 链表转红黑树阈值                                   |
| UNTREEIFY_THRESHOLD = 6；          | 红黑树转链表阈值                                   |
| MIN_TREEIFY_CAPACITY = 64；        | 链表转红黑树时hash表最小容量阈值，达不到优先扩容。 |



补充：HashMap内部数组初始容量是16，不一定非得是16也可以自己设置大小，但一定要是2的指数。





**HashMap内部是一个Node类型的tables数组**



```java
transient Node<K,V>[] table;
```





**Node里面有key value hash next四个值**

- key:键 
- value:值 
- hash:经过hash()函数计算出来的哈希值 
- next:指向下一个Node结点



引言：大家都知道HashMap是通过哈希函数计算key值以及经过转换将元素放进数组的某个下标，以至于取的时候，也可以通过哈希函数来取出数组中对应的元素，来达到O(1)时间复杂度。



## hash（）

那HashMap中的哈希函数逻辑是什么呢？

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

> 上面就是HashMap中的哈希函数，是通过每个对象的hashcode值异或hashcode值右移16位
>
> hashCode右移16位，正好是32bit的一半。与自己本身做异或操作（相同为0，不同为1）。就是为了混合哈希值的高位和地位，增加低位的随机性。并且混合后的值也变相保持了高位的特征。



计算出来的hash值在经过下面运算就可以映射到数组的下标，在put函数中有具体运用到

```java
int index = (n - 1) & hash//哈希值 与上 数组长度减一
```



多个key值经过运算肯能映射到同一数组下标，但是多个key值的hash值可能不同，



同一个桶下面的链表节点或者树节点是hash值对数组长度取余结果相同，但不一定hash值相等，比如 3 % 5 == 8 % 5，结果都是3，但一个hash值是3一个是8，所以他们位于同一桶下但hash值不同，所以还需要再比较一下。



另外相同的key的hash值一定相同，但是key值不同hash值也可能相同









# 哈希冲突

就是不同的key经过哈希值相应的计算映射到了相同的数组下标。











![image-20220520114917983](../../images/HashMap/image-20220520114917983.png)





**开发定址法**

**所谓开放定址法，即是由关键码得到的哈希地址一旦产生了冲突，也就是说，该地址已经存放了数据元素，就去寻找下一个空的哈希地址，只要哈希表足够大，空的哈希地址总能找到，并将数据元素存入。**

`线性探测`、`平方探测`、



**线性探测**

以增量序列 1，2，……（TableSize -1）循环试探下一个存储地址。当碰撞发生时，我们直接检查散列表中的下一个位置（将索引值加1），如果不同则继续查找，直到找到该键或遇到一个空元素。

**di 为增量序列1，2，……，m-1，且di=i**

> 例如在长度为11的哈希表中，已经填入关键字分别为17,60，29的记录（哈希函数为H(key) MOD 11),现有第四个记录，其关键字为38，由哈希函数得到的哈希地址为5，和关键字为60的哈希地址产生冲突；若用线性探测再散列的方法处理时，得到下一个地址为6，仍冲突；再求下一个地址7，仍然冲突；直到哈希地址为8的位置时，得到一个为“空”的位置，处理冲突的过程结束，并将38插入到哈希表中第8号位置。





**二次探测**

**d = 1^2, -1^2, 2^2, -2^2, 3^2,**





# 头插（1.7）尾插（1.8）

**1.7是头插入，1.8是尾插入，解决了死循环问题。**

当采用头插法时会容易出现逆序且环形链表死循环问题。



为什么要用红黑树：

因为链表查询的时候链表过长了查询效率非常低，所以需要红黑树 。







 

# 加载因子0.75

为什么加载因子是0.75，不是其他呢

取了一个平衡

牛顿二项式：基于空间与世界的折中考虑



没有太大，不至于发生太多哈希冲突导致查询效率降低

没有太小，不至于空间没有利用完全



> 最近在看HashMap源码，对于扩容因子=0.75感到很费解，为什么在用了75%的容量的时候就要进行扩容呢？数组中明明还有25%的空间没有使用。为什么不等到数组几乎满了（扩容因子=0.95）的时候才进行扩容？扩容因子=0.95和扩容因子=0.75有什么区别吗？
> 首先来看一下什么是扩容因子。假设hash函数是理想的，数据会通过hash函数均匀的映射到数组上。一个数据映射到每一个桶（bucket）的概率是相等的。那么在任意的数组容量下，put一个数据发生碰撞的概率=数 组 中 元 素 的 个 数 数 组 容 量 \frac{数组中元素的个数}{数组容量} 
> 数组容量
> 数组中元素的个数
>
>  。而数组的扩容门槛threshold = capacity * loadFactorloadFactor。也就是说扩容因子就是HashMap在扩容门槛的状态下，put操作发生碰撞的概率。
> 那么，扩容因子等于0.75还是0.95的区别就很明显了。扩容因子=0.75。当使用量接近数组容量的75%的时候，数组中还有25%的剩余空间。平均来看，就是每4个桶（bucket）中还有一个是空的，当我们向map中put数据的时候，发生碰撞的概率是75%。因为这25%的空闲空间的存在，发生hash碰撞的概率还处在一个可以接受的范围内。
> 而当扩容因子=0.95的时候，平均来看，就是每20个桶（bucket）中才有一个是空的，此时数组中几乎没有空闲的桶（bucket），当我们put数据的时候，碰撞的概率是95%，几乎可以认为会发生碰撞。
> 除此之外，碰撞的概率越大，put的元素就越多，平均到每个桶中的元素的数量也越多。一旦发生碰撞，需要付出更大的代价。所以，如果扩容因子越大，碰撞的概率也就越大，发生碰撞后的代价也更大，结果导致效率大打折扣。
> 因此扩容因子=0.75也是一个空间换时间的考虑，0.75这个数值应该是经过充分的考虑决定的。



选择0.75作为默认的加载因子，完全是时间和空间成本上寻求的一种折衷选择。





额外：加载因子0.75是指，当hashmap中的元素个数超过数组大小*loadFactor（0.75）时候就会扩容。





# 链表转红黑树treeifyBin内部达不到阈值优先扩容



```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();//优先扩容, 默认阈值为64
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```



**在put方法里面的链表转红黑树逻辑treeifyBin中，如果此时数组的长度小于阈值时候，就会优先扩容，其次才是转红黑树逻辑**





**所以转红黑树必须链表长度大于8并且数组的长度大于64**













# put()



```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```



具体逻辑在下面这个函数，看看put函数是怎样将键值对插入HashMap的

hash:要插入的键值对的key计算出来的hash值 

key : 键

value:键值对的值



```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    if ((tab = table) == null || (n = tab.length) == 0)//table数组还没有初始化，先初始化
        n = (tab = resize()).length;//调用扩容函数初始化
    if ((p = tab[i = (n - 1) & hash]) == null)//先看看这个映射的下标有没有值，如果没有就直接插入，如果有代表哈希冲突了
        tab[i] = newNode(hash, key, value, null);
    else {//else 代表出现了Hash冲突
        Node<K,V> e; K k;//e是要后面要遍历链表的结点
        
        /** 当前桶中的第一个结点的hash值和将要存入的key的hash值相等，且key是相等的------地址相等或equals方法执行相等 **/
        if ( 第一步  p.hash == hash &&//p是这个下标上的第一个结点
                  	        d           ((k = p.key) == key || (key != null && key.equals(k))))
                   //将当前桶中该位置元素（第一个结点）保存下来 
            e = p;
        else if (p instanceof TreeNode)//查看当前桶中元素（第一个结点）为红黑树节点时，使用红黑树保存将要插入的元素 
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {//遍历链表
                if ((e = p.next) == null) {//链表上没有相同的key那再链表后面插入新结点
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // 大于8，转成红黑树
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                /**
                 ①先判断就算出来的hash值是否相同，如果是相等的那进行②
                 ②hash值相同，两个key不一定相等，因为hashcode()可能重写，这样就有可能不同的key出现相同的hash值，就需要用 
                 == 和equals（）比较，如果是相等的覆盖原值、
                 为什么先判断hash值是否相等再判断的key: 性能
                 hash不相等的，一定不是同一个key，hash相等的，不一定是同一个key，key相同的,hash值一定相等。
               
                **/
                
                p = e;
            }
        }
        if (e != null) { // 更新相同key对应的value
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)//大于阀值就扩容 size是所有结点数
        resize();//扩容
    afterNodeInsertion(evict);
    return null;
}
```



## put逻辑（重点）



```java
下面为put方法的主逻辑
    
     if() { 
         第一个if
             判断当前table数组是否厨初始化，如果没有初始化的话，调用resize()进行扩容初始化
     }
	
     if() {
         第二个if
             判断根据当前传的hash值映射到数组下标是否有值，如果没有值直接插入，如果有就代表哈希冲突了
     } else{
         else -- 到这一步说明出现哈希冲突了,下面是处理哈希冲突的逻辑
             
           if()  {
               第一个if
                   判断当前哈希桶中第一个元素的哈希值是否与传来的hash值相同，并且当前结点的key和传来的key相同（用== 或者 equals判断）
           } else if() {
               第二个else_if
                   判断当前结点是不是红黑树结点，如果是的话，调用红黑树函数保存元素
           } else {
               到这一步else说明哈希桶第一个元素不是要插入的元素而且此时还没有转成红黑树，那就开始for循环比遍历
                   for(int binCount = 0; ; ++binCount) {
                       if() {
                           到这一步说明链表没有相同的结点相匹配，那就插入到链表结尾
                           
                           插入完毕后要判断链表结点是否大于等于8，如果是那就调用函数进行链表转红黑树
                           
                       }
                       if() {    老样子，判断当前遍历到的结点的hash值是否与传来的hash值相同，然后判断key(==或者equals)
                     		符合之后，保存当前遍历到的结点并且break结束循环
                       }
                   }
               
           }
           
         
         
         
     }		
```









































## put总结图



![img](../../images/HashMap/4171)





# resize()

## jdk1.7扩容

https://blog.csdn.net/wildyuhao/article/details/108182478

```java
//扩容是2倍扩容,是因为HashMap的算下角标是与运算,是hashcode与2的幂-1与运算,效率比取模运算快几倍以上
void resize(int newCapacity) {
    // 判断是否需要扩容。
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }
    // 创建扩容后的新数组，并且在扩容后替换老数组。
    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}
```



### 死锁问题

HashMap是线程不安全的，不安全的具体原因就是在高并发场景下，扩容可能产生死锁(Jdk1.7存在)以及get操作可能带来的数据丢失。

死锁问题核心在于下面代码，多线程扩容导致形成的链表环!

```java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;//第一行
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);//第二行
            e.next = newTable[i];//第三行
            newTable[i] = e;//第四行
            e = next;//第五行
        }
    }
}
```

去掉了一些冗余的代码， 层次结构更加清晰了。

- 第一行：记录oldhash表中e.next
- 第二行：rehash计算出数组的位置(hash表中桶的位置)
- 第三行：e要插入链表的头部， 所以要先将e.next指向new hash表中的第一个元素
- 第四行：将e放入到new hash表的头部
- 第五行： 转移e到下一个节点， 继续循环下去

**单线程扩容**

**假设：**hash算法就是简单的key与length(数组长度)求余。hash表长度为2，如果不扩容， 那么元素key为3,5,7按照计算(key%table.length)的话都应该碰撞到table[1]上。

**扩容：**hash表长度会扩容为4重新hash，key=3 会落到table[3]上(3%4=3)， 当前e.next为key(7), 继续while循环重新hash，key=7 会落到table[3]上(7%4=3), 产生碰撞， 这里采用的是头插入法，所以key=7的Entry会排在key=3前面(这里可以具体看while语句中代码)当前e.next为key(5), 继续while循环重新hash，key=5 会落到table[1]上(5%4=3)， 当前e.next为null, 跳出while循环，resize结束。

如下图所示

​    ![0](../../images/HashMap/4136)

**多线程扩容**

下面就是多线程同时put的情况了， 然后同时进入transfer方法中：假设这里有两个线程同时执行了put()操作，并进入了transfer()环节

​                while(null != e) {      Entry<K,V> next = e.next;//第一行，线程1执行到此被调度挂起      int i = indexFor(e.hash, newCapacity);//第二行      e.next = newTable[i];//第三行      newTable[i] = e;//第四行      e = next;//第五行 }              

那么此时状态为：

​    ![0](../../images/HashMap/4148)

从上面的图我们可以看到，因为线程1的 e 指向了 key(3)，而 next 指向了 key(7)，在线程2 rehash 后，就指向了线程2 rehash 后的链表。

然后线程1被唤醒了：

1. 执行e.next = newTable[i]，于是 key(3)的 next 指向了线程1的新 Hash 表，因为新 Hash 表为空，所以e.next = null，
2. 执行newTable[i] = e，所以线程1的新 Hash 表第一个元素指向了线程2新 Hash 表的 key(3)。好了，e 处理完毕。
3. 执行e = next，将 e 指向 next，所以新的 e 是 key(7)

然后该执行 key(3)的 next 节点 key(7)了:

1. 现在的 e 节点是 key(7)，首先执行Entry next = e.next,那么 next 就是 key(3)了
2. 执行e.next = newTable[i]，于是key(7) 的 next 就成了 key(3)
3. 执行newTable[i] = e，那么线程1的新 Hash 表第一个元素变成了 key(7)
4. 执行e = next，将 e 指向 next，所以新的 e 是 key(3)

此时状态为：

​    ![0](../../images/HashMap/4153)

然后又该执行 key(7)的 next 节点 key(3)了：

1. 现在的 e 节点是 key(3)，首先执行Entry next = e.next,那么 next 就是 null
2. 执行e.next = newTable[i]，于是key(3) 的 next 就成了 key(7)
3. 执行newTable[i] = e，那么线程1的新 Hash 表第一个元素变成了 key(3)
4. 执行e = next，将 e 指向 next，所以新的 e 是 key(7)

这时候的状态如图所示：

​    ![0](../../images/HashMap/4156)

很明显，环形链表出现了。









## jdk1.8扩容

**前提：当hashmap中的元素个数超过数组大 小*loadFactor（0.75）时候就会自动扩容。**



**Java8 HashMap扩容跳过了Jdk7扩容的坑，对源码进行了优化，采用高低位拆分转移方式，并且采用尾插入，避免了链表环的产生**





**扩容前：**

![img](../../images/HashMap/16193)

**扩容后：**

​    ![0](../../images/HashMap/4117)



```java
        final Node<K, V>[] resize () {
            Node<K, V>[] oldTab = table;
            //记住扩容前的数组长度和最大容量
            int oldCap = (oldTab == null) ? 0 : oldTab.length;
            int oldThr = threshold;
            int newCap, newThr = 0;
            if (oldCap > 0) {
                //超过数组在java中最大容量就无能为力了，冲突就只能冲突
                if (oldCap >= MAXIMUM_CAPACITY) {
                    threshold = Integer.MAX_VALUE;
                    return oldTab;
                }
                //长度和最大容量都扩容为原来的二倍 
                else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                        oldCap >= DEFAULT_INITIAL_CAPACITY)
                    newThr = oldThr << 1;
                double threshold
            }
            //...... ......
            //更新新的最大容量为扩容计算后的最大容量
            threshold = newThr;
            //更新扩容后的新数组长度
            Node<K, V>[] newTab = (Node<K, V>[]) new Node[newCap];
            table = newTab;
            if (oldTab != null) {
                //遍历老数组下标索引
                for (int j = 0; j < oldCap; ++j) {
                    Node<K, V> e;
                    //如果老数组对应索引上有元素则取出链表头元素放在e中
                    if ((e = oldTab[j]) != null) {
                        oldTab[j] = null;
                        //如果老数组j下标处只有一个元素则直接计算新数组中位置放置
                        if (e.next == null)
                            newTab[e.hash & (newCap - 1)] = e;
                        else if (e instanceof TreeNode)
                            //如果是树结构进行单独处理
                            ((TreeNode<K, V>) e).split(this, newTab, j, oldCap);
                        else {
                            preserve order
                            //能进来说明数组索引j位置上存在哈希冲突的链表结构
                            Node<K, V> loHead = null, loTail = null;
                            Node<K, V> hiHead = null, hiTail = null;
                            Node<K, V> next;
                            //循环处理数组索引j位置上哈希冲突的链表中每个元素
                            do {
                                next = e.next;
                                //判断key的hash值与老数组长度与操作后结果决定元素是放在原索引处还是新索引
                                if ((e.hash & oldCap) == 0) {
                                    //放在原索引处的建立新链表
                                    if (loTail == null) loHead = e;
                                    else loTail.next = e;
                                    loTail = e;
                                } else {
                                    //放在新索引（原索引 + oldCap）处的建立新链表
                                    if (hiTail == null) hiHead = e;
                                    else hiTail.next = e;
                                    hiTail = e;
                                }
                            }
                            while ((e = next) != null);
                            if (loTail != null) {
                                //放入原索引处 loTail.next = null;
                                newTab[j] = loHead;
                            }
                            if (hiTail != null) {
                                //放入新索引处
                                hiTail.next = null;
                                newTab[j + oldCap] = hiHead;
                            }
                        }
                    }
                }
            }
            return newTab;
        }
```









**在扩容时，不需要重新计算元素的hash了，只需要判断最高位是1还是0就好了**。







**扩容先将数组长度扩容为原来的2倍**



把扩容前每个哈希桶中的结点分为两种，一种是依旧在之前那个桶对应的下标的桶中，另一种是之前所在的桶的下标+oldCap 



**分开的条件是该结点的K的hash值与扩容之前的总桶数n做了一个&运算（以前做映射的时候的公式为hash&（n-1）n为总桶数注意两个公式区别）（并且hash & n只有两种结果 0 或者1 ，0为低位，1为高位 进行拆分）**



为什么要使用这个条件分为两个链表，主要是判断出扩容之后哪些结点依旧在之前那个桶对应的下标的桶中，哪些结点在之前所在的桶的下标+oldCap的桶中原理






**经过rehash之后，元素的位置要么是在原位置，要么是在原位置加原数组长度的位置。**



### 1.8死锁问题

**必看**：https://blog.csdn.net/Liu_JM/article/details/105582711



**1.8采用尾插避免了1.7多线程扩容死锁问题，但是1.8还是会有线程安全问题，并发put的时候会有数据覆盖**



所以jdk1.8的HashMap在多线程的情况下也会出现死循环的问题，但是1.8是在链表转换树或者对树进行操作的时候会出现线程安全的问题。





> 在JDK1.8之后，HashMap底层的数组扩容后迁移的方法进行了优化。把一个链表分成了两组，分成高为和低位分别去迁移，避免了死环问题。而且在迁移的过程中并没有进行任何的rehash（重新记算hash），提高了性能。它是直接将链表给断掉，进行几乎是一个均等的拆分，然后通过头指针的指向将整体给迁移过去，这样就减小了链表的长度。
>
> 





# get()



```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;//
}

/**
 * Implements Map.get and related methods.
 *
 * @param hash hash for key
 * @param key the key
 * @return the node, or null if none
 */
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))//主要逻辑，先比较hash值，如果不同则直接直接下一个Node结点，如果相同，则 == 或者equals比较是否相同
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```





# 专题

## 为什么数组长度要是2的N次幂



![image-20220520114325177](../../images/HashMap/image-20220520114325177.png)















**提高运算速度**

- **(n - 1) & hash，当n为2次幂时，会满足一个公式：(n - 1) & hash = hash % n**

- **&运算速度快，至少比%取模运算块**





## 为什么是8



**为什么链表长度大于8时候就转红黑树**

HashMap在jdk1.8之后引入了红黑树的概念，表示若桶中链表元素超过8时，会自动转化成红黑树；若桶中元素小于等于6时，树结构还原成链表形式。

原因：

　　红黑树的平均查找长度是log(n)，长度为8，查找长度为log(8)=3，链表的平均查找长度为n/2，当长度为8时，平均查找长度为8/2=4，这才有转换成树的必要；链表长度如果是小于等于6，6/2=3，虽然速度也很快的，但是转化为树结构和生成树的时间并不会太短。





在理想情况下，链表长度符合泊松分布，各个长度的命中概率依次递减，注释中给我们展示了1-8长度的具体命中概率，当长度为8的时候，概率概率仅为0.00000006，这么小的概率，HashMap的红黑树转换几乎不会发生

## 为什么是6



 红黑树为什么桶中结点少于6会退化成链表



主要是一个过渡，避免链表和红黑树之间频繁的转换。如果阈值是7的话，删除一个元素红黑树就必须退化为链表，增加一个元素就必须树化，来回不断的转换结构无疑会降低性能，所以阈值才不设置的那么临界。







还有选择6和8的原因是：

　　中间有个差值7可以防止链表和树之间频繁的转换。假设一下，如果设计成链表个数超过8则链表转换成树结构，链表个数小于8则树结构转换成链表，如果一个HashMap不停的插入、删除元素，链表个数在8左右徘徊，就会频繁的发生树转链表、链表转树，效率会很低



# 手写简易版HashMap



## map接口

```java
public interface Map<K,V> {
    V put(K k, V v);

    V get(K k);

    int size();

    V remove(K k);

    boolean isEmpty();

    void clear();
}
```

## Node结点

```java
class Node<K,V> {
    K k;
    V v;
    Node<K,V> next;
    public Node(K k,V v, Node<K,V> next) {
        this.k = k;
        this.v = v;
        this.next = next;
    }
}
```



## MyHashMap

```java
public class MyHashMap<K,V> implements Map<K,V> {


    //默认初始化容量
    final static int DEFAULT_CAPACITY = 16;
    //默认加载因子
    final static float DEFAULT_LOAD_FACTOR = 0.75f;

    //容量
    int capacity;
    //加载因子
    float loadFactor;

    //数组
    Node<K,V>[] table;

    //哈希表的所有结点
    int size;

    public MyHashMap(int capacity, float loadFactor) {
        this.capacity = capacity;
        this.loadFactor = loadFactor;
    }


    public MyHashMap() {
        this(DEFAULT_CAPACITY, DEFAULT_LOAD_FACTOR);
    }


    @Override
    public V put(K k, V v) {


        int index = k.hashCode() % table.length;
        Node<K,V> current = table[index];

        if(current == null) {
            //如果table[index] 为null,直接赋值
            table[index] = new Node<K,V>(k,v,null);
            size++;
        }else {
            //遍历链表
            while (current != null) {
                if(current.k == k) {
                   V oldValue = current.v;
                   //新value覆盖旧value
                   current.v = v;
                   return oldValue;
                }
                current = current.next;
            }

            //如果链表没有相同的key就头插
            //右边的table[index] 是旧的第一个结点，当前结点插入这个旧的结点前面
            table[index] = new Node<K,V>(k,v,table[index]);
            size++;
            return null;
         }
        return null;
    }

    @Override
    public V get(K k) {
        int index = k.hashCode() % table.length;

        Node<K,V> current = table[index];

       while (current != null) {
           if(current.k == k) {
               return current.v;
           }
           current = current.next;
       }
        return null;
    }

    @Override
    public int size() {
        return size;
    }

    @Override
    public V remove(K k) {
        int index = k.hashCode() % table.length;
        Node<K,V> current = table[index];
         //如果直接匹配第一个结点
        if(current.k == k && current.next == null) {
            table[index] = null;
            size--;
            return current.v;
        }

        //在链表中删除结点
        while (current.next != null) {
            if(current.next.k == k) {
                V oldValue = current.next.v;
                current.next = current.next.next;
                size--;
                return oldValue;
            }
            current = current.next;
        }
        return null;
    }

    @Override
    public boolean isEmpty() {
        return size == 0;
    }

    @Override
    public void clear() {

    }
}
```





























# 什么时候给HashMap分配容量















# resize扩容操作























 
