---
title: Rocketmq
categories:
  - 中间件
tags:
  - Rocketmq
abbrlink: 56288
date: 2022-05-28 19:54:20
cover : https://www.leixue.com/uploads/2020/07/RocketMQ.png!760
---

Rocketmq是一款消息队列

# 作用



## 解耦

引入消息队列之前，下单完成之后，需要订单服务去调用库存服务减库存，调用营销服务加营销数据……引入消息队列之后，可以把订单完成的消息丢进队列里，下游服务自己去调用就行了，这样就完成了订单服务和其它服务的解耦合。

![image-20220524200720935](../../images/Rocketmq/image-20220524200720935.png)

## 异步

订单支付之后，我们要扣减库存、增加积分、发送消息等等，这样一来这个链路就长了，链路一长，响应时间就变长了。引入消息队列，除了`更新订单状态`，其它的都可以**异步**去做，这样一来就来，**就能降低响应时间**。



![image-20220524200814011](../../images/Rocketmq/image-20220524200814011.png)





## 削峰

消息队列合一用来削峰，例如秒杀系统，平时流量很低，但是要做秒杀活动，秒杀的时候流量疯狂怼进来，我们的服务器，Redis，MySQL各自的承受能力都不一样，直接全部流量照单全收肯定有问题啊，严重点可能直接打挂了。



我们可以把请求扔到队列里面，只放出我们服务能处理的流量，这样就能抗住短时间的大流量了





![image-20220524201005576](../../images/Rocketmq/image-20220524201005576.png)







解耦、异步、削峰，是消息队列最主要的三大作用。



# 消息队列对比

![图片](../../images/Rocketmq/640.png)



面向C端系统，像电商网站这种，具有一定的并发量，对性能也有比较高的要求，所以选择低延迟、吞吐量、可用性比较高的Rocketmq,而Kafka多用于大数据实时计算于日志采集/=。





# 消息领域模型



RocketMQ使用的消息模型是标准的发布-订阅模型，在RocketMQ的术语表中，生产者、消费者和主题，与发布-订阅模型中的概念是完全一样的。



![image-20220524201931620](../../images/Rocketmq/image-20220524201931620.png)



## Message

**Message**（消息）就是要传输的信息。



一条消息必须有一个主题（Topic），主题可以看做是你的信件要邮寄的地址。

## **Topic**

**Topic**（主题）可以看做消息的归类，它是消息的第一级类型。比如一个电商系统可以分为：交易消息、物流消息等，一条消息必须有一个 Topic 



**Topic** 与生产者和消费者的关系非常松散，一个 Topic 可以有0个、1个、多个生产者向其发送消息，一个生产者也可以同时向不同的 Topic 发送消息。

一个 Topic 也可以被 0个、1个、多个消费者订阅。



## **Tag**

**Tag**（标签）可以看作子主题，它是消息的第二级类型，用于为用户提供额外的灵活性。使用标签，同一业务模块不同目的的消息就可以用相同 Topic 而不同的 **Tag** 来标识。比如交易消息又可以分为：交易创建消息、交易完成消息等，一条消息可以没有 **Tag** 。





## **Group**

RocketMQ中，订阅者的概念是通过消费组（Consumer Group）来体现的。每个消费组都消费主题中一份完整的消息，不同消费组之间消费进度彼此不受影响，也就是说，一条消息被Consumer Group1消费过，也会再给Consumer Group2消费。





消费组中包含多个消费者，同一个组内的消费者是竞争消费的关系，每个消费者负责消费组内的一部分消息。默认情况，如果一条消息被消费者Consumer1消费了，那同组的其他消费者就不会再收到这条消息。





## **Message Queue**



**Message Queue**（消息队列），一个 Topic 下可以设置多个消息队列，Topic 包括多个 Message Queue ，如果一个 Consumer 需要获取 Topic下所有的消息，就要遍历所有的 Message Queue。



PS:Topic下面的Message Queue是分布在多个Broker上的



## **Offset**

在Topic的消费过程中，由于消息需要被不同的组进行多次消费，所以消费完的消息并不会立即被删除，这就需要RocketMQ为每个消费组在每个队列上维护一个消费位置（Consumer Offset），这个位置之前的消息都被消费过，之后的消息都没有被消费过，每成功消费一条消息，消费位置就加一。

也可以这么说，`Queue` 是一个长度无限的数组，**Offset** 就是下标。

RocketMQ的消息模型中，这些就是比较关键的概念了。画张图总结一下：



![image-20220524202620231](../../images/Rocketmq/image-20220524202620231.png)





# 架构组成

他主要有四大核心组成部分：**NameServer**、**Broker**、**Producer**以及**Consumer**四部分。

![image-20220524202953540](../../images/Rocketmq/image-20220524202953540.png)

## NameServer

**NameServer**是一个功能齐全的服务器，其角色类似Dubbo中的Zookeeper，但NameServer与Zookeeper相比更轻量。主要是因为每个NameServer节点互相之间是独立的，没有任何信息交互。



**NameServer**压力不会太大，平时主要开销是在维持心跳和提供Topic-Broker的关系数据。





每个 Broker 在启动的时候会到 NameServer 注册，Producer 在发送消息前会根据 Topic 到 **NameServer** 获取到 Broker 的路由信息，Consumer 也会定时获取 Topic 的路由信息。





