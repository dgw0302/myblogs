---
title: KafKa
categories:
  - 中间件
tags:
  - KafKa
abbrlink: 56288
date: 2022-07-23 19:54:20
cover : https://www.javazhiyin.com/wp-content/uploads/2019/02/java2-1550467241.jpeg
---









# Kafka是什么



Kafka 是一个分布式的流式处理平台，用于实时构建流处理应用。主要应用在大数据实时处理领域。Kafka 凭借「**高性能**」、「**高吞吐**」、「**高可用**」、「**低延迟**」、「**可伸缩**」几大特性，成为「**消息队列」**的首选。



> 
>
> 1）**高性能：**以时间复杂度为 O(1) 的方式提供消息持久化能力，即使对TB级以上数据也能保证常数时间的访问性能。
>
> 2）**高吞吐、低延迟：**在非常廉价的机器上也能做到单机支持每秒几十万条消息的传输，并保持毫秒级延迟。
>
> 3）**持久性、可靠性：**消息最终被持久化到磁盘，且提供数据备份机制防止数据丢失。
>
> 4）**容错性：**支持集群节点故障容灾恢复，即使 Kafka 集群中的某一台 Kafka 服务节点宕机，也不会影响整个系统的功能（若副本数量为N， 则允许N-1台节点故障）。
>
> 5）**高并发：**可以支撑数千个客户端同时进行读写操作。



## 适用场景



![image-20220722210449182](../../images/Kafka/image-20220722210449182.png)



1）**日志收集方向：**可以用 Kafka 来收集各种服务的 log，然后统一输出，比如日志系统 elk，用 Kafka 进行数据中转。

2）**消息系统方向：**Kafka 具备系统解耦、副本冗余、流量削峰、消息缓冲、可伸缩性、容错性等功能，同时还提供了消息顺序性保障以及消息回溯功能等。

3）**大数据实时计算方向：**Kafka 提供了一套完整的流式处理框架， 被广泛应用到大数据处理，如与 flink、spark、storm 等整合。









# Kafka 核心组件





![image-20220722210819841](../../images/Kafka/image-20220722210819841.png)









> 1）**Producer：**即消息生产者，向 Kafka Broker 发消息的客户端。
>
> 2）**Consumer：**即消息消费者，从 Kafka Broker 读消息的客户端。
>
> 3）**Consumer Group：**即消费者组，**由多个 Consumer 组成**。消费者组内每个消费者负责消费不同分区的数据，以提高消费能力。**一个分区只能由组内一个消费者消费，不同消费者组之间互不影响**。
>
> 4）**Broker：****一台 Kafka 服务节点就是一个 Broker**。一个集群是由1个或者多个 Broker 组成的，且一个 Broker 可以容纳多个 Topic。
>
> 5）**Topic：**一个逻辑上的概念，Topic 将消息分类，生产者和消费者面向的都是同一个 Topic, **同一个 Topic 下的 Partition 的消息内容是不相同的**。
>
> 6）**Partition：**为了实现 Topic 扩展性，提高并发能力，一个非常大的 Topic 可以分布到多个 Broker 上，**一个 Topic 可以分为多个 Partition 进行存储，且每个 Partition 是消息内容是有序的**。
>
> 7）**Replica：**即副本，为实现**数据备份**的功能，保证集群中的某个节点发生故障时，该节点上的 Partition 数据不丢失，且 Kafka 仍然能够继续工作，为此 Kafka 提供了副本机制，**一个 Topic 的每个 Partition 都有若干个副本，一个 Leader 副本和若干个 Follower 副本**。
>
> 8）**Leader：**即每个分区多个副本的主副本，**生产者发送数据的对象，以及消费者消费数据的对象，都是 Leader**。
>
> 9）**Follower：**即每个分区多个副本的从副本，**会实时从 Leader 副本中同步数据，并保持和 Leader 数据的同步**。Leader 发生故障时，某个 Follower 还会被选举并成为新的 Leader , 且不能跟 Leader 在同一个 Broker 上, 防止崩溃数据可恢复。
>
> 10）**Offset：**消费者消费的位置信息，**监控数据消费到什么位置**，当消费者挂掉再重新恢复的时候，可以从消费位置继续消费。







## Zookeeper 



Kafka 集群能够正常工作，**目前还是**需要依赖于 ZooKeeper，主要用来「**负责 Kafka集群元数据管理，集群协调工作**」，在每个 Kafka 服务器启动的时候去连接并将自己注册到 Zookeeper，类似注册中心。



Kafka 使用 Zookeeper 存放「**集群元数据**」、「**集群**成员管理**」、 「**Controller 选举**」、「**其他管理类任务」等。待 KRaft 提案完成后，Kafka 将完全不依赖 Zookeeper。





![image-20220722210944615](../../images/Kafka/image-20220722210944615.png)





Kafka 3.X 「**2.8版本开始**」为什么移除 Zookeeper 的依赖的原因有以下2点：



1）**集群运维层面**：Kafka 本身就是一个分布式系统，如果还需要重度依赖 Zookeeper，集群运维成本和系统复杂度都很高。

2）**集群性能层面：Zookeeper 架构设计并不适合这种高频的读写更新操作, 由于之前的提交位移的操作都是保存在 Zookeeper 里面的，这样的话会严重影响 Zookeeper 集群的性能。





## 副本



在 Kafka 中，为实现「**数据备份**」的功能，保证集群中的某个节点发生故障时，该节点上的 Partition 数据不丢失，且 Kafka 仍然能够继续工作，**为此 Kafka 提供了副本机制，**一个 Topic 的每个 Partition 都有若干个副本，一个 Leader 副本和若干个 Follower 副本。



![image-20220722213823758](../../images/Kafka/image-20220722213823758.png)





1）Leader 主副本负责对外提供读写数据服务。

2）Follower 从副本只负责和 Leader 副本保持数据同步，并不对外提供任何服务



**同一个分区下的所有副本保存相同的消息数据，这些副本分散保存在不同的 Broker 上，保证了 Broker 的整体可用性**。



### **副本同步机制**

既然所有副本的消息内容相同，我们该如何保证副本中所有的数据都是一致的呢？当 Producer 发送消息到某个 Topic 后，消息是如何同步到对应的所有副本 Replica 中的呢？Kafka 中 只有 Leader 副本才能对外进行读写服务，所以解决同步问题，Kafka 是采用基于 **Leader** 的副本机制来完成的。



> 1）在 Kafka 中，一个 Topic 的每个 Partition 都有若干个副本，**副本分成两类：领导者副本「Leader Replica」和追随者副本「Follower Replica」**。每个分区在创建时都要选举一个副本作为领导者副本，其余的副本作为追随者副本。
>
> 2）在 Kafka 中，Follower 副本是不对外提供服务的。也就是说，任何一个Follower 副本都不能响应客户端的读写请求。**所有的读写请求都必须先发往 Leader 副本所在的 Broker，由该 Broker 负责处理。Follower 副本不处理客户端请求，它唯一的任务就是从 Leader 副本异步拉取消息，并写入到自己的提交日志中，从而实现与 Leader 副本的同步**。
>
> 3）在 Kafka 2.X 版本中，当 Leader 副本 所在的 Broker 宕机时，ZooKeeper 提供的监控功能能够实时感知到，并立即开启新一轮的 Leader 选举，从 ISR 副本集合中 Follower 副本中选一个作为新的 Leader ，当旧的 Leader 副本重启回来后，只能作为 Follower 副本加入到集群中。3.x 的选举后续会有单篇去介绍。





### 副本管理（重点）

![image-20220723105838254](../../images/Kafka/image-20220723105838254.png)





> 1）**AR 副本集合:** 分区 Partition 中的所有 Replica（副本） 组成 AR 副本集合。
>
> 2）**ISR 副本集合:** 所有与 Leader 副本能保持一定程度同步的 Replica 组成 ISR 副本集合， 其中也包括 Leader 副本。
>
> 3）**OSR 副本集合:** 与 Leader 副本同步滞后过多的 Replica 组成 OSR 副本集合。







### **ISR 副本集合**



  **Kafka会在Zookeeper上针对每个Topic维护一个称为ISR（in-sync replica，已同步的副本）的集合，该集合中是一些分区的副本。只有当这些副本都跟Leader中的副本同步了之后，kafka才会认为消息已提交，并反馈给消息的生产者。如果这个集合有增减，kafka会更新zookeeper上的记录。**

 **如果某个分区的Leader不可用，Kafka就会从ISR集合中选择一个副本作为新的Leader。**

上面强调过，Follower 副本不提供服务，只是定期地异步拉取 Leader 副本中的数据。既然是异步的，就一定会存在不能与 Leader 实时同步的情况出现。



Kafka 为了解决这个问题， 引入了「 **In-sync Replicas**」机制，即 ISR 副本集合。要求 ISR 副本集合中的 Follower 副本都是与 Leader 同步的副本。



**那么，到底什么样的副本能够进入到 ISR 副本集合中呢？**



首先要明确的，Leader 副本天然就在 ISR 副本集合中。也就是说，ISR 不只是有 Follower 副本集合，它必然包括 Leader 副本。另外，能够进入到 ISR 副本集合的 Follower 副本要满足一定的条件。

