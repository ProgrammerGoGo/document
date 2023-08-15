
> [20讲幻读是什么，幻读有什么问题](https://funnylog.gitee.io/mysql45/20%E8%AE%B2%E5%B9%BB%E8%AF%BB%E6%98%AF%E4%BB%80%E4%B9%88%EF%BC%8C%E5%B9%BB%E8%AF%BB%E6%9C%89%E4%BB%80%E4%B9%88%E9%97%AE%E9%A2%98.html)


> 建表语句：
> 
> ```sql
> CREATE TABLE `t` (
>  `id` int(11) NOT NULL,
>   `c` int(11) DEFAULT NULL,
>   `d` int(11) DEFAULT NULL,
>   PRIMARY KEY (`id`),
>   KEY `c` (`c`)
> ) ENGINE=InnoDB;
>  
> insert into t values(0,0,0),(5,5,5),
> (10,10,10),(15,15,15),(20,20,20),(25,25,25);
> ```

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/87c48f76-0c52-4cfd-b935-cbfd38539a82)


# 什么是幻读？

* 在可重复读隔离级别下，普通的查询是快照读，是不会看到别的事务插入的数据的。因此，幻读在**“当前读”**下才会出现。
* 上面 session B 的修改结果，被 session A 之后的 select 语句用“当前读”看到，不能称为幻读。幻读仅专指**“新插入的行”**。


# 间隙锁

产生幻读的原因是，行锁只能锁住行，但是新插入记录这个动作，要更新的是记录之间的“间隙”。因此，为了解决幻读问题，InnoDB只好引入新的锁，也就是**间隙锁**(Gap Lock)。

顾名思义，间隙锁，锁的就是两个值之间的空隙。比如表t，初始化插入了6个记录，这就产生了7个间隙。

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/690ba665-0406-4305-b999-b2722a3ad7b4)

这样，当执行 `select * from t where d=5 for update`的时候，就不止是给数据库中已有的6个记录加上了行锁，还同时加了7个间隙锁。这样就确保了无法再插入新的记录。

也就是说这时候，在一行行扫描的过程中，不仅将给行加上了行锁，还给行两边的空隙，也加上了间隙锁。

数据行可以加上锁的实体，数据行之间的间隙，也是可以加上锁的实体。但是间隙锁跟之前碰到过的锁都不太一样。

比如行锁，分成读锁和写锁。这两种类型行锁有冲突关系。

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/04f1f966-3d6b-4a57-aee7-140c94314d6d)

也就是说，跟行锁有冲突关系的是“另外一个行锁”。

但是间隙锁不一样，**跟间隙锁存在冲突关系的，是“往这个间隙中插入一个记录”这个操作。** 间隙锁之间都不存在冲突关系。

举个例子：

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/6c251d12-50e9-47b5-84b9-b9cd72fcc436)

这里session B并不会被堵住。因为表t里并没有c=7这个记录，因此session A加的是间隙锁(5,10)。而session B也是在这个间隙加的间隙锁。它们有共同的目标，即：保护这个间隙，不允许插入值。但它们之间是不冲突的。

> [加锁规则](https://github.com/ProgrammerGoGo/document/blob/main/MySQL/MySQL%E7%9A%84%E5%8A%A0%E9%94%81%E8%A7%84%E5%88%99.md)

间隙锁和行锁合称`next-key lock`，每个`next-key lock`是前开后闭区间。也就是说，我们的表t初始化以后，如果用`select * from t for update`要把整个表所有记录锁起来，就形成了7个`next-key lock`，分别是 (-∞,0]、(0,5]、(5,10]、(10,15]、(15,20]、(20, 25]、(25, +suprenum]。

因为+∞是开区间，实现上，InnoDB给每个索引加了一个不存在的最大值suprenum，这样才符合我们前面说的“都是前开后闭区间”。

# 间隙锁带来的问题

















