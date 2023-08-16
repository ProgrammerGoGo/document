
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

# 案例二：非唯一索引等值查询

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/56b19773-821e-4b05-87fa-3168770e37ec)

这里session A要给索引c上c=5的这一行加上读锁。

1. 根据原则1，加锁单位是next-key lock，因此会给(0,5]加上next-key lock。

2. 要注意c是普通索引，因此仅访问c=5这一条记录是不能马上停下来的，需要向右遍历，查到c=10才放弃。根据原则2，访问到的都要加锁，因此要给(5,10]加next-key lock。

3. 但是同时这个符合优化2：等值判断，向右遍历，最后一个值不满足c=5这个等值条件，因此退化成间隙锁(5,10)。

4. 根据原则2 ，**只有访问到的对象才会加锁**，这个查询使用覆盖索引，并不需要访问主键索引，所以主键索引上没有加任何锁，这就是为什么session B的update语句可以执行完成。

但session C要插入一个(7,7,7)的记录，就会被session A的间隙锁(5,10)锁住。

需要注意，在这个例子中，`lock in share mode`只锁覆盖索引，但是如果是`for update`就不一样了。 执行 `for update`时，系统会认为你接下来要更新数据，因此会顺便给主键索引上满足条件的行加上行锁。

这个例子说明，锁是加在索引上的；同时，它给我们的指导是，如果你要用`lock in share mode`来给行加读锁避免数据被更新的话，就必须得绕过覆盖索引的优化，在查询字段中加入索引中不存在的字段。比如，将session A的查询语句改成`select d from t where c=5 lock in share mode`。

# 案例三：主键索引范围查询

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/2a28b426-1ef2-4f58-9f7b-fb5c912d2d69)

现在我们就用前面提到的加锁规则，来分析一下session A 会加什么锁呢？

1. 开始执行的时候，要找到第一个id=10的行，因此本该是next-key lock(5,10]。 根据优化1， 主键id上的等值条件，退化成行锁，只加了id=10这一行的行锁。

2. 范围查找就往后继续找，找到id=15这一行停下来，因此需要加next-key lock(10,15]。

所以，session A这时候锁的范围就是主键索引上，行锁id=10和next-key lock(10,15]。

注意：首次session A定位查找id=10的行的时候，是当做等值查询来判断的，而向右扫描到id=15的时候，用的是范围查询判断。

# 案例四：非唯一索引范围查询

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/e4ad4586-fbd7-447f-8e70-5716c0f19d14)

session A用字段c来判断，加锁规则跟案例三唯一的不同是：在第一次用c=10定位记录的时候，索引c上加了(5,10]这个next-key lock后，由于索引c是非唯一索引，没有优化规则，也就是说不会蜕变为行锁，因此最终sesion A加的锁是，索引c上的(5,10] 和(10,15] 这两个next-key lock。

所以从结果上来看，sesson B要插入（8,8,8)的这个insert语句时就被堵住了。

这里需要扫描到c=15才停止扫描，是合理的，因为InnoDB要扫到c=15，才知道不需要继续往后找了。

# 案例五：唯一索引范围锁bug

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/77fc3808-eba4-4c2a-8954-00645556ef48)

session A是一个范围查询，按照原则1的话，应该是索引id上只加(10,15]这个next-key lock，并且因为id是唯一键，所以循环判断到id=15这一行就应该停止了。

但是实现上，InnoDB会往前扫描到第一个不满足条件的行为止，也就是id=20。而且由于这是个范围扫描，因此索引id上的(15,20]这个next-key lock也会被锁上。

所以session B要更新id=20这一行，是会被锁住的。同样地，session C要插入id=16的一行，也会被锁住。

# 案例六：非唯一索引上存在"等值"的例子

表t插入一条新记录。

```sql
insert into t values(30,10,30);
```
此时索引c如下：

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/7b4df11f-2b91-47de-8b40-d8115a03028b)

虽然有两个c=10，但是它们的主键值id是不同的（分别是10和30），因此这两个c=10的记录之间，也是有间隙的。

图中画出了索引c上的主键id。为了跟间隙锁的开区间形式进行区别，用(c=10,id=30)这样的形式，来表示索引上的一行。