![图片](../../images/Kafka/640.png)



图中有 3 个副本：1 个 Leader 副本和 2 个 Follower 副本。Leader 副本当前写入了 6 条消息，Follower1 副本同步了其中的 4 条消息，而 Follower2 副本只同步了其中的 3 条消息。那么，对于这 2 个 Follower 副本，你觉得哪个 Follower 副本与 Leader 不同步？



事实上，这2个 Follower 副本都有可能与 Leader 副本同步，但也可能不与 Leader 副本同步，这个完全依赖于 Broker 端参数 replica.lag.time.max.ms 参数值。

这个参数是指 **Follower 副本能够落后 Leader 副本的最长时间间隔，当前默认值是 10 秒，从 2.5 版本开始，**默认值从 10 秒增加到 30 秒**。即只要一个 Follower 副本落后Leader 副本的时间不连续超过 30 秒，Kafka 就认为该 Follower 副本与 Leader 是同步的，即使 Follower 副本中保存的消息明显少于 Leader 副本中的消息。



此时如果这个副本同步过程的速度持续慢于 Leader 副本的消息写入速度的时候，那么在 replica.lag.time.max.ms 时间后，该 Follower 副本就会被认为与 Leader 副本是不同步的，因此 Kafka 会自动收缩，将其踢出 ISR 副本集合中。后续如果该副本追上了 Leader 副本的进度的话，那么它是能够重新被加回 ISR副本集合的。

在默认情况下，当 Leader 副本发生故障时，只有在 ISR 副本集合中的 Follower 副本才有资格被选举为新Leader，而 OSR 中副本集合的副本是没有机会的（可以通过**unclean.leader.election.enable** 进行配置执行脏选举）。



**总结：ISR 副本集合是一个动态调整的集合。**









## 消费者组







消费者组 Consumer Group，顾名思义就是由多个 Consumer 组成，且拥有一个公共且唯一的 Group ID。组内每个消费者负责消费不同分区的数据，**但一个分区只能由一个组内消费者消费，**消费者组之间互不影响。



**为什么 Kafka 要设计 Consumer Group,  只有 Consumer 不可以吗？** 



我们知道 Kafka 是一款高吞吐量，低延迟，高并发, 高可扩展的消息队列产品， 那么如果某个 Topic 拥有数百万到数千万的数据量， 仅仅依靠 Consumer 进程消费， 消费速度可想而知， **所以需要一个扩展性较好的机制来保障消费进度， 这个时候 Consumer Group 应运而生， Consumer Group 是 Kafka 提供的可扩展且具有容错性的消费者机制**。





## 网络模型



**Kafka 采用多路复用方案，Reactor 设计模式，并引用 Java NIO 的方式更好的解决网络超高并发请求问题。**









# Kafka选举

这里所说的 Leader 选举是指分区 Leader 副本的选举，**它是由 Kafka Controller 负责具体执行的，当创建分区或分区副本上线的时候都需要执行 Leader 的选举动作**。







## **控制器的选举**

文章：https://mp.weixin.qq.com/s/Ny_VRCotJNE_4ZRLlMGSzg





所谓的控制器「**Controller**」就是通过 ZooKeeper 来管理和协调整个 Kafka 集群的组件。集群中任意一台 Broker 都可以充当控制器的角色，但是在正常运行过程中，只能有一个 Broker 成为控制器。



**控制器的职责主要包括：**

> 1）集群元信息管理及更新同步 (Topic路由信息等)。
>
> 2）主题管理（创建、删除、增加分区等）。
>
> 3）分区重新分配。
>
> 4）副本故障转移、 Leader 选举、ISR 变更。
>
> 5）集群成员管理（通过 watch 机制自动检测新增 Broker、Broker 主动关闭、Broker 宕机等）。



 

>
> 1）Kafka 使用 Zookeeper 的临时节点来选举 Controller。
>
> 2）Zookeeper 在 Broker 加入集群或退出集群时通知 Controller。
>
> 3）Controller 负责在 Broker 加入或离开集群时进行分区 Leader 选举。









每个broker还是会对/controller节点添加监听器,以此来监昕此节点的数据变化，当/controller节点发生变更，就会触发新一轮的选举。







**其实选举最终都是通过调用底层的 elect 方法进行选举，如下图所示：**



![image-20220723111322762](../../images/Kafka/image-20220723111322762.png)







## **分区Leader的选举**



分区leader副本的选举由控制器负责具体实施。当创建分区（创建主题或增加分区都有创建分区的动作），或分区上线（比如分区中原先的leader 副本下线，此时分区需要选举一个新的leader上线来对外提供服务）的时候，都需要执行leader的选举动作**。基本策略是按照AR集合中副本的顺序查找第一个存活的副本，并且这个副本在ISR集合中 。**一个分区的AR集合在分配的时候就被指定,并且只要不发生重分配的情况，集合内部副本的顺序是保持不变的，而分区的ISR集合中副本的顺序可能会改变。









## **消费组Leader的选举**



GroupCoordinator需要为消费组内的消费者选举出一个消费组的leader，这个选举的算法很简单，当消费组内还没有leader，那么第一个加入消费组的消费者即为消费组的leader，如果当前leader退出消费组，则会挑选以HashMap结构保存的消费者节点数据中，第一个键值对来作为leader。









# 生产者











## 发送消息



Kafka 生产者发送消息主要有三种模式:

![image-20220722211156375](../../images/Kafka/image-20220722211156375.png)







### **发后即忘发送模式**

**发后即忘模式**「fire-and-forget」，它只管发送消息，并不需要关心消息是否发送成功**。其本质上也是一种**异步发送**的方式，消息先存储在缓冲区中，达到设定条件后再批量进行发送。**这是 kafka 吞吐量最高的方式，但同时也是消息最不可靠的方式，因为对于发送失败的消息并没有做任何处理，某些异常情况下会导致消息丢失。



```java
ProducerRecord<k,v> record = new ProducerRecord<k,v>("this-topic", key, value);
try {
  //fire-and-forget 模式   
  producer.send(record);
} catch (Exception e) {
  e.printStackTrace();
}
```





### **同步发送模式**

**同步发送模式 「sync」**，调用 send() 方法会返回一个 Future 对象，再通过调用 Future 对象的 get() 方法，等待结果返回，根据返回的结果可以判断消息是否发送成功， **由于是同步发送会阻塞，只有当消息通过 get() 返回数据时，才会继续下一条消息的发送**。



```java
ProducerRecord<k,v> record = new ProducerRecord<k,v>("this-topic", key, value);
try {
  //sync 模式 调用future.get()
  future = producer.send(record);
  RecordMetadata metadata = future.get();
} catch (Exception e) {
  e.printStackTrace();
}
producer.flush();
producer.close();
```





### **异步发送模式**



**异步发送模式「async」**，在调用 send() 方法的时候指定一个 callback 函数，当 Broker 接收到返回的时候，该 callback 函数会被触发执行，**通过回调函数能够对异常情况进行处理，当调用了回调函数时，只有回调函数执行完毕生产者才会结束，否则一直会阻塞**。



```java
Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback);
public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
        //intercept the record, which can be potentially modified; this method does not throw exceptions
        ProducerRecord<K, V> interceptedRecord = this.interceptors == null ? record : this.interceptors.onSend(record);
        return doSend(interceptedRecord, callback);
}
```



### 场景



> 1）**场景1**：**如果业务只是关心消息的吞吐量，且允许少量消息发送失败，也不关注消息的发送顺序的话，那么可以使用发后即忘发送「**fire-and-forget」的方式，配合参数 acks = 0，这样生产者并不需要等待服务器的响应，以网络能支持的最大速度发送消息。
>
> 2）**场景2**：**如果业务要求消息必须是按**顺序发送**的话，且数据只能落在一个 Partition 上，那么可以使用同步发送「**sync**」的方式，并结合参数来设置 retries 的值让消息发送失败时可以进行多次重试「**retries > 0**」，再结合参数设置「**acks=all & max_in_flight_requests_per_connection=1」，可以控制生产者在收到服务器成功响应之前只能发送1条消息，在消息发送成功后立即进行 flush, 从而达到控制消息顺序发送。
>
> 3）**场景3**：**如果业务需要知道消息是否发送成功，但对消息的顺序并不关心的话，那么可以用「**异步async + 回调 callback 函数」的方式来发送消息，并配合参数 retries=0，待发送失败时将失败的消息记录到日志文件中进行后续处理。





## 生产者如何选择分区

生产者发送消息的时候选择分区的策略方式主要有以下4种：



