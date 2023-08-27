
> [40讲insert语句的锁为什么这么多](https://funnylog.gitee.io/mysql45/40%E8%AE%B2insert%E8%AF%AD%E5%8F%A5%E7%9A%84%E9%94%81%E4%B8%BA%E4%BB%80%E4%B9%88%E8%BF%99%E4%B9%88%E5%A4%9A.html)

# insert...select语句加锁

表结构：

```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(null, 1,1);
insert into t values(null, 2,2);
insert into t values(null, 3,3);
insert into t values(null, 4,4);

create table t2 like t
```

在可重复读隔离级别下，binlog_format=statement时执行下面的 `insert` 语句，会对表 t 的所有行和间隙加锁。

```sql
insert into t2(c,d) select c,d from t;
```

## insert...select 语句加锁分析

这里加锁主要是考虑日志和数据的一致性问题。对于下面这个执行序列来说：

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/f087f4e0-6eba-4325-ad35-f2f15b6fa123)

如果session B先执行，由于这个语句会对表t主键索引加了(-∞,1]这个`next-key lock`，会在语句执行完成后，才允许session A的insert语句执行。

但如果没有锁的话，就可能出现session B的insert语句先执行，但是后写入binlog的情况。于是，在`binlog_format=statement`的情况下，binlog里面就记录了这样的语句序列：

```sql
insert into t values(-1,-1,-1);
insert into t2(c,d) select c,d from t;
```

这个语句到了备库执行，就会把 id=-1 这一行也写到表t2中，出现主备不一致。

> **注意：执行insert … select 的时候，对目标表也不是一定锁全表，而是只锁住需要访问的资源。本案例中需要访问目标表的全表资源，所以锁了全表。**

要往表t2中插入一行数据，这一行的c值是表t中c值的最大值加1。

SQL语句为 ：

```sql
insert into t2(c,d)  (select c+1, d from t force index(c) order by c desc limit 1);
```

这个语句的加锁范围，就是表t索引c上的(4,supremum]这个next-key lock和主键索引上id=4这一行。

它的执行流程也比较简单，从表t中按照索引c倒序，扫描第一行，拿到结果写入到表t2中。

因此整条语句的扫描行数是1。

这个语句执行的慢查询日志（slow log），如下图所示：

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/1b7a1ed9-1e1c-47c9-b313-fe0ea9937a62)

通过这个慢查询日志，我们看到Rows_examined=1，正好验证了执行这条语句的扫描行数为1。

# insert 循环写入

如果把上述需求改为：要往表t中插入一行数据，这一行的c值是表t中c值的最大值加1。

```sql
insert into t(c,d)  (select c+1, d from t force index(c) order by c desc limit 1);
```

慢日志中Rows_examined的值是5：

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/e3f083e3-1e0f-4452-a8c7-931c79951fbc)

explain结果为：

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/f87dde98-ba7c-4186-b937-a38f8c7d37c8)

从Extra字段可以看到“Using temporary”字样，表示这个语句用到了临时表。也就是说，执行过程中，需要把表t的内容读出来，写入临时表。

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/ca1364ae-2c75-4046-ac7d-f03022a8a71f)

这个语句执行前后，Innodb_rows_read的值增加了4。因为默认临时表是使用Memory引擎的，所以这4行查的都是表t，也就是说对表t做了全表扫描。

整个语句的执行过程为：

1. 创建临时表，表里有两个字段c和d。
2. 按照索引c扫描表t，依次取c=4、3、2、1，然后回表，读到c和d的值写入临时表。这时，Rows_examined=4。
3. 由于语义里面有limit 1，所以只取了临时表的第一行，再插入到表t中。这时，Rows_examined的值加1，变成了5。

也就是说，这个语句会导致在表t上做全表扫描，并且会给索引c上的所有间隙都加上共享的next-key lock。所以，这个语句执行期间，其他事务不能在这个表上插入数据。

至于这个语句的执行为什么需要临时表，原因是这类一边遍历数据，一边更新数据的情况，如果读出来的数据直接写回原表，就可能在遍历过程中，读到刚刚插入的记录，新插入的记录如果参与计算逻辑，就跟语义不符。

由于在实现上，这个语句没有在子查询中就直接使用limit 1，从而导致了这个语句的执行需要遍历整个表t。它的优化方法也比较简单，就是用前面介绍的方法，先insert into到临时表temp_t，这样就只需要扫描一行；然后再从表temp_t里面取出这行数据插入表t。

当然，由于这个语句涉及的数据量很小，可以考虑使用内存临时表来做这个优化。使用内存临时表优化时，语句序列的写法如下：

```sql
create temporary table temp_t(c int,d int) engine=memory;
insert into temp_t  (select c+1, d from t force index(c) order by c desc limit 1);
insert into t select * from temp_t;
drop table temp_t;
```

# insert 唯一键冲突

待补充。。。