## Broker

- **Broker**是具体提供业务的服务器，单个Broker节点与所有的NameServer节点保持长连接及心跳，并会定时将**Topic**信息注册到NameServer，顺带一提底层的通信和连接都是**基于Netty实现**的。
- **Broker**负责消息存储，以Topic为纬度支持轻量级的队列，单机可以支撑上万队列规模，支持消息推拉模型。
- 官网上有数据显示：具有**上亿级消息堆积能力**，同时可**严格保证消息的有序性**。



## Producer

消息生产者，负责产生消息，一般由业务系统负责产生消息。



### 发送消息



- **RocketMQ** 提供了三种方式发送消息：同步、异步和单向

- **同步发送**：同步发送指消息发送方发出数据后会在收到接收方发回响应之后才发下一个数据包。一般用于重要通知消息，例如重要通知邮件、营销短信。
- **异步发送**：异步发送指发送方发出数据后，不等接收方发回响应，接着发送下个数据包，一般用于可能链路耗时较长而对响应时间敏感的业务场景，例如用户视频上传后通知启动转码服务。
- **单向发送**：单向发送是指只负责发送消息而不等待服务器回应且没有回调函数触发，适用于某些耗时非常短但对可靠性要求并不高的场景，例如日志收集。



## Consumer



- **Consumer**也由用户部署，支持PUSH和PULL两种消费模式，支持**集群消费**和**广播消息**，提供**实时的消息订阅机制**。
- **Pull**：拉取型消费者（Pull Consumer）主动从消息服务器拉取信息，只要批量拉取到消息，用户应用就会启动消费过程，所以 Pull 称为主动消费型。
- **Push**：推送型消费者（Push Consumer）封装了消息的拉取、消费进度和其他的内部维护工作，将消息到达时执行的回调接口留给用户应用程序来实现。所以 Push 称为被动消费类型，但从实现上看还是从消息服务器中拉取消息，不同于 Pull 的是 Push 首先要注册消费监听器，当监听器处触发后才开始消费消息。





## 集群消费与广播消费

**集群消费**

默认情况下就是集群消费，这种模式下`一个消费者组共同消费一个主题的多个队列，一个队列只会被一个消费者消费`，如果某个消费者挂掉，分组内其它消费者会接替挂掉的消费者继续消费。



**即任意一条消息，只需要被消费者集群内消费组的任意一个消费者处理即可**



**广播消费**

而广播消费消息会发给消费者组中的每一个消费者进行消费。





# 消息丢失

![image-20220524204450176](../../images/Rocketmq/image-20220524204450176.png)

## 什么情况下会丢失消息

其中，1，2，4三个场景都是跨网络的，而跨网络就肯定会有丢消息的可能。

然后关于3这个环节，通常MQ存盘时都会先写入操作系统的缓存page cache中，然后再由操作系统异步的将消息写入硬盘。这个中间有个时间差，就可能会造成消息丢失。如果服务挂了，缓存中还没有来得及写入硬盘的消息就会丢失。





## 生产阶段



在生产阶段，主要**通过请求确认机制，来保证消息的可靠传递**。



- 1、同步发送的时候，要注意处理响应结果和异常。如果返回响应OK，表示消息成功发送到了Broker，如果响应失败，或者发生其它异常，都应该重试。
- 2、异步发送的时候，应该在回调方法里检查，如果发送失败或者异常，都应该进行重试。
- 3、如果发生超时的情况，也可以通过查询日志的API，来检查是否在Broker存储成功。





## 存储阶段



存储阶段，可以通过**配置可靠性优先的 Broker 参数来避免因为宕机丢消息**，简单说就是可靠性优先的场景都应该使用同步。



- 1、消息只要持久化到CommitLog（日志文件）中，即使Broker宕机，未消费的消息也能重新恢复再消费。
- 2、Broker的刷盘机制：同步刷盘和异步刷盘，不管哪种刷盘都可以保证消息一定存储在pagecache中（内存中），但是同步刷盘更可靠，它是Producer发送消息后等数据持久化到磁盘之后再返回响应给Producer。



![image-20220524204807535](../../images/Rocketmq/image-20220524204807535.png)



- 3、Broker通过主从模式来保证高可用，Broker支持Master和Slave同步复制、Master和Slave异步复制模式，生产者的消息都是发送给Master，但是消费既可以从Master消费，也可以从Slave消费。同步复制模式可以保证即使Master宕机，消息肯定在Slave中有备份，保证了消息不会丢失。

## 消费阶段

- Consumer保证消息成功消费的关键在于确认的时机，不要在收到消息后就立即发送消费确认，而是应该在执行完所有消费业务逻辑之后，再发送消费确认。因为消息队列维护了消费的位置，逻辑执行失败了，没有确认，再去队列拉取消息，就还是之前的一条。



# 消息重复



**这块极客时间-消息队列高手课-基础篇-消息重复讲的很好**





> 在 MQTT 协议中，给出了三种传递消息时能够提供的服务质量标准，这三种服务质量从低到高依次是：
>
> - **At most once**: 至多一次。消息在传递时，最多会被送达一次。换一个说法就是，没什么消息可靠性保证，允许丢消息。一般都是一些对消息可靠性要求不太高的监控场景使用，比如每分钟上报一次机房温度数据，可以接受数据少量丢失。
> - **At least once**: 至少一次。消息在传递时，至少会被送达一次。也就是说，不允许丢消息，但是允许有少量重复消息出现。
> - **Exactly once**：恰好一次。消息在传递时，只会被送达一次，不允许丢失也不允许重复，这个是最高的等级。