(1）**轮询策略：**顺序分配消息，即按照消息顺序依次发送到某Topic下不同的分区，它总是能保证消息最大限度地被平均分配到所有分区上，**如果消息在创建的时候 key 为 null， 那么Kafka 默认会采用这种策略**。



2）**消息key指定分区策略：**Kafka 允许为每条消息定义 key，即消息在创建的时候 key 不为空，此时 Kafka 会根据消息的 key 进行 hash, 然后根据 hash 值对 Partition 进行取模映射到指定的分区上， 这样的好处就是相同 key的消息会发送到同一个分区上， 这样 **Kafka 虽然不能保证全局有序，但是可以保证每个分区的消息是有序的，这就是消息分区有序性**，适应场景有下单支付的时候希望消息有序，可以通过订单 id 作为 key 发送消息达到分区有序性。



3）**随机策略：**随机发送到某个分区上，看似也是将消息均匀打散分配到各个分区，但是性能还是无法跟轮询策略比，「**如果追求数据的均匀分布，最好还是使用轮询策略**」。



4）**自定义策略：**可以通过实现 org.apache.kafka.clients.producer.Partitioner 接口，重写 partition 方法来达到自定义分区效果。



































# 消费者



## 消费模型

一般情况下消息消费总共有两种模式：「**推模型**」和 「**拉模型**」。在 Kafka 中的消费模型是属于「**拉模型**」，此模式的消息消费方式实现有两种：「**点对点方式**」和 「**发布订阅方式**」。



### 点对点模式

![image-20220722214542079](../../images/Kafka/image-20220722214542079.png)



**点对点方式:** 假如所有消费者都同属于一个消费组的话，此时所有的消息都会被分配给每一个消费者，**但是消息只会被其中一个消费者进行消费**。



### **发布订阅方式**

![image-20220722214555448](../../images/Kafka/image-20220722214555448.png)








发布订阅:假如所有消费者属于不同的消费组，此时所有的消息都会被分配给每一个消费者，**每个消费者都会收到该消息**。







### 总结：

对于点对点模式来说：生产者生产的消息发送至topic,只有一个消费者组进行消费消息，但topic多个partation的每个消息只能被消费者组某个消费者消费一次。

对于发布订阅模式，有多个消费者组进行消费消息，同一条消息可能被多个消费者消费。





## 消息分配策略（关注）



![image-20220723111909273](../../images/Kafka/image-20220723111909273.png)



这里主要说的是**消费的分区分配策略**，我们知道一个 Consumer Group 中有多个 Consumer，一个 Topic 也有多个 Partition，所以必然会有 Partition 分配问题「 ***\*确定哪个 Partition 由哪个 Consumer 来消费的问题\***」。



Kafka 客户端提供了3 种分区分配策略：RangeAssignor、RoundRobinAssignor 和 StickyAssignor，前两种分配方案相对简单一些StickyAssignor 分配方案相对复杂一些。



### **RangeAssignor** 



**RangeAssignor 是 Kafka 默认的分区分配算法，它是按照 Topic 的维度进行分配的**，首先对 每个Topic 的 Partition 按照分区ID进行排序，然后对订阅该 Topic 的 Consumer Group 的 Consumer 按名称字典进行排序，之后**尽量均衡的按照范围区段将分区分配给 Consumer**。此时也可能会造成先分配分区的 Consumer 任务过重（分区数无法被消费者数量整除）。



**分区分配场景分析如下图所示（同一个消费者组下的多个 consumer）：**



![image-20220723112206818](../../images/Kafka/image-20220723112206818.png)



**总结：该分配方式明显的问题就是随着消费者订阅的Topic的数量的增加，不均衡的问题会越来越严重。**





### **RoundRobinAssignor**

**该分区分配策略是将 Consumer Group 订阅的所有 Topic 的 Partition 及所有 Consumer 按照字典进行排序后尽量均衡的挨个进行分配。**如果 Consumer Group 内，每个 Consumer 订阅都订阅了相同的Topic，那么分配结果是均衡的。如果订阅 Topic 是不同的，那么分配结果是不保证「 **尽量均衡**」的，因为某些 Consumer 可能不参与一些 Topic 的分配。





**分区分配场景分析如下图所示：**

**1) 当组内每个 Consumer 订阅的相同 Topic ：**



![image-20220723112526211](../../images/Kafka/image-20220723112526211.png)



 **2) 当组内每个订阅的不同的 Topic ，这样就可能会造成分区订阅的倾斜:**



![image-20220723112542448](../../images/Kafka/image-20220723112542448.png)





### **StickyAssignor**





该分区分配算法是最复杂的一种，**可以通过 partition.assignment.strategy 参数去设置**，从 0.11 版本开始引入，目的就是在执行新分配时，尽量在上一次分配结果上少做调整，其主要实现了以下2个目标：



**1、Topic Partition 的分配要尽量均衡。**

**2、当 Rebalance 发生时，尽量与上一次分配结果保持一致。**



注意：**当两个目标发生冲突的时候，优先保证第一个目标，这样可以使分配更加均匀**，其中第一个目标是3种分配策略都尽量去尝试完成的， 而第二个目标才是该算法的精髓所在。



下面我们看看该策略与RoundRobinAssignor策略的不同：



**分区**分配场景分析如下图所示：

**1）组内每个 Consumer 订阅的相同的 Topic ，**RoundRobinAssignor 跟**StickyAssignor** **分配一致：**

![image-20220723112627281](../../images/Kafka/image-20220723112627281.png)



**当发生 Rebalance 情况后，可能分配会不太一样，假如这时候C1发生故障下线：**

**RoundRobinAssignor：**



![image-20220723112638425](../../images/Kafka/image-20220723112638425.png)



**StickyAssignor：**

![image-20220723112701295](../../images/Kafka/image-20220723112701295.png)





**结论: 从上面 Rebalance 发生后的结果可以看出，虽然两种分配策略最后都是均匀分配的，但是 RoundRoubin**Assignor 分区分配策略 **完全是重新分配了一遍，而 Sticky**Assignor **则是在原先的基础上达到了均匀的状态。**





**2) 当组内每个 Consumer 订阅的 Topic 是不同情况:**



https://mp.weixin.qq.com/s/Ny_VRCotJNE_4ZRLlMGSzg











## **Rebalance** 



**Kafka 消费者重平衡机制**



对于 Consumer Group 来说，可能随时都会有 Consumer 加入或退出，那么 Consumer 列表的变化必定会引起 Partition 的重新分配。我们将这个分配过程叫做 Consumer Rebalance，但是**这个分配过程需要借助 Broker 端的 Coordinator 协调者组件，在 Coordinator 的帮助下完成整个消费者组的分区重分配，也是通过监听ZooKeeper 的 /admin/reassign_partitions 节点触发的**。



### **Rebalance 触发与通知**



**Rebalance 的触发条件有三种:**



> 1）当 Consumer Group 组成员数量发生变化(主动加入或者主动离组，故障下线等)。
>
> 2）当订阅主题数量发生变化。
>
> 3）当订阅主题的分区数发生变化。



**Rebalance 触发后如何通知其他 Consumer 进程？**

> Rebalance 的通知机制就是**靠 Consumer 端的心跳线程**，它会定期发送心跳请求到 Broker 端的 Coordinator 协调者组件,当协调者决定开启 Rebalance 后，它会将「**REBALANCE_IN_PROGRESS**」封装进心跳请求的响应中发送给 Consumer ,当 Consumer 发现心跳响应中包含了「**REBALANCE_IN_PROGRESS**」，就知道是 Rebalance 开始了。













### **Rebalance 协议说明**



其实 Rebalance 本质上也是一组协议，Consumer Group 与 Coordinator 共同使用它来完成 Consumer Group 的 Rebalance。



下面我看看这5种协议完成了什么功能： 

> 1）**Heartbeat 请求：**Consumer 需要定期给 Coordinator 发送心跳来证明自己还活着。
>
> 2）**LeaveGroup 请求：**主动告诉 Coordinator 要离开 Consumer Group。
>
> 
>
> 3）**SyncGroup 请求：**Group Leader Consumer 把分配方案告诉组内所有成员。
>
> 
>
> 4）**JoinGroup 请求：**成员请求加入组。
>
> 
>
> 5）**DescribeGroup 请求：**显示组的所有信息，包括成员信息，协议名称，分配方案，订阅信息等。通常该请求是给管理员使用。



**Coordinator 在 Rebalance 的时候主要用到了前面4种请求。**





**Consumer Group 状态机**


如果 Rebalance 一旦发生，就会涉及到 Consumer Group 的状态流转，此时 Kafka 为我们设计了一套完整的状态机机制，来帮助 Broker Coordinator 完成整个重平衡流程。



了解整个状态流转过程可以帮助我们深入理解 Consumer Group 的设计原理。5种状态，定义分别如下：



> 1）**Empty 状态**：表示当前组内无成员， 但是可能存在 Consumer Group 已提交的位移数据，且未过期，这种状态只能响应 JoinGroup 请求。。
>
> 2）**Dead 状态**：表示组内已经没有任何成员的状态，组内的元数据已经被 Broker Coordinator 移除，这种状态响应各种请求都是一个Response：UNKNOWN_MEMBER_ID。
>
> 
>
> 3）**PreparingRebalance 状态**：表示准备开始新的 Rebalance, 等待组内所有成员重新加入组内。
>
> 
>
> 4）**CompletingRebalance 状态**：**表示组内成员都已经加入成功，正在等待分配方案，旧版本中叫「**AwaitingSyn」。
>
> 
>
> 5）**Stable 状态**：表示 Rebalance 已经完成， 组内 Consumer 可以开始消费了。







![image-20220723114407680](../../images/Kafka/image-20220723114407680.png)



### **Rebalance 流程分析**



通过上面5种状态可以看出，Rebalance 主要分为两个步骤：加入组「**JoinGroup请求**」和等待 Leader Consumer 分配方案「**SyncGroup 请求**」。



