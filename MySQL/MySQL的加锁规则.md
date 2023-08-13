
> [21讲为什么我只改一行的语句，锁这么多](https://funnylog.gitee.io/mysql45/21%E8%AE%B2%E4%B8%BA%E4%BB%80%E4%B9%88%E6%88%91%E5%8F%AA%E6%94%B9%E4%B8%80%E8%A1%8C%E7%9A%84%E8%AF%AD%E5%8F%A5%EF%BC%8C%E9%94%81%E8%BF%99%E4%B9%88%E5%A4%9A.html)

# 表结构

```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```

# 加锁规则

两个“原则”、两个“优化”和一个“bug”：

* 原则1：加锁的基本单位是`next-key lock`。（`next-key lock`是前开后闭区间）
* 原则2：查找过程中访问到的对象才会加锁。
* 优化1：索引上的等值查询，给唯一索引加锁的时候，`next-key lock`退化为行锁。
* 优化2：索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，`next-key lock`退化为间隙锁。
* 一个bug：唯一索引上的范围查询会访问到不满足条件的第一个值为止。

# 案例一：主键查询不存在的值

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/b4d06e2c-c826-4750-8965-645341484235)

由于表t中没有id=7的记录，加锁如下：

1. 根据原则1，加锁单位是next-key lock，session A加锁范围就是(5,10]；

2. 同时根据优化2，这是一个等值查询(id=7)，而id=10不满足查询条件，next-key lock退化成间隙锁，因此最终加锁的范围是(5,10)。

所以，session B要往这个间隙里面插入id=8的记录会被锁住，但是session C修改id=10这行是可以的。

# 案例二：非唯一索引等值锁



