回到正题

在消息传递过程中，如果出现传递失败的情况，发送方会执行重试，重试的过程中就有可能会产生重复的消息。对使用消息队列的业务系统来说，如果没有对重复消息进行处理，就有可能会导致系统的数据出现错误。

比如说，一个消费订单消息，统计下单金额的微服务，如果没有正确处理重复消息，那就会出现重复统计，导致统计结果错误。



**去重原则：使用业务端逻辑保持幂等性**



## 幂等性

**幂等性**：就是用户对于同一操作发起的一次请求或者多次请求的结果是一致的，不会因为多次点击而产生了副作用，数据库的结果都是唯一的，不可变的。



一个幂等操作的特点是，**其任意多次执行所产生的影响均与一次执行的影响相同。**



只要保持幂等性，不管来多少条重复消息，最后处理的结果都一样，需要业务端来实现。



## **去重策略**

具体的实现方法是，在发送消息时，给每条消息（就算重发，重发的消息还是一样的）指定一个全局唯一的 ID，消费时，先根据这个 ID 检查这条消息是否有被消费过，如果没有消费过，才更新数据，然后将消费状态置为已消费。



**这个全局唯一ID很难实现**



其他「解决方案」

- **「**[**数据库**](https://cloud.tencent.com/solution/database?from=10680)**表」** 处理消息前，使用消息主键在表中带有约束的字段中insert
- **「Map」** 单机时可以使用map ConcurrentHashMap -> putIfAbsent  guava cache
- **「**[**Redis**](https://cloud.tencent.com/product/crs?from=10680)**」** 分布式锁搞起来。







# 消息积压



RocketMQ的消息积压一般指**某个topic的大量消息没有被消费**，比如消费这个topic的消费逻辑涉及的数据库宕机了或其他原因



![image-20220524212319142](../../images/Rocketmq/image-20220524212319142.png)

## 解决



- **消费者扩容**：如果当前Topic的Message Queue的数量大于消费者数量，就可以对消费者进行扩容，增加消费者，来提高消费能力，尽快把积压的消息消费玩。



- **消息迁移Queue扩容**：如果当前Topic的Message Queue的数量小于或者等于消费者数量，这种情况，再扩容消费者就没什么用，就得考虑扩容Message Queue。可以新建一个临时的Topic，临时的Topic多设置一些Message Queue，然后先用一些消费者把要消费的数据丢到临时的Topic，因为不用业务处理，只是转发一下消息，还是很快的。接下来用扩容的消费者去消费新的Topic里的数据，消费完了之后，恢复原状。

# 顺序消息



顺序消息是指消息的消费顺序和产生顺序相同，在有些业务逻辑下，必须保证顺序，比如订单的生成、付款、发货，这个消息必须按顺序处理才行。



顺序消息分为全局顺序消息和部分顺序消息，全局顺序消息指某个 Topic 下的所有消息都要保证顺序；



部分顺序消息只要保证每一组消息被顺序消费即可，比如订单消息，只要保证同一个订单 ID 个消息能按顺序消费即可。





## 局部顺序



部分顺序消息相对比较好实现，生产端需要做到把同 ID 的消息发送到同一个 Message Queue ；在消费过程中，要做到从同一个Message Queue读取的消息顺序处理——消费端不能并发处理顺序消息，这样才能达到部分有序。

![image-20220524213644634](../../images/Rocketmq/image-20220524213644634.png)

发送端使用 MessageQueueSelector 类来控制 把消息发往哪个 Message Queu



> **选择队列的过程由 messageQueueSelector 和 hashKey 在实现类 SelectMessageQueueByHash 中完成**
>
> 为了保证发送有序，**RocketMQ**提供了**MessageQueueSelector**队列选择机制，他有三种实现:

![图片](../../images/Rocketmq/640-16533999394352.png)





我们可使用**Hash取模法**，让同一个订单发送到同一个队列中，再使用同步发送，只有同个订单的创建消息发送成功，再发送支付消息。这样，我们保证了发送有序。



**RocketMQ**的topic内的队列机制,可以保证存储满足**FIFO**（First Input First Output 简单说就是指先进先出）,剩下的只需要消费者顺序消费即可。







### MessageQueueSelector 



RocketMQ 提供了很多 MessageQueueSelector 的实现，例如随机选择策略，哈希选择策略和同机房选择策略等，



**1.SelectMessageQueueByHash**

```java
public class SelectMessageQueueByHash implements MessageQueueSelector {
    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        int value = arg.hashCode();
        if (value < 0) {
            value = Math.abs(value);
        }
        value = value % mqs.size();
        return mqs.get(value);
    }
}
```

- SelectMessageQueueByHash实现了MessageQueueSelector接口，其select方法取arg参数的hashcode的绝对值，然后对mqs.size()取余，得到目标队列在mqs的下标
- 顺序消息，生产者就是基于这个方法来选择队列的。



**2.SelectMessageQueueByRandom**



```java
public class SelectMessageQueueByRandom implements MessageQueueSelector {
    private Random random = new Random(System.currentTimeMillis());

    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        int value = random.nextInt(mqs.size());
        return mqs.get(value);
    }
}
```



- SelectMessageQueueByRandom实现了MessageQueueSelector接口，其select方法直接根据mqs.size()随机一个值作为目标队列在mqs的下标



**3.SelectMessageQueueByMachineRoom**



```java
public class SelectMessageQueueByMachineRoom implements MessageQueueSelector {
    private Set<String> consumeridcs;
    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        return null;
    }

    public Set<String> getConsumeridcs() {
        return consumeridcs;
    }
    public void setConsumeridcs(Set<String> consumeridcs) {
        this.consumeridcs = consumeridcs;
    }
}
```



- SelectMessageQueueByMachineRoom实现了MessageQueueSelector接口，其select方法目前返回null





消费端通过使用 MessageListenerOrderly 来解决单 Message Queue 的消息被并发处理的问题。





## 全局顺序



RocketMQ 默认情况下不保证顺序，比如创建一个 Topic ，默认八个写队列，八个读队列，这时候一条消息可能被写入任意一个队列里；在数据的读取过程中，可能有多个 Consumer ，每个 Consumer 也可能启动多个线程并行处理，所以消息被哪个 Consumer 消费，被消费的顺序和写人的顺序是否一致是不确定的。

要保证全局顺序消息， 需要先把 Topic 的读写队列数设置为 一，然后Producer Consumer 的并发设置，也要是一。简单来说，为了保证整个 Topic全局消息有序，只能消除所有的并发处理，各部分都设置成单线程处理 ，这时候就完全牺牲RocketMQ的高并发、高吞吐的特性了（也就是设置成一个队列）。





# 事务消息



**消息队列中的“事务”，主要解决的是消息生产者和消息消费者的数据一致性问题。**





## 消息队列是如何实现分布式事务的？



事务消息需要消息队列提供相应的功能才能实现，Kafka 和 RocketMQ 都提供了事务相关功能。



在 RocketMQ 中的事务实现中，增加了事务反查的机制来解决事务消息提交失败的问题。如果 Producer 也就是订单系统，在提交或者回滚事务消息时发生网络异常，RocketMQ 的 Broker 没有收到提交或者回滚的请求，Broker 会定期去 Producer 上反查这个事务对应的本地事务的状态，然后根据反查结果决定提交或者回滚这个事务。



> 为了支撑这个事务反查机制，我们的业务代码需要实现一个反查本地事务状态的接口，告知 RocketMQ 本地事务是成功还是失败。
>
> 在例子中（创建订单后将商品从购物车中删除），反查本地事务的逻辑也很简单，我们只要根据消息中的订单 ID，在订单库中查询这个订单是否存在即可，如果订单存在则返回成功，否则返回失败。RocketMQ 会自动根据事务反查的结果提交或者回滚事务消息。
>
> 这个反查本地事务的实现，并不依赖消息的发送方，也就是订单服务的某个实例节点上的任何数据。这种情况下，即使是发送事务消息的那个订单服务节点宕机了，RocketMQ 依然可以通过其他订单服务的节点来执行反查，确保事务的完整性。



![image-20220524215333833](../../images/Rocketmq/image-20220524215333833.png)





再来个实例图对比一下

![image-20220524215607642](../../images/Rocketmq/image-20220524215607642.png)





在第四步提交事务消息时失败了Kafka 直接抛出异常，让用户自行处理。我们可以在业务代码中反复重试提交，直到提交成功，或者删除之前创建的订单进行补偿



而RocketMq提供了事务反查的机制。



## 步骤



- 1、Producer 向 broker 发送半消息
- 2、Producer 端收到响应，消息发送成功，此时消息是半消息，标记为 “不可投递” 状态，Consumer 消费不了。
- 3、Producer 端执行本地事务。
- 4、正常情况本地事务执行完成，Producer 向 Broker 发送 Commit/Rollback，如果是 Commit，Broker 端将半消息标记为正常消息，Consumer 可以消费，如果是 Rollback，Broker 丢弃此消息。
- 5、异常情况，Broker 端迟迟等不到二次确认。在一定时间后，会查询所有的半消息，然后到 Producer 端查询半消息的执行情况。（消息回查）
- 6、Producer 端查询本地事务的状态
- 7、根据事务的状态提交 commit/rollback 到 broker 端。
- 8、消费者段消费到消息之后，执行本地事务，执行本地事务。





# 消息过滤



有两种方案：

- 一种是在 Broker 端按照 Consumer 的去重逻辑进行过滤，这样做的好处是避免了无用的消息传输到 Consumer 端，缺点是加重了 Broker 的负担，实现起来相对复杂。
- 另一种是在 Consumer 端过滤，比如按照消息设置的 tag 去重，这样的好处是实现起来简单，缺点是有大量无用的消息到达了 Consumer 端只能丢弃不处理。

一般采用Cosumer端过滤，如果希望提高吞吐量，可以采用Broker过滤。



对消息的过滤有三种方式：

![image-20220524215718090](../../images/Rocketmq/image-20220524215718090.png)















# **Buffer**



Broker的**Buffer**通常指的是Broker中一个队列的内存Buffer大小，这类**Buffer**通常大小有限。

另外，RocketMQ没有内存**Buffer**概念，RocketMQ的队列都是持久化磁盘，数据定期清除。

RocketMQ同其他MQ有非常显著的区别，RocketMQ的内存**Buffer**抽象成一个无限长度的队列，不管有多少数据进来都能装得下，这个无限是有前提的，Broker会定期删除过期的数据。

例如Broker只保存3天的消息，那么这个**Buffer**虽然长度无限，但是3天前的数据会被从队尾删除。



# 延迟消息

电商的订单超时自动取消，就是一个典型的利用延时消息的例子，用户提交了一个订单，就可以发送一个延时消息，1h后去检查这个订单的状态，如果还是未付款就取消订单释放库存。



RocketMQ是支持延时消息的，只需要在生产消息的时候设置消息的延时级别：



## RocketMQ怎么实现延时消息的？





简单，八个字：`临时存储`+`定时任务`。





Broker收到延时消息了，会先发送到主题（SCHEDULE_TOPIC_XXXX）的相应时间段的Message Queue中，然后通过一个定时任务轮询这些队列，到期后，把消息投递到目标Topic的队列中，然后消费者就可以正常消费这些消息。

![image-20220524220333709](../../images/Rocketmq/image-20220524220333709.png)







# 死信队列



死信队列用于处理无法被正常消费的消息，即死信消息。

当一条消息初次消费失败，**消息队列 RocketMQ 会自动进行消息重试**；达到最大重试次数后，若消费依然失败，则表明消费者在正常情况下无法正确地消费该消息，此时，消息队列 RocketMQ 不会立刻将消息丢弃，而是将其发送到该**消费者对应的特殊队列中**，该特殊队列称为**死信队列**。





**死信消息的特点**：

- 不会再被消费者正常消费。
- 有效期与正常消息相同，均为 3 天，3 天后会被自动删除。因此，需要在死信消息产生后的 3 天内及时处理。



# 负载均衡

RocketMQ中的负载均衡都在Client端完成，具体来说的话，主要可以分为Producer端发送消息时候的负载均衡和Consumer端订阅消息的负载均衡。



## Producer

Producer端在发送消息的时候，会先根据Topic找到指定的TopicPublishInfo，在获取了TopicPublishInfo路由信息后，RocketMQ的客户端在默认方式下selectOneMessageQueue()方法会从TopicPublishInfo中的messageQueueList中选择一个队列（MessageQueue）进行发送消息。具这里有一个sendLatencyFaultEnable开关变量，如果开启，在随机递增取模的基础上，再过滤掉not available的Broker代理。



![image-20220524222352370](../../images/Rocketmq/image-20220524222352370.png)







```java
public MessageQueue selectOneMessageQueue(final String lastBrokerName) {
    // 第一次就是null，第二次（也就是重试的时候）就不是null了。
    if (lastBrokerName == null) {
        // 第一次选择队列的逻辑
        return selectOneMessageQueue();
    } else {
        // 第一次选择队列发送消息失败了，第二次重试的时候选择队列的逻辑
        
        int index = this.sendWhichQueue.getAndIncrement();
        for (int i = 0; i < this.messageQueueList.size(); i++) {
            int pos = Math.abs(index++) % this.messageQueueList.size();
            if (pos < 0)
                pos = 0;
            MessageQueue mq = this.messageQueueList.get(pos);
   // 过滤掉上次发送消息失败的队列
            if (!mq.getBrokerName().equals(lastBrokerName)) {
                return mq;
            }
        }`
        return selectOneMessageQueue();
    }
}
```









第一次选择队列的逻辑



```java
public MessageQueue selectOneMessageQueue() {
    // 当前线程有个ThreadLocal变量，存放了一个随机数 {@link org.apache.rocketmq.client.common.ThreadLocalIndex#getAndIncrement}
    // 然后取出随机数根据队列长度取模且将随机数+1
    int index = this.sendWhichQueue.getAndIncrement();
    int pos = Math.abs(index) % this.messageQueueList.size();
    if (pos < 0) {
        pos = 0;
    }
    return this.messageQueueList.get(pos);
}
```









## Consumer







### **概述**

所谓的负载均衡指的就是，在集群消费模式下，一个消费组里面有多个消费者订阅了一个主题，此主题有多个[消息队列](https://so.csdn.net/so/search?q=消息队列&spm=1001.2101.3001.7020)(`MessageQueue`)，负载均衡组件就将这些消息队列平均分给消费组里面的消费者。





### doRebalance

```java
case CLUSTERING: {

    //先获得主题对应的消息队列列表
    //这个map是会被一个从NameServer拉取路由信息来更新的
    Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);

    //找到主题对应的该消费组内所有的消费ID
    List<String> cidAll = this.mQClientFactory.findConsumerIdList(topic, consumerGroup);
    if (null == mqSet) {
        if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
            log.warn("doRebalance, {}, but the topic[{}] not exist.", consumerGroup, topic);
        }
    }

    if (null == cidAll) {
        log.warn("doRebalance, {} {}, get consumer id list failed", consumerGroup, topic);
    }

    if (mqSet != null && cidAll != null) {
        List<MessageQueue> mqAll = new ArrayList<MessageQueue>();
        mqAll.addAll(mqSet);

        Collections.sort(mqAll);
        Collections.sort(cidAll);

        AllocateMessageQueueStrategy strategy = this.allocateMessageQueueStrategy;

        List<MessageQueue> allocateResult = null;
        try {

            //调用具体的负载均衡策略进行负载
            //负载的含义：将该主题的mq在该消费组的所有的消费者进行分配
            allocateResult = strategy.allocate(
                this.consumerGroup,
                this.mQClientFactory.getClientId(),
                mqAll,
                cidAll);
        } catch (Throwable e) {
            log.error("AllocateMessageQueueStrategy.allocate Exception. allocateMessageQueueStrategyName={}", strategy.getName(),
                e);
            return;
        }

        Set<MessageQueue> allocateResultSet = new HashSet<MessageQueue>();
        if (allocateResult != null) {
            allocateResultSet.addAll(allocateResult);
        }
        //看当前主题当前消费者的分配到的mq是否变化了
        // 下面这个函数很重要，如果消费者分配到了一个之前没有的mq，那么将会创建一个PullRequest给PullMessageService实现对消息的拉取，如果之前有但是现在没有的那么将会调用里面的ProcessQueue的droped方法来销毁，即不再拉取消息。
        boolean changed = this.updateProcessQueueTableInRebalance(topic, allocateResultSet, isOrder);
        if (changed) {
            log.info(
             
                strategy.getName(), consumerGroup, topic, this.mQClientFactory.getClientId(), mqSet.size(), cidAll.size(),
                allocateResultSet.size(), allocateResultSet);
            this.messageQueueChanged(topic, mqSet, allocateResultSet);
        }
    }
    break;
}
```







### 原理

![img](../../images/Rocketmq/1566274-20220311154643278-574763268.png)





- 第一步：先获得该topic下面的所有queueId,即先获得该主题下的所有消费消息队列
- 第二部：获取订阅了该topic的消费者组的消费者id列表
- 第三步：经过负载均衡分配算法（平均分配算法），最终在该topic下的所有队列当中，选择其中几个队列分配给当前消费者端
- 第四步：处理队列变更



![图片](../../images/Rocketmq/640-16534025215105.png)







消费者计算出分配给自己的队列结果后，需要与之前进行比较，判断添加了新的队列，或者移除了之前分配的队列，也可能没有变化。

- 对于新增的队列，需要先计算从哪个位置开始消费，接着从这个位置开始拉取消息进行消费；
- 对于移除的队列，要移除缓存的消息，并停止拉取消息，并持久化offset。







```java
 boolean changed = this.updateProcessQueueTableInRebalance(topic, allocateResultSet, isOrder);