**JoinGroup请求**



> 组内所有成员向 Coordinator 发送 JoinGroup 请求，请求加入组，顺带会上报自己订阅的 Topic，这样 Coordinator 就能收集到所有成员的 JoinGroup 请求和订阅 Topic 信息，Coordinator 就会从这些成员中选择一个担任这个Consumer Group 的 Leader「**一般情况下，第一个发送请求的 Consumer 会成为 Leader**」。
>
> 
>
> **这里说的 Leader 是指具体的某一个 Consumer，它的任务就是收集所有成员的订阅 Topic 信息，然后制定具体的消费分区分配方案。**待选出 Leader 后，Coordinator 会把 Consumer Group 的订阅 Topic 信息封装进 JoinGroup 请求的 Response 中，然后发给 Leader ，然后由 Leader 统一做出分配方案后，进入到下一步



![image-20220723114654742](../../images/Kafka/image-20220723114654742.png)



**SyncGroup 请求**

> Leader 开始分配消费方案，**即哪个 Consumer 负责消费哪些 Topic 的哪些 Partition**。
>
> 
>
> 一旦完成分配，Leader 会将这个分配方案封装进 SyncGroup 请求中发给 Coordinator ，其他成员也会发 SyncGroup 请求，只是内容为空，待 Coordinator 接收到分配方案之后会把方案封装进 SyncGroup 的 Response 中发给组内各成员, 这样各自就知道应该消费哪些 Partition 了。



![image-20220723114721440](../../images/Kafka/image-20220723114721440.png)





### **Rebalance 场景分析**



刚刚详细的分析了关于 Rebalance 的状态流转，接下来我们通过时序图来重点分析几个场景来加深对 Rebalance 的理解。



**场景一：新成员(c1)加入组**



这里新成员加入组是指组处于 Stable 稳定状态后，有新成员加入的情况。当协调者收到新的 JoinGroup 请求后，它会通过心跳机制通知组内现有的所有成员，强制开启新一轮的重平衡。

**![图片](../../images/Kafka/640-16585480556943.png)**



**场景二：成员(c2)主动离组**



这里主动离组是指消费者所在线程或进程调用 close() 方法主动通知协调者它要退出。当协调者收到 LeaveGroup 请求后，依然会以心跳机制通知其他成员，强制开启新一轮的重平衡。





**![图片](../../images/Kafka/640-16585480556954.png)**



**场景三：成员(c2)超时被踢出组**



这里超时被踢出组是指消费者实例出现故障或者处理逻辑耗时过长导致的离组。此时离组是被动的，协调者需要等待一段时间才能感知到，**一般是由消费者端参数 session.timeout.ms 控制的**。



**![图片](../../images/Kafka/640-16585480556955.png)**



**场景四：成员(c2)提交位移数据**



当重平衡开启时，协调者会要求组内成员必须在这段缓冲时间内快速地提交自己的位移信息，然后再开启正常的 JoinGroup/SyncGroup 请求发送。



![图片](../../images/Kafka/640-16585480556966.png)







# 文件系统



## **消耗文件句柄方面分析**



**在 Kafka 的 Broker 中，每个 Partition 都会对应磁盘文件系统中一个目录**。在 Kafka 的日志文件目录中，每个日志数据段都会分配三个文件，两个索引文件和一个数据文件。**每个 Broker 会为每个日志段文件打开两个 index 文件句柄和一个 log 数据文件句柄**。因此，随着 Partition 的增多，所需要保持打开状态的文件句柄数也就越多，最终可能超过底层操作系统配置的文件句柄数量限制。



![image-20220722211903231](../../images/Kafka/image-20220722211903231.png)





## **端到端的延迟方面分析**




首先我们得先了解 Kafka 端对端延迟是什么? Producer 端发布消息到 Consumer 端接收消息所需要的时间，**即 Consumer 端接收消息的时间减去 Producer 端发布消息的时间**。



在 Kafka 中只对「**已提交的消息做最大限度的持久化保证不丢失**」，因此 Kafka 只有在消息提交之后，才会将消息暴露给消费者。**此时如果分区越多那么副本之间需要同步的数据就会越多**，假如消息需要在所有 ISR 副本集合列表同步复制完成之后才能进行暴露。**因此 ISR 副本集合间复制数据所花时间将是 kafka 端对端延迟的最主要部分**。



此时我们可以通过加大 kafka 集群来进行缓解。比如，我们将 100 个分区 Leader 放到一个 Broker 节点和放到 10 个 Broker 节点，它们之间的延迟是不一样的。如在 10 个 Broker 节点的集群中，每个 Broker 节点平均只需要处理 10 个分区的数据复制。此时端对端的延迟将会变成原来的十分之一。



**因此根据实战经验，如果你特别关心消息延迟问题的话，此时限制每个 Broker 节点的 Partition 数量是一个非常不错的主意：**对于 N 个 Broker 节点和复制副本因子「**replication-factor**」为 F 的 Kafka 集群，那么整个 Kafka 集群的 Partition 数量最好不超过 「**100 \* N \* F**」 个，即单个  Broker 节点 Partition 的 Leader 数量不超过100。











## **日志删除**

https://mp.weixin.qq.com/s/gm-c3gikWU0MwEJW_QIDFQ



Kafka 的日志管理器（LogManager）中有一个专门的日志清理任务通过周期性检测和删除不符合条件的日志分段文件（LogSegment），这里我们可以通过设置 Kafka Broker 端的参数「 ***\*log.retention.check.\**\**interval.ms\****」，默认值为300000，即5分钟。





## Offset



在 Kafka 中每个 Topic 分区下面的每条消息都被赋予了一个唯一的ID值，用来标识它在分区中的位置。这个ID值就被称为位移「**Offset**」或者叫**偏移量，**一旦消息被写入到日志分区中，它的位移值将不能被修改





### **位移 Offset 管理方式**



Kafka 旧版本（0.9版本之前）是把位移保存在 ZooKeeper 中，减少 Broker 端状态存储开销。



鉴于 Zookeeper 不适合频繁写更新，而 Consumer Group 的位移提交又是高频写操作，这样会拖慢 ZooKeeper 集群的性能， 于是在新版 Kafka 中， 社区采用了将位移保存在 Kafka 内部「**Kafka Topic 天然支持高频写且持久化**」，这就是所谓大名鼎鼎的__consumer_offsets。



**__consumer_offsets：**用来保存 Kafka Consumer 提交的位移信息，另外它是由 Kafka 自动创建的，和普通的 Topic 相同，它的消息格式也是 Kafka 自己定义的，我们无法进行修改。



__consumer_offsets 有3种消息格式：

> 1）用来保存 Consumer Group 信息的消息。
>
> 2）用来删除 Group 过期位移甚至是删除 Group 的消息，也可以称为 tombstone 消息，即墓碑消息，它的主要特点是空消息体，一旦某个 Consumer Group 下的所有Consumer 位移数据都已被删除时，Kafka会向 __consumer_offsets 主题的对应分区写入 tombstone 消息，表明要彻底删除这个 Group 的信息。
>
> \3) 用来保存位移值。





### **__consumer_offsets 消息格式分析揭秘：(重要重要重要)**



> 1) 消息格式我们可以简单理解为是一个 KV 对。Key 和 Value 分别表示消息的键值和消息体。
>
> 2) 那么 Key 存什么呢？既然是存储 Consumer 的位移信息，在 Kafka 中，Consumer 数量会很多，必须有字段来标识位移数据是属于哪个 Consumer 的，怎么来标识 Consumer 字段呢？我们知道 Consumer Group 会共享一个公共且唯一的 Group ID，那么只保存它就可以了吗？我们知道 Consumer 提交位移是在分区的维度进行的，很显然，key中还应该保存 Consumer 要提交位移的分区。
>
> 
>
> 3) 总结：位移主题的 Key 中应该保存 3 部分内容：<**Group ID，主题名，分区号**>
>
> 
>
> 4) value 可以简单认为存储的是 offset 值，当然底层还存储其他一些元数据，帮助 Kafka 来完成一些其他操作，比如删除过期位移数据等。





### **__consumer_offsets 消息格式示意图：**

![image-20220722214955087](../../images/Kafka/image-20220722214955087.png)







### **__consumer_offsets 创建**





__consumer_offsets 是怎么被创建出来的呢？ 当 Kafka 集群中的第一个 Consumer 启动时，Kafka 会自动创建__consumer_offsets。



它就是普通的 Topic，也有对应的分区数，如果由 Kafka 自动创建的，那么分区数又是怎么设置的呢？



这个依赖 Broker 端参数主题分区位移个数即「**offsets.topic.num.partitions**」 默认值是50，因此 Kafka 会自动创建一个有 50 个分区的 __consumer_offsets 。既然有分区数，必然就会有分区对应的副本个数，这个是依赖Broker 端另外一个参数来完成的，即 「**offsets.topic.replication.factor**」默认值为3。



总结一下， __consumer_offsets 由 Kafka 自动创建的，那么该 Topic 的分区数是 50，副本数是 3，而具体 Consumer Group 的消费情况要存储到哪个 Partition ，根据**abs(GroupId.hashCode()) % NumPartitions** 来计算的，这样就可以保证 Consumer Offset 信息与 Consumer Group 对应的 Coordinator 处于同一个 Broker 节点上。如下图所示：