这次用delete语句来验证。注意，delete语句加锁的逻辑，其实跟`select ... for update` 是类似的，也就是我在文章开始总结的两个“原则”、两个“优化”和一个“bug”。

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/103f551a-3753-4cec-afb1-da5679d34d62)

这时，session A在遍历的时候，先访问第一个c=10的记录。同样地，根据原则1，这里加的是(c=5,id=5)到(c=10,id=10)这个next-key lock。

然后，session A向右查找，直到碰到(c=15,id=15)这一行，循环才结束。根据优化2，这是一个等值查询，向右查找到了不满足条件的行，所以会退化成(c=10,id=10) 到 (c=15,id=15)的间隙锁。

也就是说，这个delete语句在索引c上的加锁范围，就是下图中蓝色区域覆盖的部分。

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/40639975-b27b-40cf-9244-2b25e77e53c9)

这个蓝色区域左右两边都是虚线，表示开区间，即(c=5,id=5)和(c=15,id=15)这两行上都没有锁。

# 案例七：limit 语句加锁

在案例六的基础上执行：

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/f87194f9-0e07-4095-8130-9adf7e968c3d)

session A的delete语句加了 limit 2。因为表t里c=10的记录其实只有两条，因此加不加limit 2，删除的效果都是一样的，但是加锁的效果却不同。可以看到，session B的insert语句执行通过了，跟案例六的结果不同。

这是因为，案例七里的delete语句明确加了limit 2的限制，因此在遍历到(c=10, id=30)这一行之后，满足条件的语句已经有两条，循环就结束了。

因此，索引c上的加锁范围就变成了从（c=5,id=5)到（c=10,id=30)这个前开后闭区间，如下图所示：

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/405c0616-030a-4a58-9a79-56d8c95d0f32)

可以看到，(c=10,id=30）之后的这个间隙并没有在加锁范围里，因此insert语句插入c=12是可以执行成功的。

这个例子对我们实践的指导意义就是，在删除数据的时候尽量加limit。这样不仅可以控制删除数据的条数，让操作更安全，还可以减小加锁的范围。

# 案例八：一个死锁的例子

> 此案例目的是说明：next-key lock实际上是间隙锁和行锁加起来的结果。

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/e49fdf6e-9ec2-4cd5-bdcc-bfa9c3f4cb31)

现在，我们按时间顺序来分析一下为什么是这样的结果。

1. session A 启动事务后执行查询语句加lock in share mode，在索引c上加了next-key lock(5,10] 和间隙锁(10,15)；

2. session B 的update语句也要在索引c上加next-key lock(5,10] ，进入锁等待；

3. 然后session A要再插入(8,8,8)这一行，被session B的间隙锁锁住。由于出现了死锁，InnoDB让session B回滚。

你可能会问，session B的next-key lock不是还没申请成功吗？

其实是这样的，session B的“加next-key lock(5,10] ”操作，实际上分成了两步，先是加(5,10)的间隙锁，加锁成功（间隙锁不是互斥的，所以session A锁住的间隙，session B仍然能再次上锁 [幻读与间隙锁](https://github.com/ProgrammerGoGo/document/blob/main/MySQL/MySQL%E4%B8%AD%E7%9A%84%E5%B9%BB%E8%AF%BB%E6%98%AF%E4%BB%80%E4%B9%88%EF%BC%9F.md)）；然后加c=10的行锁，这时候才被锁住的。

也就是说，在分析加锁规则的时候可以用`next-key lock`来分析。但是要知道，具体执行的时候，是要分成间隙锁和行锁两段来执行的。

# 案例九：锁分析

为什么session B的insert操作，会被锁住呢？

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/8f910c0f-0c04-4ca3-b641-0b6331fad2ee)

session A的select语句加锁分析：

1. 由于是order by c desc，第一个要定位的是索引c上“最右边的”c=20的行（从左向右找，等值查询定位到25），所以会加上间隙锁(20,25)和next-key lock (15,20]。

2. 在索引c上向左遍历，要扫描到c=10才停下来，所以next-key lock会加到(5,10]，这正是阻塞session B的insert语句的原因。

3. 在扫描过程中，c=20、c=15、c=10这三行都存在值，由于是select *，所以会在主键id上加三个行锁。

因此，session A 的select语句锁的范围就是：

1. 索引c上 (5, 25)；

2. 主键索引上id=10、15、20三个行锁。







