
> [【Kafka】Kafka的重复消费和消息丢失问题](https://blog.csdn.net/dl962454/article/details/128087396)

# 重复消费

## 重复消费的原因

消息重复消费的根本原因都在于：**已经消费了数据，但是offset没有成功提交。**

* 原因1：消费者宕机、重启或者被强行kill进程，导致消费者消费的offset没有提交。
* 原因2：设置 `enable.auto.commit` 为 true (即消费者使用自动提交offset，自动提交和手动提交详见[Kafka中位移提交那些事儿](https://github.com/ProgrammerGoGo/document/blob/main/MQ/Kafka/Kafka%E4%B8%AD%E4%BD%8D%E7%A7%BB%E6%8F%90%E4%BA%A4%E9%82%A3%E4%BA%9B%E4%BA%8B%E5%84%BF.md))，如果在关闭消费者进程之前，取消了消费者的订阅，则有可能部分offset没提交，下次重启会重复消费。
* 原因3（重复消费最常见的原因）：消费后的数据，当offset还没有提交时，Partition就断开连接。比如，通常会遇到消费的数据，处理很耗时，导致超过了Kafka的 `session timeout.ms` 时间（0.10.x版本默认是30秒），那么就会触发reblance重平衡，此时可能存在消费者offset没提交，会导致重平衡后重复消费。

## 解决方案

* 提高消费者的处理速度。例如：对消息处理中比较耗时的步骤可通过异步的方式进行处理、利用多线程处理等。在缩短单条消息消费的同时，根据实际场景可将 `max.poll.interval.ms` 值设置大一点，避免不必要的Rebalance。可根据实际消息速率适当调小 `max.poll.records` 的值。
* 引入消息去重机制。例如：生成消息时，在消息中加入唯一标识符如消息id等。在消费端，可以保存最近的 `max.poll.records` 条消息id到redis或mysql表中，这样在消费消息时先通过查询去重后，再进行消息的处理。
* 保证消费者逻辑幂等。[一文理解如何实现接口的幂等性](https://mp.weixin.qq.com/s?__biz=MzUyNzgyNzAwNg%3D%3D&idx=1&mid=2247484349&scene=21&sn=b54c0819bc100db816cda52d11476401#wechat_redirect)


# 消息丢失