# 高可用高性能



**我们知道 Kafka 是通过多副本复制技术来实现集群的高可用和稳定性的**。每个 Partition 都会有多个数据副本，每个副本分别存在于不同的 Broker 上。所有的数据副本中，其中一个数据副本为 Leader，其他的数据副本为 Follower。





**在Kafka集群内部，所有的数据副本采用自动化的方式管理且会确保所有副本之间的数据是保持同步状态的。**当 Broker 发生故障时，对于 Leader 副本所在 Broker 的所有 Partition 将会变得暂不可用。Kafka 将自动在其它副本中选择出一个 Leader，用于接收客户端的请求。这个过程由 Kafka Controller 节点 Broker 自动选举完成。





正常情况下，当一个 Broker 在有计划地停止服务时候，那么 Controller 会在服务停止之前，将该 Broker上 的所有 Leader 副本一个个地移走。对于单个 Leader 副本的移动速度非常快，从客户层面看，有计划的服务停服只会导致系统很短时间窗口不可用。



但是，当 Broker 不是正常停止服务时「**kill -9 强杀方式**」，系统的不可用时间窗口将会与受影响的 Partition 数量有关。如果此时发生宕机的 Broker 是 Controller 节点时, 这时 Controller 节点故障恢复会自动的进行，**但是新的 Controller 节点需要从 Zookeeper 中读取每一个 Partition 的元数据信息用于初始化数据**。假设一个 Kafka 集群存在10000个 Partition，从 Zookeeper 中恢复元数据时每个 Partition 大约花费2ms，则 Controller 恢复将会增加约20秒的不可用时间窗口。



**总之，通常情况下 Kafka 集群中越多的 Partition 会带来越高的吞吐量。但是，如果 Kafka 集群中 Partition 总量过大或者单个 Broker 节点 Partition 过多，都可能会对系统的可用性和消息延迟带来潜在的负面影响，需要引起我们的重视。**



## 读写数据快



### **顺序追加写**



kafka 在写数据的时是以「**磁盘顺序写」**的方式来进行落盘的, 即将数据追加到文件的末尾。对于普通机械磁盘, 如果是随机写的话, 涉及到磁盘寻址的问题, 导致性能极低  **但是如果只是按照顺序的方式追加文件末尾的话, 这种磁盘顺序写的性能基本可以跟写内存的性能差不多的**。下图所示普通机械磁盘的顺序I/O性能指标是53.2M values/s。



![image-20220722214242369](../../images/Kafka/image-20220722214242369.png)





### **Page Cache**

首先 Kafka 为了保证磁盘写入性能，**通过 mmap 内存映射的方式利用操作系统的 Page Cache 异步写入 。**也可以称为 os cache，意思就是操作系统自己管理的缓存。**那么在写磁盘文件的时候，就可以先直接写入 os cache 中，接下来由操作系统自己决定什么时候把 os cache 里的数据真正刷入到磁盘中, 这样大大提高写入效率和性能**。 如下图所示:



![image-20220722214257327](../../images/Kafka/image-20220722214257327.png)



### **零拷贝技术**

**零拷贝还得看小林coding**:https://xiaolincoding.com/os/8_network_system/zero_copy.html#mmap-write



kafka 为了解决**内核态和用户态数据不必要 Copy** 这个问题, 在读取数据的时候就引入了「**零拷贝技术**」。即让操作系统的 os cache 中的数据**直接发送到**网卡后传出给下游的消费者，中间跳过了两次拷贝数据的步骤，从而减少拷贝的 CPU 开销, 减少用户态内核态的上下文切换次数, 从而优化数据传输的性能, **而Socket缓存中仅仅会拷贝一个描述符过去，不会拷贝数据到Socket缓存**，如下图所示：



![image-20220722214309835](../../images/Kafka/image-20220722214309835.png)







在 Kafka 中主要有以下两个地方使用到了「**零拷贝技术**」**:**

> 1）**基于 mmap 机制实现的索引文件：**首先索引文件都是**基于 MappedByBuffer 实现**，即让用户态和内核态来共享内核态的数据缓冲区，此时数据不需要 Copy 到用户态空间。虽然 mmap 避免了不必要的 Copy，但是在不同操作系统下， 其创建和销毁成功是不一样的，不一定都能保证高性能。所以在 Kafka 中只有索引文件使用了 mmap。
>
> 2）**基于sendfile 机制实现的日志文件读写：**在 Kafka 传输层接口中有个 TransportLayer 接口，它的实现类中有使用了 Java FileChannel 中 transferTo 方法。该方法底层就是使用 sendfile 实现的零拷贝机制， 目前只是在 I/O 通道是普通的 PLAINTEXT 的时候才会使用到零拷贝机制。







## 数据可靠性



**HW：**全称「**Hign WaterMark**」 ，即高水位，它标识了一个特定的消息偏移量 offset ，消费者只能拉取到这个水位 offset 之前的消息。



**LEO：**全称「**Log End Offset**」，它标识当前日志文件中下一条待写入的消息的 offset，在 ISR 副本集合中的每个副本都会维护自身的LEO。



**作用：**

> **HW 作用：**
>
> 1）用来标识分区下的哪些消息是可以被消费者消费的。
>
> 2）协助 Kafka 完成副本数据同步。
>
> **LEO 作用：**
>
> 1）如果 Follower 和 Leader 的 LEO 数据同步了, 那么 HW 就可以更新了。
>
> 2）HW 之前的消息数据对消费者是可见的，属于 commited 状态, HW 之后的消息数据对消费者是不可见的。





# 消息有序

我们知道在 Kafka 中，并不保证消息全局有序，但是可以**保证分区有序性，**分区与分区之间是无序的。**那么如何保证 Kafka 中的消息是有序的呢？** 可以从以下三个方面来入手分析：

## **Producer**

首先 Kafka 的 Producer 端发送消息，如果是不对默认参数进行任何设置且网络没有抖动的情况下，消息是可以一批批的按消息发送的顺序被发送到 Kafka Broker 端。但是，一旦有网络波动了，则消息就可能出现乱序。



Kafka 分布式的单位是 partition，同一个 partition 用一个 write ahead log 组织，所以可以保证 FIFO 的顺序。不同 partition 之间不能保证顺序。但是绝大多数用户都可以通过 message key 来定义，因为同一个 key 的 message 可以保证只发送到同一个 partition。

Kafka 中发送 1 条消息的时候，可以指定(topic, partition, key) 3 个参数。partiton 和 key 是可选的。如果你指定了 partition，那就是所有消息发往同 1个 partition，就是有序的。并且在消费端，Kafka 保证，1 个 partition 只能被1 个 consumer 消费。或者你指定 key（ 比如 order id），具有同 1 个 key 的所有消息，会发往同 1 个 partition。但是消费者内部如果多线程就有问题，此时的解决方案是【使用内存队列处理，将key hash后分发到内存队列中，然后每个线程处理一个内存队列的数据。



key做hash然后取模分区数量，这样就可以根据key将消息发送到同一个分区。

## **Broker**

在 Kafka 中，Topic 只是一个逻辑上的概念，**而组成 Topic 的分区 Partition 才是真正存消息的地方**。



Kafka 只保证单分区内的消息是有序的，**所以如果要保证业务全局严格有序，就要设置 Topic 为单分区的方式**。不过对业务来说一般不需要考虑全局有序的，只需要**保证业务中不同类别的消息有序**即可。



**但是这里有个必须要受到重视的问题**，就是当我们对分区 Partition 进行数量改变的时候，由于是简单的 Hash 算法会把以前可能分到相同分区的消息分到别的分区上。这样就不能保证消息顺序了。面对这种情况，**就需要在动态变更分区的时候，考虑对业务的影响。有可能需要根据业务和当前分区需求，重新划分消息类别**。





## **Consumer**

在 Consumer 端，**根据 Kafka 的模型，一个 Topic 下的每个分区只能从属于这个 Topic 的消费者组中的某一个消费者**。



当消息被发送分配到同一个 Partition 中，消费者从 Partition 中取出来数据的时候，也一定是有顺序的，没有错乱。



但是消费者可能会有多个线程来并发来消费消息。如果单线程消费数据，吞吐量太低了，而多个线程并发消费的话，顺序可能就乱掉了。



**重点：**

**此时可以通过写多个内存队列，将相同 key 的消息都写入同一个队列，然后对于多个线程，每个线程分别消息一个队列即可保证消息顺序**。







# 消息批量发送



Kafka 在发送消息的时候并不是一条条的发送的，而是会**把多条消息合并成一个批次**Batch **进行处理发送**，消费消息也是同样，一次拉取一批次的消息进行消费。如下图所示：



![image-20220722214431095](../../images/Kafka/image-20220722214431095.png)

# 消息语义





对于 Kafka 来说，当消息从 Producer 到 Consumer，有许多因素来影响消息的消费，因此「**消息传递语义**」就是 Kafka 提供的 Producer 和 Consumer 之间的消息传递过程中消息传递的保证性。主要分为三种， 如下图所示：





![image-20220723101109575](../../images/Kafka/image-20220723101109575.png)







