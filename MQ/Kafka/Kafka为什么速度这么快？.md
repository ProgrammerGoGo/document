
> [Kafka 为什么那么快的 6 个原因！](https://blog.csdn.net/zl1zl2zl3/article/details/107963699)
> [Kafka为什么速度这么快？.md](https://strikefreedom.top/archives/why-kafka-is-so-fast)

一、顺序读写

二、Page Cache

Broker 收到数据后，写磁盘时只是将数据写入 Page Cache，并不保证数据一定完全写入磁盘。从这一点看，可能会造成机器宕机时，Page Cache 内的数据未写入磁盘从而造成数据丢失。但是这种丢失只发生在机器断电等造成操作系统不工作的场景，而这种场景完全可以由 Kafka 层面的 Replication 机制去解决。如果为了保证这种情况下数据不丢失而强制将 Page Cache 中的数据 Flush 到磁盘，反而会降低性能。也正因如此，Kafka 虽然提供了 flush.messages 和 flush.ms 两个参数将 Page Cache 中的数据强制 Flush 到磁盘，但是 Kafka 并不建议使用。


三、零拷贝

Producer 生产的数据持久化到 broker，采用 mmap 文件映射，实现顺序的快速写入

Customer 从 broker 读取数据，采用 sendfile，将磁盘文件读到 OS 内核缓冲区后，转到 NIO buffer进行网络发送，减少 CPU 消耗

四、分区分段+索引 partition

五、批量读写

六、批量压缩

