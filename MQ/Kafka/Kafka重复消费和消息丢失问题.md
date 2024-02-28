
> [【Kafka】Kafka的重复消费和消息丢失问题](https://blog.csdn.net/dl962454/article/details/128087396)

# 重复消费

## 重复消费的原因

消息重复消费的根本原因都在于：**已经消费了数据，但是offset没有成功提交。**

* 原因1：消费者宕机、重启或者被强行kill进程，导致消费者消费的offset没有提交。
* 原因2：设置 `enable.auto.commit` 为 true (即消费者使用自动提交offset，自动提交和手动提交详见[Kafka中位移提交那些事儿](https://github.com/ProgrammerGoGo/document/blob/main/MQ/Kafka/Kafka%E4%B8%AD%E4%BD%8D%E7%A7%BB%E6%8F%90%E4%BA%A4%E9%82%A3%E4%BA%9B%E4%BA%8B%E5%84%BF.md))，如果在关闭消费者进程之前，取消了消费者的订阅，则有可能部分offset没提交，下次重启会重复消费。
* 原因3（**重复消费最常见的原因**）：消费后的数据，当offset还没有提交时，Partition就断开连接。比如，通常会遇到消费的数据，处理很耗时，导致超过了Kafka的 `session timeout.ms` 时间（0.10.x版本默认是30秒），那么就会触发reblance重平衡，此时可能存在消费者offset没提交，会导致重平衡后重复消费。

## 解决方案

* 提高消费者的处理速度。例如：对消息处理中比较耗时的步骤可通过异步的方式进行处理、利用多线程处理等。在缩短单条消息消费的同时，根据实际场景可将 `max.poll.interval.ms` 值设置大一点，避免不必要的Rebalance。可根据实际消息速率适当调小 `max.poll.records` 的值。
* 引入消息去重机制。例如：生成消息时，在消息中加入唯一标识符如消息id等。在消费端，可以保存最近的 `max.poll.records` 条消息id到redis或mysql表中，这样在消费消息时先通过查询去重后，再进行消息的处理。
* 保证消费者逻辑幂等。[一文理解如何实现接口的幂等性](https://mp.weixin.qq.com/s?__biz=MzUyNzgyNzAwNg%3D%3D&idx=1&mid=2247484349&scene=21&sn=b54c0819bc100db816cda52d11476401#wechat_redirect)
* 手动提交offset。

# 消息丢失

> [如何确保消息不会丢失](https://learn.lianglianglee.com/%e4%b8%93%e6%a0%8f/%e6%b6%88%e6%81%af%e9%98%9f%e5%88%97%e9%ab%98%e6%89%8b%e8%af%be/05%20%20%e5%a6%82%e4%bd%95%e7%a1%ae%e4%bf%9d%e6%b6%88%e6%81%af%e4%b8%8d%e4%bc%9a%e4%b8%a2%e5%a4%b1.md)
> [](https://zhuanlan.zhihu.com/p/307480336)
消息丢失需要从生产者、消费者和broker三个方向来分析。

一条消息从生产到消费完成这个过程，可以划分三个阶段。

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/66ad44a3-b2a2-417c-a02a-ee324c85e629)

* 生产阶段: 在这个阶段，从消息在 Producer 创建出来，经过网络传输发送到 Broker 端。
* 存储阶段: 在这个阶段，消息在 Broker 端存储，如果是集群，消息会在这个阶段被复制到其他的副本上。
* 消费阶段: 在这个阶段，Consumer 从 Broker 上拉取消息，经过网络传输发送到 Consumer 上。

这三个阶段中每个阶段都有可能会丢失消息。

## 生产阶段

在生产阶段，消息队列通过最常用的请求确认机制，来保证消息的可靠传递：当你的代码调用发消息方法时，消息队列的客户端会把消息发送到 Broker，Broker 收到消息后，会给客户端返回一个确认响应，表明消息已经收到了。客户端收到响应后，完成了一次正常消息的发送。

只要 Producer 收到了 Broker 的确认响应，就可以保证消息在 **生产阶段** 不会丢失。有些消息队列在长时间没收到发送确认响应后，会自动重试，如果重试再失败，就会以返回值或者异常的方式告知用户。所以发送消息时，需要注意，正确处理返回值或者捕获异常，就可以保证这个阶段的消息不会丢失。以 Kafka 为例，我们看一下如何可靠地发送消息：

同步发送时，只要注意捕获异常即可。

```java
try {
    RecordMetadata metadata = producer.send(record).get();
    System.out.println(" 消息发送成功。");
} catch (Throwable e) {
    System.out.println(" 消息发送失败！");
    System.out.println(e);
}
```

异步发送时，则需要在回调方法里进行检查。这个地方是需要特别注意的，很多丢消息的原因就是，我们使用了异步发送，却没有在回调中检查发送结果。

```java
producer.send(record, (metadata, exception) -> {
    if (metadata != null) {
        System.out.println(" 消息发送成功。");
    } else {
        System.out.println(" 消息发送失败！");
        System.out.println(exception);
    }
});
```


### Kafka的ACK机制
* acks=0，producer不等待broker的响应，效率最高，但是消息很可能会丢。
* acks=1，leader broker收到消息后，不等待其他follower的响应，即返回ack。也可以理解为ack数为1。此时，如果follower还没有收到leader同步的消息leader就挂了，那么消息会丢失。按照上图中的例子，如果leader收到消息，成功写入PageCache后，会返回ack，此时producer认为消息发送成功。但此时，按照上图，数据还没有被同步到follower。如果此时leader断电，数据会丢失。
* acks=-1，leader broker收到消息后，挂起，等待所有ISR列表中的follower返回结果后，再返回ack。-1等效与all。这种配置下，只有leader写入数据到pagecache是不会返回ack的，还需要所有的ISR返回“成功”才会触发ack。如果此时断电，producer可以知道消息没有被发送成功，将会重新发送。如果在follower收到数据以后，成功返回ack，leader断电，数据将存在于原来的follower中。在重新选举以后，新的leader会持有该部分数据。

数据从leader同步到follower，需要2步：  
* 数据从pageCache被刷盘到disk。因为只有disk中的数据才能被同步到replica。
* 数据同步到replica，并且replica成功将数据写入PageCache。在producer得到ack后，哪怕是所有机器都停电，数据也至少会存在于leader的磁盘内。

上面第三点提到了ISR的列表的follower，需要配合另一个参数才能更好的保证ack的有效性。ISR是Broker维护的一个“可靠的follower列表”，in-sync Replica列表，broker的配置包含一个参数：min.insync.replicas。该参数表示ISR中最少的副本数。如果不设置该值，ISR中的follower列表可能为空。此时相当于acks=1。

## 存储阶段

Broker丢失消息是由于Kafka本身的原因造成的，kafka为了得到更高的性能和吞吐量，将数据异步批量的存储在磁盘中。消息的刷盘过程，为了提高性能，减少刷盘次数，kafka采用了批量刷盘的做法。即，按照一定的消息量，和时间间隔进行刷盘。这种机制也是由于linux操作系统决定的。将数据存储到linux操作系统种，会先存储到页缓存（Page cache）中，按照时间或者其他条件进行刷盘（从page cache到file），或者通过fsync命令强制刷盘。数据在page cache中时，如果系统挂掉，数据会丢失。

broker收到消息后，需要将消息写入磁盘的log文件中，但是并不是马上写，因为我们知道，生产者发送消息后，消费者那边需要马上获取，如果broker要写入磁盘，那么消费者拉取消息，broker还要从log文件中获取消息，这显然是不合理的，所以kafka引入了(page cache)页缓存。

page cache是磁盘和broker之间的消息映射关系，它是基于内存的，当broker收到消息后，会将消息写入page cache，然后由操作系统进行刷盘，将page cache中的数据写入磁盘。

如果broker发生故障，那么此时page cache的数据就会丢失，broker端可以设置刷盘的参数，比如多久刷盘一次，不过这个参数不建议去修改，最好的方案还是设置多副本，一个分区设置几个副本，当broker故障的时候，如果还有其他副本，那么数据就不会丢失。


## 消费者

Consumer消费消息有下面几个步骤：  
* 接收消息
* 处理消息
* 反馈“处理完毕”（commited）

Consumer自动提交的机制是根据一定的时间间隔，将收到的消息进行commit。commit过程和消费消息的过程是异步的。也就是说，可能存在消费过程未成功（比如抛出异常），commit消息已经提交了。此时消息就丢失了。