## **Producer端**



**生产者发送语义：**首先当 Producer 向 Broker 发送数据后，会进行消息提交，如果成功消息不会丢失。因此发送一条消息后，可能会有几种情况发生：



> 1）遇到网络问题造成通信中断， 导致 Producer 端无法收到 ack，Producer 无法准确判断该消息是否已经被提交， 又重新发送消息，这就可能造成 「**at least once**」语义。
>
> 
>
> 2）在 Kafka 0.11之前的版本，会导致消息在 Broker 上重复写入（保证至少一次语义），但在0.11版本开始，通过引入「**PID及Sequence Number**」支持幂等性，保证精确一次「**exactly once**」语义。
>
> 
>
> **其中启用幂等传递的方法配置**：enable.idempotence = true。**启用事务支持的方法配置**：设置属性 transcational.id = "指定值"。
>
> 
>
> 3）可以根据 Producer 端 request.required.acks 的配置来取值。
>
> 
>
> **acks = 0：**由于发送后就自认为发送成功，这时如果发生网络抖动， Producer 端并不会校验 ACK 自然也就丢了，且无法重试。
>
> 
>
> **acks = 1：**消息发送 Leader Parition 接收成功就表示发送成功，这时只要 Leader Partition 不 Crash 掉，就可以保证 Leader Partition 不丢数据，保证 「**at least once**」语义。
>
> 
>
> **acks = -1 或者 all：** 消息发送需要等待 ISR 中 Leader Partition 和 所有的 Follower Partition 都确认收到消息才算发送成功, 可靠性最高, 但也不能保证不丢数据,比如当 ISR 中只剩下 Leader Partition 了, 这样就变成 acks = 1 的情况了，保证 「**at least once**」语义。 

![image-20220723101403330](../../images/Kafka/image-20220723101403330.png)







## **Consumer端**



**消费者接收语义：**从 Consumer 角度来剖析，我们知道 Offset 是由 Consumer 自己来维护的。



Consumer 消费消息时，有以下2种选择:



> 1）**读取消息 -> 提交offset -> 处理消息:** 如果此时保存 offset 成功，但处理消息失败，Consumer 又挂了，会发生 Reblance，新接管的 Consumer 将从上次保存的 offset 的下一条继续消费，导致消息丢失，保证「**at most once**」语义。
>
> 2）**读取消息 -> 处理消息 -> 提交offset：**如果此时消息处理成功，但保存 offset 失败，Consumer 又挂了，导致刚才消费的 offset 没有被成功提交，会发生 Reblance，新接管的 Consumer 将从上次保存的 offset 的下一条继续消费，导致消息重复消费，保证「**at least once**」语义。



### 总结：

**默认 Kafka 提供 「**at least once**」语义的消息传递，允许用户通过在处理消息之前保存 Offset 的方式提供 「**at most once**」 语义。如果我们可以自己实现消费幂等，理想情况下这个系统的消息传递就是严格的「**exactly once**」, 也就是保证不丢失、且只会被精确的处理一次，但是这样是很难做到的。







# 消息积压



**Kafka线上大量消息积压你是如何处理的？**



**消息大量积压这个问题，直接原因一定是某个环节出现了性能问题，来不及消费消息，才会导致消息积压。**接着就比较坑爹了，此时假如 Kafka 集群的磁盘都快写满了，都没有被消费，这个时候怎么办？或者消息积压了⼏个⼩时，这个时候怎么办？生产环境挺常⻅的问题，⼀般不出问题，而⼀出就是⼤事故。



所以，我们先来分析下，在使用 Kafka 时如何来优化代码的性能，避免出现消息积压。如果你的线上 Kafka 系统出现了消息积压，该如何进行紧急处理，最大程度地避免消息积压对业务的影响。





## **优化性能来避免消息积压**


对于 Kafka 性能的优化，主要体现在生产者和消费者这两部分业务逻辑中。而 Kafka 本身不需要多关注的主要原因是，对于绝大多数使用Kafka 的业务来说，Kafka 本身的处理能力要远大于业务系统的处理能力。Kafka 单个节点，消息收发的性能可以达到每秒钟处理几十万条消息的水平，还可以通过水平扩展 Broker 的实例数成倍地提升处理能力。



对于业务系统处理的业务逻辑要复杂一些，单个节点每秒钟处理几百到几千次请求，已经非常不错了，所以我们应该更关注的是消息的收发两端。



### **发送端性能优化**



发送端即生产者业务代码都是先执行自己的业务逻辑，最后再发送消息。**如果说性能上不去，需要你优先检查一下，是不是发消息之前的业务逻辑耗时太多导致的**。



对于发送消息的业务逻辑，只需要注意设置合适的并发和批量大小，就可以达到很好的发送性能。我们知道 Producer 发消息给 Broker 且收到消息并返回 ack 响应，假设这一次过程的平均时延是 1ms，它包括了下面这些步骤的耗时：



> 1）发送端在发送网络请求之前的耗时。
>
> 2）发送消息和返回响应在网络传输中的耗时。
>
> 
>
> 3）Broker 端处理消息的时延。

假设此时你的发送端是单线程，每次只能发送 1 条消息，那么每秒只能发送 1000 条消息，这种情况下并不能发挥出 Kafka 的真实性能。此时无论是增加每次发送消息的批量大小，还是增加并发，都可以成倍地提升发送性能。



如果当前发送端是在线服务的话，比较在意请求响应时延，此时可以采用并发方式来提升性能。



如果当前发送端是离线服务的话，更关注系统的吞吐量，发送数据一般都来自数据库，此时更适合批量读取，批量发送来提升性能。



**另外还需要关注下消息体是否过大**，如果消息体过大，势必会增加 IO 的耗时，影响 Kafka 生产和消费的速度，也可能会造成消息积压。



### 消费端性能优化

**而在使用 Kafka 时，大部分的性能问题都出现在消费端，如果消费的速度跟不上发送端生产消息的速度，就会造成消息积压。如果只是暂时的，那问题不大，只要消费端的性能恢复之后，超过发送端的性能，那积压的消息是可以逐渐被消化掉的。**



要是消费速度一直比生产速度慢，时间长了系统就会出现问题，比如Kafka 的磁盘存储被写满无法提供服务，或者消息丢失，对于整个系统来说都是严重故障。



所以我们在设计的时候，**一定要保证消费端的消费性能要高于生产端的发送性能，这样的系统才能健康的持续运行**。



消费端的性能优化除了优化业务逻辑外，也可以通过水平扩容，增加消费端的并发数来提升总体的消费性能。需要注意的是，**在扩容 Consumer 的实例数量的同时，必须同步扩容主题中的分区数量，确保 Consumer 的实例数和分区数量是相等的**，如果 Consumer 的实例数量超过分区数量，这样的扩容实际上是没有效果的。





## **消息积压后如何处理**


日常系统正常时候，没有积压或者只有少量积压很快就消费掉了，但某时刻，突然开始积压消息且持续上涨。这种情况下需要你在短时间内找到消息积压的原因，迅速解决问题。



**导致消息积压突然增加，只有两种：发送变快了或者消费变慢了。**



假如赶上大促或者抢购时，短时间内不太可能优化消费端的代码来提升消费性能，**此时唯一的办法是通过扩容消费端的实例数来提升总体的消费能力**。如果短时间内没有足够的服务器资源进行扩容，只能降级一些不重要的业务，减少发送方发送的数据量，最低限度让系统还能正常运转，保证重要业务服务正常。



假如通过内部监控到消费变慢了，需要你检查消费实例，分析一下是什么原因导致消费变慢？



1、优先查看日志是否有大量的消费错误。

2、此时如果没有错误的话，可以通过打印堆栈信息，看一下你的消费线程是不是卡在哪里「**触发死锁或者卡在某些等待资源**」。



# 消息丢失



https://mp.weixin.qq.com/s/5L4mODPQZrfwz94qcFNYuQ



## Producer端丢失



### 生产者可靠性保证

**消息经生产者发送至Broker，可能还没到Broker，消息就丢失了。**

所以

 **Kafka 在 Producer 里面提供了消息确认机制**



> 从本质上来说，生产者与Broker之间是通过网络进行通讯的，因此保障生产者的消息可靠性就必须要保证网络可靠性，这里Kafka使用`acks=all`可以设置Broker收到消息并同步到所有从节点后给生产者一个确认消息。如果生产者没有收到确认消息就会多次重复向Broker发送消息，保证在生产者与Broker之间的消息可靠性。





通过 `request.required.acks` 参数设置的，





