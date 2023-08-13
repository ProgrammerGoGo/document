
`MDL`（metadata lock）锁不需要显式使用，在访问一个表的时候会被自动加上。

`MDL` 的作用是，保证读写的正确性。你可以想象一下，如果一个查询正在遍历一个表中的数据，而执行期间另一个线程对这个表结构做变更，删了一列，那么查询线程拿到的结果跟表结构对不上，肯定是不行的。

因此，在MySQL 5.5版本中引入了MDL，当对一个表做增删改查操作的时候，加`MDL`读锁；当要对表做结构变更操作的时候，加`MDL`写锁。

* 读锁之间不互斥，因此你可以有多个线程同时对一张表增删改查。

* 读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。因此，如果有两个线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行。

虽然`MDL`锁是系统默认会加的，但却是你不能忽略的一个机制。比如下面这个例子，我经常看到有人掉到这个坑里：给一个小表加个字段，导致整个库挂了。

你肯定知道，给一个表加字段，或者修改字段，或者加索引，需要扫描全表的数据。在对大表操作的时候，你肯定会特别小心，以免对线上服务造成影响。而实际上，即使是小表，操作不慎也会出问题。我们来看一下下面的操作序列，假设表t是一个小表。

> 备注：这里的实验环境是MySQL 5.6。而MySQL 5.7版本修改了MDL的加锁策略，所以就不能复现这个场景了。

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/577209d6-5c27-4c71-9f1e-a3c362eea402)

我们可以看到session A先启动，这时候会对表t加一个MDL读锁。由于session B需要的也是MDL读锁，因此可以正常执行。

之后session C会被blocked，是因为session A的MDL读锁还没有释放，而session C需要MDL写锁，因此只能被阻塞。

如果只有session C自己被阻塞还没什么关系，但是之后所有要在表t上新申请MDL读锁的请求也会被session C阻塞。前面我们说了，所有对表的增删改查操作都需要先申请MDL读锁，就都被锁住，等于这个表现在完全不可读写了。

如果某个表上的查询语句频繁，而且客户端有重试机制，也就是说超时后会再起一个新session再请求的话，这个库的线程很快就会爆满。

你现在应该知道了，**事务中的MDL锁，在语句执行开始时申请，但是语句结束后并不会马上释放，而会等到整个事务提交后再释放。**


> 在MySQL 5.7版本下复现这个场景。

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/60d665f6-5691-40fb-96d3-75166b990d26)

session A 通过`lock table`命令持有表t的`MDL`写锁，而session B的查询需要获取`MDL`读锁。所以，session B进入等待状态。

这类问题的处理方式，就是找到谁持有MDL写锁，然后把它kill掉。

但是，由于在`show processlist`的结果里面，session A的Command列是“Sleep”，导致查找起来很不方便。不过有了performance_schema和sys系统库以后，就方便多了。（MySQL启动时需要设置performance_schema=on，相比于设置为off会有10%左右的性能损失)

通过查询sys.schema_table_lock_waits这张表，我们就可以直接找出造成阻塞的process id，把这个连接用kill 命令断开即可。

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/b47d5e4d-5bd9-45ff-a95c-91b38c6b1bea)
