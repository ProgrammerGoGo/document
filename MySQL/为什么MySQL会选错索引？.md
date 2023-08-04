
> [10讲MySQL为什么有时候会选错索引](https://funnylog.gitee.io/mysql45/10%E8%AE%B2MySQL%E4%B8%BA%E4%BB%80%E4%B9%88%E6%9C%89%E6%97%B6%E5%80%99%E4%BC%9A%E9%80%89%E9%94%99%E7%B4%A2%E5%BC%95.html)

优化器选择索引的目的，是找到一个最优的执行方案，并用最小的代价去执行语句。

优化器会结合**扫描行数**、**是否使用临时表**、**是否排序**等因素进行综合判断执行代价。

建表语句：
```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`),
  KEY `b` (`b`)
) ENGINE=InnoDB；
```
插入10万行记录，取值按整数递增，即：(1,1,1)，(2,2,2)，(3,3,3) 直到(100000,100000,100000)。
```sql
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=100000)do
    insert into t values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();
```

# 问题1：扫描行数是怎么判断的？
MySQL在真正开始执行语句之前，并不能精确地知道满足这个条件的记录有多少条，而只能根据统计信息来估算记录数。

这个统计信息就是索引的“区分度”。显然，一个索引上不同的值越多，这个索引的区分度就越好。而一个索引上不同的值的个数，我们称之为“基数”（cardinality）。也就是说，这个基数越大，索引的区分度越好。

我们可以使用`show index`方法，看到一个索引的基数。如下图所示，就是表t的`show index` 的结果 。虽然这个表的每一行的三个字段值都是一样的，但是在统计信息中，这三个索引的基数值并不同，而且其实都不准确。

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/6c44d5c8-1237-4e31-ada9-70ac8083d0cc)

MySQL是通过采样统计索引的基数的。

因为把整张表取出来一行行统计，虽然可以得到精确的结果，但是代价太高了，所以只能选择“采样统计”。

采样统计的时候，InnoDB默认会选择N个数据页，统计这些页面上的不同值，得到一个平均值，然后乘以这个索引的页面数，就得到了这个索引的基数。

而数据表是会持续更新的，索引统计信息也不会固定不变。所以，当变更的数据行数超过1/M的时候，会自动触发重新做一次索引统计。

在MySQL中，有两种存储索引统计的方式，可以通过设置参数innodb_stats_persistent的值来选择：

* 设置为on的时候，表示统计信息会持久化存储。这时，默认的N是20，M是10。
* 设置为off的时候，表示统计信息只存储在内存中。这时，默认的N是8，M是16。

# 案例1


# 案例2