> - `acks = 0`：意味着如果生产者能够通过网络把消息发送出去，那么就认为消息已成功写入 Kafka 。在这种情况下还是有可能发生错误，比如发送的对象无能被序列化或者网卡发生故障，但如果是分区离线或整个集群长时间不可用，那就不会收到任何错误。在 acks=0 模式下的运行速度是非常快的（这就是为什么很多基准测试都是基于这个模式），你可以得到惊人的吞吐量和带宽利用率，不过如果选择了这种模式， 一定会丢失一些消息。
> - `acks = 1`：意味着 Leader 在收到消息并把它写入到本地磁盘时会返回确认或错误响应，不管其它的 Follower 副本有没有同步过这条消息。在这个模式下，如果发生正常的 Leader 选举，生产者会在选举时收到一个 LeaderNotAvailableException 异常，如果生产者能恰当地处理这个错误，它会重试发送消息，最终消息会安全到达新的 Leader 那里。不过在这个模式下仍然有可能丢失数据，比如消息已经成功写入 Leader，但在消息被复制到 Follower 副本之前 Leader发生崩溃。
> - `acks = all`（这个和 `request.required.acks = -1` 含义一样）：意味着 Leader 在返回确认或错误响应之前，会等待所有同步副本都收到消息。如果和 `min.insync.replicas` 参数结合起来，就可以决定在返回确认前至少有多少个副本能够收到消息，生产者会一直重试直到消息被成功提交。不过这也是最慢的做法，因为生产者在继续发送其他消息之前需要等待所有副本都收到当前的消息。





根据实际的应用场景，我们设置不同的 acks，以此保证数据的可靠性。





另外，Producer 发送消息还可以选择同步（默认，通过 producer.type=sync 配置） 或者异步（producer.type=async）模式。如果设置成异步，虽然会极大的提高消息发送的性能，但是这样会增加丢失数据的风险。如果需要确保消息的可靠性，必须将 producer.type 设置为 sync。







##  Broker 端丢失场景







**Kafka Broker 集群接收到数据后会将数据进行持久化存储到磁盘，为了提高吞吐量和性能，采用的是「异步批量刷盘的策略」，也就是说按照一定的消息量和间隔时间进行刷盘。首先会将数据存储到 「PageCache」 中，至于什么时候将 Cache 中的数据刷盘是由「操作系统」根据自己的策略决定或者调用 fsync 命令进行强制刷盘，如果此时 Broker 宕机 Crash 掉，且选举了一个落后 Leader Partition 很多的 Follower Partition 成为新的 Leader Partition，那么落后的消息数据就会丢失。**



一句话:异步刷盘这个过程还有可能导致数据会丢

### 解决


既然 Broker 端消息存储是通过异步批量刷盘的，那么这里就可能会丢数据的**!!!**

- 由于 Kafka 中并没有提供「**同步刷盘**」的方式，所以说从**单个 Broker** 来看还是很有可能丢失数据的。（rocketmq好像有同步刷盘机制）
- kafka 通过「**多 Partition （分区）多 Replica（副本）机制」****已经可以最大限度的保证数据不丢失**，如果数据已经写入 PageCache 中但是还没来得及刷写到磁盘，此时如果所在 Broker 突然宕机挂掉或者停电，极端情况还是会造成数据丢失。







## 引用

> 在Broker收到了生产者的消息后，也有可能会丢失消息，最常见的情况是消息到达某个Broker后服务器就宕机了。这里需要补充说明一下Kafka的高可用性，直观的看，Kafka一般可被分成多个Broker节点，而为了增加Kafka的吞吐量，一个topic通常被分为多个partition，每个partition分布在不同的Broker上。如果一个partition丢失就会导致topic内容的部分丢失，因此partition往往需要多个副本，以此来保证高可用。
>
> ![img](../../images/Kafka/ef624202071d4678a425a6704f3c016btplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp)
>
> 如上图所示，一个topic被分成三个partition，每个partition又进行复制，假如此时Broker1挂了，在Broker2和Broker3上依然有完整的三个partition。此时只需要重新选举出partition的leader即可。这里还是需要强调一下，一定要将leader上的数据同步到follower上才能给生产者返回消息，否则可能造成消息丢失。
>
> 
> 





##  Consumer 端丢失场景

**offset**：指的是kafka的topic中的每个消费组消费的下标。



**Consumer消费消息有下面几个步骤**：

- 接收消息
- 处理消息
- 反馈“处理完毕”（commited）,提交偏移量。

**Consumer的消费方式主要分为两种：**

- 自动提交offset，Automatic Offset Committing
- 手动提交offset，Manual Offset Control



**自动提交offset**

Consumer自动提交的机制是根据一定的时间间隔，将收到的消息进行commit（**提交偏移量**）。commit过程和消费消息的过程是异步的。也就是说，**可能存在消费过程未成功（比如抛出异常），commit消息已经提交了**。此时消息就丢失了。



拉取消息后「**先提交 Offset，后处理消息**」，如果此时处理消息的时候异常宕机，由于 Offset 已经提交了,  待 Consumer 重启后，会从之前已提交的 Offset 下一个位置重新开始消费， 之前未处理完成的消息不会被再次处理，对于该 Consumer 来说消息就丢失了。



### 解决

**手动提交offset**

将提交类型改为手动以后，可以保证消息“至少被消费一次”(at least once)。但此时可能出现**重复消费**的情况（消费者消费了数据没来得及手动提交offset然后消费者挂了，重启就会重复消费）**重复消费的话：手动解决幂等：将每条消息的全局唯一id(生产者生成的)存进mysql或者redis,要操作数据的话先判断mysql或者redis有没有**



- 拉取消息后「**先处理消息，在进行提交 Offset**」， 如果此时在提交之前发生异常宕机，由于没有提交成功 Offset， 待下次 Consumer 重启后还会从上次的 Offset 重新拉取消息，不会出现消息丢失的情况， 但是会出现重复消费的情况，这里只能业务自己保证幂等性。  （消息消费成功后才会提交offset）











## 消息丢失解决方案

在剖析 Producer 端丢失场景的时候， 我们得出其是通过「**异步**」方式进行发送的，所以如果此时是使用「**发后即焚**」的方式发送，即调用 Producer.send(msg) 会立即返回，由于没有回调，可能因网络原因导致 Broker 并没有收到消息，此时就丢失了。



因此我们可以从以下几方面进行解决 Producer 端消息丢失问题：

**更换调用方式：**

弃用调用发后即焚的方式，使用带回调通知函数的方法进行发送消息，即 **Producer.send(msg, callback)**, 这样一旦发现发送失败， 就可以做针对性处理。



弃用调用发后即焚的方式，使用带回调通知函数的方法进行发送消息，即 **Producer.send(msg, callback)**, 这样一旦发现发送失败， 就可以做针对性处理。

```java

Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback);

public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
        // intercept the record, which can be potentially modified; this method does not throw exceptions
        ProducerRecord<K, V> interceptedRecord = this.interceptors == null ? record : this.interceptors.onSend(record);
        return doSend(interceptedRecord, callback);
}
```



**ACK 确认机制**



 **Broker 端解决方案**

在剖析 Broker 端丢失场景的时候， 我们得出其是通过「**异步批量刷盘**」的策略，先将数据存储到 「**PageCache**」，再进行异步刷盘， 由于没有提供 「**同****步刷盘**」策略， 因此 Kafka 是通过「**多分区多副本**」的方式来最大限度的保证数据不丢失。









 **Consumer 端解决方案**



在剖析 Consumer 端丢失场景的时候，我们得出其拉取完消息后是需要提交 Offset 位移信息的，因此为了不丢数据，正确的做法是：**拉取数据、**业务逻辑处理、提交消费 Offset 位移信息。


我们还需要设置参数 **enable.auto.commit = false, 采用手动提交位移的方式。**



另外对于消费消息重复的情况，业务自己保证幂等性, **保证只成功消费一次即可**。



























# 高吞吐日志存储架构



对于 Kafka 来说， 它主要用来处理海量数据流，这个场景的特点主要包括：



> 1) **写操作：**写并发要求非常高，基本得达到百万级 TPS，顺序追加写日志即可，无需考虑更新操作。
>
> 
>
> 2）**读操作：**相对写操作来说，比较简单，只要能按照一定规则高效查询即可,支持（offset或者时间戳）读取。



根据上面两点分析，对于写操作来说，直接采用「**顺序追加写日志**」的方式就可以满足 Kafka 对于百万TPS写入效率要求。



如何解决高效查询这些日志呢？我们可以设想把消息的 Offset 设计成一个有序的字段，这样消息在日志文件中也就有序存放了，也不需要**额外引入哈希表结构**，可以直接将消息划分成若干个块，**对于每个块我们只需要索引当前块的第一条消息的 Offset ，这个是不是有点二分查找算法的意思**。即先根据 Offset 大小找到对应的块， 然后再从块中顺序查找。如下图所示：



![图片](../../images/Kafka/640-165854923322511.png)



这样就可以快速定位到要查找的消息的位置了，在 Kafka 中，我们将这种索引结构叫做「**稀疏哈希索引**」。



上面得出了 Kafka 最终的存储实现方案， 即**基于顺序追加写日志 + 稀疏哈希索引。**



接下来我们来看看 Kafka 日志存储结构：



![图片](../../images/Kafka/640-165854923322612.png)



从上图看出来，Kafka 是基于「**主题 + 分区 + 副本 + 分段 + 索引**」的结构进行日志存储的。



了解了整体的日志存储架构，我们来看下 Kafka 日志格式，Kafka 日志格式也经历了多个版本迭代，这里我们主要看下V2版本的日志格式：

![图片](../../images/Kafka/640-165854923322613.png)



通过上图可以得出：V2 版本日志格式主要是通过可变长度提高了消息格式的空间使用率，并将某些字段抽取到消息批次（RecordBatch）中，同时消息批次可以存放多条消息，从而在批量发送消息时，可以大幅度地节省了磁盘空间。