```



该方法 会遍历processQueueTable 中的 MessageQueue 是否在 新分配的集合中 ， 会做如下几件事：

- **将 processQueueTable 中 不在新分配集合中的 mq删除， 同时删除 offsetStore 中的该mq**
- **将 processQueueTable 中 超时未拉取新消息的 mq 删除，同时删除 offsetStore 中的该mq**
- **添加 新分配的 mq 到 processQueueTable 中 。 并向 拉消息服务 中发起 已给 拉取 该mq消费信息的 请求**







下面是这个方法详解，虽然我看的有点懵

> 调用updateProcessQueueTableInRebalance()方法，具体的做法是，先将分配到的消息队列集合（mqSet）与processQueueTable做一个过滤比对。![图片](../../images/Rocketmq/640-16534025215106.png)
>
> - 上图中processQueueTable标注的红色部分，表示与分配到的消息队列集合mqSet互不包含。将这些队列设置Dropped属性为true，然后查看这些队列是否可以移除出processQueueTable缓存变量，这里具体执行removeUnnecessaryMessageQueue()方法，即每隔1s 查看是否可以获取当前消费处理队列的锁，拿到的话返回true。如果等待1s后，仍然拿不到当前消费处理队列的锁则返回false。如果返回true，则从processQueueTable缓存变量中移除对应的Entry；
> - 上图中processQueueTable的绿色部分，表示与分配到的消息队列集合mqSet的交集。判断该ProcessQueue是否已经过期了，在Pull模式的不用管，如果是Push模式的，设置Dropped属性为true，并且调用removeUnnecessaryMessageQueue()方法，像上面一样尝试移除Entry；
> - 最后，为过滤后的消息队列集合（mqSet）中的每个MessageQueue创建一个ProcessQueue对象并存入RebalanceImpl的processQueueTable队列中（其中调用RebalanceImpl实例的computePullFromWhere(MessageQueue mq)方法获取该MessageQueue对象的下一个进度消费值offset，随后填充至接下来要创建的pullRequest对象属性中），并创建拉取请求对象—pullRequest添加到拉取列表—pullRequestList中，最后执行dispatchPullRequest()方法，将Pull消息的请求对象PullRequest依次放入PullMessageService服务线程的阻塞队列pullRequestQueue中，待该服务线程取出后向Broker端发起Pull消息的请求。其中，可以重点对比下，RebalancePushImpl和RebalancePullImpl两个实现类的dispatchPullRequest()方法不同，RebalancePullImpl类里面的该方法为空。













### 重要

**当消费负载均衡consumer和queue不对等的时候会发生什么？**



解：Consumer和queue会优先平均分配，如果Consumer少于queue的个数，则会存在部分Consumer消费多个queue的情况，如果Consumer等于queue的个数，那就是一个Consumer消费一个queue，如果Consumer个数大于queue的个数，那么会有部分Consumer空余出来，白白的浪费了。




**消息消费队列在同一消费组不同消费者之间的负载均衡，其核心设计理念是在一个消息消费队列在同一时间只允许被同一消费组内的一个消费者消费，一个消息消费者能同时消费多个消息队列。**







# ConsumeQueue



ConsumeQueue作为RocketMQ存储实现的重要部分，它提供了每个Topic下每个Queue中消息存储的索引功能。也就是说，每个Topic下的每个Queue都会有一个ConsumeQueue保存该Queue下消息存储在CommitLog的位置。
ConsumeQueue和CommitLog一样，都是基于MappedFileQueue实现的。所以ConsumeQueue和CommitLog的实现有很多相似的地方



下面是 consumeQueue记录的信息

![image-20220526192741928](../../images/Rocketmq/image-20220526192741928.png)



# RocketMQ原理





## RocketMQ工作流程

RocketMQ是一个分布式消息队列，也就是`消息队列`+`分布式系统`



RocketMQ由NameServer注册中心集群、Producer生产者集群、Consumer消费者集群和若干Broker（RocketMQ进程）组成



1. Broker在启动的时候去向所有的NameServer注册，并保持长连接，每30s发送一次心跳
2. Producer在发送消息的时候从NameServer获取Broker服务器地址，根据负载均衡算法选择一台服务器来发送消息
3. Conusmer消费消息的时候同样从NameServer获取Broker地址，然后主动拉取消息来消费



![image-20220524221423177](../../images/Rocketmq/image-20220524221423177.png)

### 消息处理整个流程



1. 消息接收：消息接收是指接收`producer`的消息，处理类是`SendMessageProcessor`，将消息写入到`commigLog`文件后，接收流程处理完毕；
2. 消息分发：`broker`处理消息分发的类是`ReputMessageService`，它会启动一个线程，不断地将`commitLong`分到到对应的`consumerQueue`，这一步操作会写两个文件：`consumerQueue`与`indexFile`，写入后，消息分发流程处理 完毕；
3. 消息投递：消息投递是指将消息发往`consumer`的流程，`consumer`会发起获取消息的请求，`broker`收到请求后，调用`PullMessageProcessor`类处理，从`consumerQueue`文件获取消息，返回给`consumer`后，投递流程处理完毕。











## 为什么RocketMQ不使用Zookeeper作为注册中心呢？

这个没学分布式的一些理论和Zookeeper，暂时还不太理解，先写答案再说。



1. 基于可用性的考虑，根据CAP理论，同时最多只能满足两个点，而Zookeeper满足的是CP，也就是说Zookeeper并不能保证服务的可用性，Zookeeper在进行选举的时候，整个选举的时间太长，期间整个集群都处于不可用的状态，而这对于一个注册中心来说肯定是不能接受的，作为服务发现来说就应该是为可用性而设计。
2. 基于性能的考虑，NameServer本身的实现非常轻量，而且可以通过增加机器的方式水平扩展，增加集群的抗压能力，而Zookeeper的写是不可扩展的，Zookeeper要解决这个问题只能通过划分领域，划分多个Zookeeper集群来解决，首先操作起来太复杂，其次这样还是又违反了CAP中的A的设计，导致服务之间是不连通的。
3. 持久化的机制来带的问题，ZooKeeper 的 ZAB 协议对每一个写请求，会在每个 ZooKeeper  节点上保持写一个事务日志，同时再加上定期的将内存数据镜像（Snapshot）到磁盘来保证数据的一致性和持久性，而对于一个简单的服务发现的场景来说，这其实没有太大的必要，这个实现方案太重了。而且本身存储的数据应该是高度定制化的。
4. 消息发送应该弱依赖注册中心，而RocketMQ的设计理念也正是基于此，生产者在第一次发送消息的时候从NameServer获取到Broker地址后缓存到本地，如果NameServer整个集群不可用，短时间内对于生产者和消费者并不会产生太大影响。





## Broker保存数据

RocketMQ主要的存储文件包括CommitLog文件、ConsumeQueue文件、Indexfile文件。



![image-20220524221654719](../../images/Rocketmq/image-20220524221654719.png)





**CommitLog**：消息主体以及元数据的存储主体，存储Producer端写入的消息主体内容,消息内容不是定长的。单个文件大小默认1G, 文件名长度为20位，左边补零，剩余为起始偏移量，比如00000000000000000000代表了第一个文件，起始偏移量为0，文件大小为1G=1073741824；当第一个文件写满了，第二个文件为00000000001073741824，起始偏移量为1073741824，以此类推。消息主要是顺序写入日志文件，当文件满了，写入下一个文件。





**ConsumeQueue**：消息消费队列，引入的目的主要是提高消息消费的性能，由于RocketMQ是基于主题topic的订阅模式，消息消费是针对主题进行的，如果要遍历commitlog文件中根据topic检索消息是非常低效的。

Consumer即可根据ConsumeQueue来查找待消费的消息。其中，ConsumeQueue（逻辑消费队列）作为消费消息的索引，保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值。





ConsumeQueue文件可以看成是基于Topic的CommitLog索引文件





**IndexFile**：IndexFile（索引文件）提供了一种可以通过key或时间区间来查询消息的方法。Index文件的存储位置是：{fileName}，文件名fileName是以创建时的时间戳命名的，固定的单个IndexFile文件大小约为400M，一个IndexFile可以保存 2000W个索引，IndexFile的底层存储设计为在文件系统中实现HashMap结构，故RocketMQ的索引文件其底层实现为hash索引。







## RocketMQ怎么对文件进行读写的

RocketMQ对文件的读写巧妙地利用了操作系统的一些高效文件读写方式——`PageCache`、`顺序读写`、`零拷贝`





#### 说说什么是零拷贝?

在操作系统中，使用传统的方式，数据需要经历几次拷贝，还要经历用户态/内核态切换。





![image-20220524221943493](../../images/Rocketmq/image-20220524221943493.png)





1. 从磁盘复制数据到内核态内存；

2. 从内核态内存复制到用户态内存；

3. 然后从用户态内存复制到网络驱动的内核态内存；

4. 最后是从网络驱动的内核态内存复制到网卡中进行传输。

   

所以，可以通过零拷贝的方式，**减少用户态与内核态的上下文切换**和**内存拷贝的次数**，用来提升I/O的性能。零拷贝比较常见的实现方式是**mmap**，这种机制在Java中是通过MappedByteBuffer实现的。







![image-20220524222034627](../../images/Rocketmq/image-20220524222034627.png)

这个零拷贝先暂且搁一下，以后搞懂。



## 消息刷盘怎么实现的呢？





RocketMQ提供了两种刷盘策略：同步刷盘和异步刷盘

- 同步刷盘：在消息达到Broker的内存之后，必须刷到commitLog日志文件中才算成功，然后返回Producer数据已经发送成功。
- 异步刷盘：异步刷盘是指消息达到Broker内存后就返回Producer数据已经发送成功，会唤醒一个线程去将数据持久化到CommitLog日志文件中。





**Broker** 在消息的存取时直接操作的是内存（内存映射文件），这可以提供系统的吞吐量，但是无法避免机器掉电时数据丢失，所以需要持久化到磁盘中。

刷盘的最终实现都是使用**NIO**中的 MappedByteBuffer.force() 将映射区的数据写入到磁盘，如果是同步刷盘的话，在**Broker**把消息写到**CommitLog**映射区后，就会等待写入完成。





异步而言，只是唤醒对应的线程，不保证执行的时机，流程如图所示。





# push与poll





消费者客户端有两种方式从消息中间件获取消息并消费。严格意义上来讲，RocketMQ并没有实现PUSH模式，而是对拉模式进行一层包装，名字虽然是 Push 开头，实际在实现时，使用 Pull 方式实现。通过 Pull 不断轮询 Broker 获取消息。当不存在新消息时，Broker 会挂起请求，直到有新消息产生，取消挂起，返回新消息。





## 区别：



pull方式里，取消息的过程需要用户自己写，首先通过打算消费的topic拿到MesssageQueue的集合，遍历MessageQueue集合，然后针对每个MessageQueue批量取消息，一次取完之后，记录该队列下一次要取的开始offset,直到取完了，再换另一个MessageQueue。



push方式里，consumer把轮询过程封装了（轮询的去拉pull消息），并注册MessageListener监听器，取到消息后，唤醒MessageListener的 consumeMessage()来消费，对用户而言，感觉消息是被推送过来的







# Offset





首先来明确一下 Offset 的含义， RocketMQ 中， 一 种类型的消息会放到 一 个 Topic 里，为了能够并行， 一般一个 Topic 会有多个 Message Queue (也可以 设置成一个)， **Offset是指某个 Topic下的一条消息在某个 Message Queue里的 位置**，通过 Offset的值可以定位到这条消息，或者指示 Consumer从这条消息 开始向后继续处理 。



> 针对集群消费，offset保存在broker，在客户端使用RemoteBrokerOffsetStore。
>
> 针对广播消费，offset保存在本地，在客户端使用LocalFileOffsetStore。
>
> 保存的offset指的是下一条消息的offset，而不是消费完最后一条消息的offset。









与上面的offset不同的是

ConsumeQueue 里面记录的**Offset**是消息在 CommitLog 里面的物理存储地址。











**总结**

![img](../../images/Rocketmq/strip.png)

producer发送消息到broker之后，会将消息具体内容持久化到commitLog文件中，再分发到topic下的消费队列consume Queue，消费者提交消费请求时，broker从该consumer负责的消费队列中根据请求参数起始offset获取待消费的消息索引信息，再从commitLog中获取具体的消息内容返回给consumer。在这个过程中，consumer提交的offset为本次请求的起始消费位置，即beginOffset；consume Queue中的offset定位了commitLog中具体消息的位置。











# 源码图



![RocektMQ源码 (../../../../images/MQ/RocektMQ%25E6%25BA%2590%25E7%25A0%2581%2520(1).jpg)](../images/MQ/RocektMQ%E6%BA%90%E7%A0%81%20(1).jpg)



**总体理念**

https://mp.weixin.qq.com/s/2TeUYAodDKG0gvuuVzAdPw



**设计理念（消息持久化，高性能，高可用）**

https://mp.weixin.qq.com/s/QObTjysGvbBRULcMY1ed-w





