接下来我们来看看日志消息写入磁盘的整体过程如下图所示：



![图片](../../images/Kafka/640-165854923322614.png)









# 集群|线上|JVM监控

**一文解决**

https://mp.weixin.qq.com/s/HksQtlhYG2hRHMKD85ktnw



## Kafka客户端如何巧妙解决JVM GC问题？





### **Kafka 客户端缓冲机制**



首先，大家知道的就是在客户端发送消息给 Kafka 服务端的时候，存在一个「**内存缓冲池机制**」 的。即消息会先写入一个内存缓冲池中，然后直到多条消息组成了一个 Batch，达到一定条件才会一次网络通信把 Batch 发送过去。





**整个发送过程图如下所示：**

![image-20220723095916226](../../images/Kafka/image-20220723095916226.png)







### **内存缓冲造成的频繁GC问题**

内存缓冲机制说白了，其实就是**把多条消息组成一个Batch，一次网络请求就是一个Batch 或者多个 Batch**。这样避免了一条消息一次网络请求，从而提升了吞吐量。



那么问题来了，试想一下一个 Batch 中的数据取出来封装到网络包里，通过网络发送到达 Kafka 服务端。**此时**这个 Batch 里的数据都发送过去了，里面的数据该怎么处理？这些 Batch 里的数据还存在客户端的 JVM 的内存里！那么一定要避免任何变量去引用 Batch 对应的数据，然后尝试触发 JVM 自动回收掉这些内存垃圾。这样不断的让 JVM 进行垃圾回收，就可以不断的腾出来新的内存空间让后面新的数据来使用。



想法是挺好，但**实际生产运行的时候最大的问题，就是 JVM Full GC 问题**。JVM GC 在回收内存垃圾的时候，会有一个「**Stop the World**」的过程，即垃圾回收线程运行的时候，会导致其他工作线程短暂的停顿，这样可以踏踏实实的回收内存垃圾。



**试想一下，在回收内存垃圾的时候，工作线程还在不断的往内存里写数据，那如何让JVM 回收垃圾呢？**我们看看下面这张图就更加清楚了：



![image-20220723100155306](../../images/Kafka/image-20220723100155306.png)



虽然现在 JVM GC 演进越来越先进，从 CMS 垃圾回收器到 G1 垃圾回收器，**核心的目标之一就是不断的缩减垃圾回收的时候，导致其他工作线程停顿的时间**。但是再先进的垃圾回收器这个停顿的时间还是存在的。



因此，如何尽可能在设计上避免 JVM 频繁的 Full GC 就是一个非常考验其设计水平了。



### **Kafka 实现的缓冲机制**



在 Kafka 客户端内部，针对这个问题实现了一个非常优秀的机制，就是「**缓冲池机制**」。即每个 Batch 底层都对应一块内存空间，这个内存空间就是专门用来存放写进去的消息。



当一个 Batch 数据被发送到了 kafka 服务端，这个 Batch 的内存空间不再使用了。**此时这个 Batch 底层的内存空间先不交给 JVM 去垃圾回收，而是把**这块内存空间给放入一个缓冲池里。



这个缓冲池里存放了很多块内存空间，下次如果有一个新的 Batch 数据了，那么直接从缓冲池获取一块内存空间是不是就可以了？然后如果一个 Batch 数据发送出去了之后，再把内存空间还回来是不是就可以了？以此类推，循环往复。



一旦使用了这个缓冲池机制之后，就不涉及到频繁的大量内存的 GC 问题了。



> 
>
> **初始化分配固定的内存，即32MB。然后把 32MB 划分为 N 多个内存块，一个内存块默认是16KB，这样缓冲池里就会有很多的内存块**。然后如果需要创建一个新的 Batch，就从缓冲池里取一个 16KB 的内存块就可以了。
>
> 
>
> 接着如果 Batch 数据被发送到 Kafka 服务端了，此时 Batch 底层的内存块就直接还回缓冲池就可以了。这样循环往复就可以利用有限的内存，那么就不涉及到垃圾回收了。没有频繁的垃圾回收，自然就避免了频繁导致的工作线程的停顿了，JVM Full GC 问题是不是就得到了大幅度的优化？
>
> 
>
> **没错，正是这个设计思想让 Kafka 客户端的性能和吞吐量都非常的高，这里蕴含了大量的优秀的机制****。**
>
> 









# 消息重复



https://mp.weixin.qq.com/s/5L4mODPQZrfwz94qcFNYuQ


生产端不重复生产消息，服务端不重复存储消息，消费端也不能重复消费消息。**

 

**相较上面“消息不丢失”的场景，“消息不重复”的服务端无须做特别的配置，因为服务端不会重复存储消息，如果有重复消息也应该是由生产端重复发送造成的。也就是说，下面我们只需要分析生产端和消费端就行。**

 

 

## **生产端：不重复生产消息**

 



**问题出现**

**生产端发送消息后，服务端已经收到消息了，但是假如遇到网络问题，无法获得响应，生产端就无法判断该消息是否成功提交到了 Kafka，而我们一般会配置重试次数，但这样会引发生产端重新发送同一条消息，从而造成消息重复的发送。**

 

### 解决

kafka 0.11.0.0版本之后，正式推出了idempotent producer，支持生产者的[幂等](https://so.csdn.net/so/search?q=幂等&spm=1001.2101.3001.7020)。

启动kafka的幂等性，需要修改配置文件:enable.idempotence=true 同时要求 ack=all 且 retries>1。



**kafka内部对生产者的幂等处理**



**为了实现生产者的幂等性，Kafka引入了 Producer ID(PID)和 Sequence Number的概念**



1.D：每个Producer在初始化时，都会分配一个唯一的PID，这个PID对用户来说，是透明的。

2.equence Number：针对每个生产者(对应PID)发送到指定主题分区的消息都对应一个从0开始递增的Sequence Number。





 每个生产者producer都有一个唯一id，producer每发送一条数据都会带上一个sequence；

在borker端（Broker端在缓存中保存了这seq numb），当消息落盘，sequence就会递增1。只需判断当前消息的sequence是否大于当前最大sequence，大于就代表此条数据没有落盘过，可以正常消费；不大于就代表落盘过，这个时候重发的消息会被服务端拒掉从而避免**消息重复。**



 

## **消费端：不能重复消费消息**



**问题出现**

**数据消费完没有及时提交offset到broke。然后消费者挂了，重启之后或者其他消费者启动拿之前记录的offset开始消费，导致消息重复消费。**



**采用自动提交，当消费者拉取到了分区的某个消息之后，消费者会自动提交了 offset。自动提交的话会有一个问题，试想一下，当消费者刚拿到这个消息准备进行真正消费的时候，突然挂掉了，消息实际上并没有被消费，但是 offset 却被自动提交了，发生了消息丢失。**







## 消费者幂等处理（重点）

生产者也也要不发送重复消息



**不消费重复消息可以做消费者这边做一个幂等处理**









- **生产者为每条消息生成一个全局唯一id**
- **在消费者消费消息之前，先去redis查是否存在该key,如果存在说明已经处理过了，丢掉消息。**
- **如果redis没处理，继续往下处理，最终的逻辑是将处理过的数据插入到业务DB上，最后在把幂等key插入到Redis上**
- **Redis其实只是一个前置处理，最终的幂等性是依赖数据库的唯一key来保证的（唯一key就是消息id）**
- **总的来说，就是通过Redis做前置处理，DB唯一索引来最终保证生产者消费消息幂等**



**总结：**

**重复的消息先redis已经去重了，然后消费消息，将消费过的消息存进db（依靠db唯一key）,再将唯一key存进redis供下次消息去重**







## 手动提交offset

![image-20220922103459830](../../images/Kafka/image-20220922103459830.png)























### 解决

**自动提交改为手动提交offset**



**下游做幂等（重要）：**

- 生产者为每条消息生成一个全局唯一id.
- 在消费者消费消息之前，先去redis查是否存在该key,如果存在说明已经处理过了，丢掉消息。
- 如果redis没处理，继续往下处理，最终的逻辑是将处理过的数据插入到业务DB上，再到最后把幂等Key插入到Redis上
- Redis其实只是一个「前置」处理，最终的幂等性是依赖数据库的唯一Key来保证的（唯一Key实际上也是订单编号+状态）
- 总的来说，就是通过Redis做前置处理，DB唯一索引做最终保证来实现幂等性的













![image-20220910174743817](../../images/Kafka/image-20220910174743817.png)



# kafka为什么快



Kafka的消息是保存或缓存在磁盘上的，一般认为在磁盘上读写数据是会降低性能的，因为寻址会比较消耗时间，但是实际上，Kafka的特性之一就是高吞吐率。



- **partition 并行处理**
- 零拷贝
- 磁盘顺序写
- 先写到内核缓冲区（Page Cache），再刷盘。





- **采用了零拷贝技术 Producer 生产的数据持久化到 broker，采用 mmap 文件映射，实现顺序的快速写入 Customer 从 broker 读取数据，采用 sendfile，将磁盘文件读到 OS 内核缓冲区后，转到 NIO buffer 进行网络发送，减少 CPU 消耗**























