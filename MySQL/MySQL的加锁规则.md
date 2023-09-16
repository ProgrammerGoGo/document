
> [21讲为什么我只改一行的语句，锁这么多](https://funnylog.gitee.io/mysql45/21%E8%AE%B2%E4%B8%BA%E4%BB%80%E4%B9%88%E6%88%91%E5%8F%AA%E6%94%B9%E4%B8%80%E8%A1%8C%E7%9A%84%E8%AF%AD%E5%8F%A5%EF%BC%8C%E9%94%81%E8%BF%99%E4%B9%88%E5%A4%9A.html)
> [用动态的观点看加锁](https://funnylog.gitee.io/mysql45/30%E8%AE%B2%E7%AD%94%E7%96%91%E6%96%87%E7%AB%A0%EF%BC%88%E4%BA%8C%EF%BC%89%EF%BC%9A%E7%94%A8%E5%8A%A8%E6%80%81%E7%9A%84%E8%A7%82%E7%82%B9%E7%9C%8B%E5%8A%A0%E9%94%81.html)

# 查看行锁

```sql
select * from performance_schema.data_locks;
```

<img width="1433" alt="屏幕快照 2023-09-16 下午4 33 45" src="https://github.com/ProgrammerGoGo/document/assets/98639494/04ed41f7-2183-43cb-bc13-6e3229d9e428">

# 表结构

```sql
drop table if exists t;
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

# 案例十：in查询非唯一索引加锁

```sql
select id from t where c in(5,20,10) lock in share mode;
```

explain结果：

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/17c77035-cf09-43a6-8a12-48a3e6b03bd7)

可以看到，这条in语句使用了索引c并且rows=3，说明这三个值都是通过B+树搜索定位的。

在查找c=5的时候，先锁住了(0,5]。但是因为c不是唯一索引，为了确认还有没有别的记录c=5，就要向右遍历，找到c=10才确认没有了，这个过程满足优化2，所以加了间隙锁(5,10)。

同样的，执行c=10这个逻辑的时候，加锁的范围是(5,10] 和 (10,15)；执行c=20这个逻辑的时候，加锁的范围是(15,20] 和 (20,25)。

通过这个分析，我们可以知道，这条语句在索引c上加的三个记录锁的顺序是：先加c=5的记录锁，再加c=10的记录锁，最后加c=20的记录锁。

# 案例十一：

```sql
-- 查询一
select id from t where c in(5,20,10) lock in share mode;

-- 查询二
select id from t where c in(5,20,10) order by c desc for update;
```

间隙锁是不互锁的，但是这两条语句都会在索引c上的c=5、10、20这三行记录上加记录锁。

由于语句二里面是`order by c desc`， 这三个记录锁的加锁顺序，是先锁c=20，然后c=10，最后是c=5。

也就是说，这两条语句要加锁相同的资源，但是加锁顺序相反。当这两条语句并发执行的时候，就可能出现死锁。

死锁信息如下：

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/1da7090e-2029-46d2-8061-2c512a5b11c9)

几个关键信息：

1. 这个结果分成三部分：
   - (1) TRANSACTION，是第一个事务的信息；
   - (2) TRANSACTION，是第二个事务的信息；
   - WE ROLL BACK TRANSACTION (1)，是最终的处理结果，表示回滚了第一个事务。

2. 第一个事务的信息中：
   - WAITING FOR THIS LOCK TO BE GRANTED，表示的是这个事务在等待的锁信息；
   - index c of table `test`.`t`，说明在等的是表t的索引c上面的锁；
   - lock mode S waiting 表示这个语句要自己加一个读锁，当前的状态是等待中；
   - Record lock说明这是一个记录锁；
   - n_fields 2表示这个记录是两列，也就是字段c和主键字段id；
   - 0: len 4; hex 0000000a; asc ;;是第一个字段，也就是c。值是十六进制a，也就是10；
   - 1: len 4; hex 0000000a; asc ;;是第二个字段，也就是主键id，值也是10；
   - 这两行里面的asc表示的是，接下来要打印出值里面的“可打印字符”，但10不是可打印字符，因此就显示空格。
   - 第一个事务信息就只显示出了等锁的状态，在等待(c=10,id=10)这一行的锁。
   - 当然你是知道的，既然出现死锁了，就表示这个事务也占有别的锁，但是没有显示出来。别着急，我们从第二个事务的信息中推导出来。

3. 第二个事务显示的信息要多一些：
   - “ HOLDS THE LOCK(S)”用来显示这个事务持有哪些锁；
   - index c of table `test`.`t` 表示锁是在表t的索引c上；
   - hex 0000000a和hex 00000014表示这个事务持有c=10和c=20这两个记录锁；
   - WAITING FOR THIS LOCK TO BE GRANTED，表示在等(c=5,id=5)这个记录锁。

从上面这些信息中可以知道：

1. “lock in share mode”的这条语句，持有c=5的记录锁，在等c=10的锁；

2. “for update”这个语句，持有c=20和c=10的记录锁，在等c=5的记录锁。

因此导致了死锁。这里，可以得到两个结论：

1. 由于锁是一个个加的，要避免死锁，对同一组资源，要按照尽量相同的顺序访问；

2. 在发生死锁的时刻，`for update` 这条语句占有的资源更多，回滚成本更大，所以InnoDB选择了回滚成本更小的`lock in share mode`语句，来回滚。

# 案例十二：锁等待分析

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/7c8a96c8-5998-49f1-a658-4a03f87f64a6)

由于session A并没有锁住c=10这个记录，所以session B删除id=10这一行是可以的。但是之后，session B再想insert id=10这一行回去就不行了。锁等待信息如下：

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/57c47a33-30a2-4c8e-aaa2-3a96fcafe94f)

几个关键信息：

1. index PRIMARY of table `test`.`t` ，表示这个语句被锁住是因为表t主键上的某个锁。

2. lock_mode X locks gap before rec insert intention waiting 这里有几个信息：
   - insert intention表示当前线程准备插入一个记录，这是一个插入意向锁。为了便于理解，你可以认为它就是这个插入动作本身。
   - gap before rec 表示这是一个间隙锁，而不是记录锁。

3. 那么这个gap是在哪个记录之前的呢？接下来的0~4这5行的内容就是这个记录的信息。

4. n_fields 5也表示了，这一个记录有5列：
   - 0: len 4; hex 0000000f; asc ;;第一列是主键id字段，十六进制f就是id=15。所以，这时我们就知道了，这个间隙就是id=15之前的，因为id=10已经不存在了，它表示的就是(5,15)。
   - 1: len 6; hex 000000000513; asc ;;第二列是长度为6字节的事务id，表示最后修改这一行的是trx id为1299的事务。
   - 2: len 7; hex b0000001250134; asc % 4;; 第三列长度为7字节的回滚段信息。可以看到，这里的acs后面有显示内容(%和4)，这是因为刚好这个字节是可打印字符。
后面两列是c和d的值，都是15。

因此，由于delete操作把id=10这一行删掉了，原来的两个间隙(5,10)、(10,15）变成了一个(5,15)。

session A执行完select语句后，什么都没做，但它加锁的范围由原来的(10,15]变成了(5,15]，所以session B的第二个语句会阻塞。

所谓“间隙”，其实根本就是由“**这个间隙右边的那个记录**”定义的。

# 案例十三：锁等待分析

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/759c60ff-ae98-4993-a2bd-571a35ca7a4d)

session A的加锁范围是索引c上的 (5,10]、(10,15]、(15,20]、(20,25]和(25,suprenum]。

之后session B的第一个update语句，要把c=5改成c=1，你可以理解为两步：

1. 插入(c=1, id=5)这个记录；

2. 删除(c=5, id=5)这个记录。

索引c上(5,10)间隙是由这个间隙右边的记录，也就是c=10定义的。所以通过session B这个操作后，session A的加锁范围变成了 (1,10]、(10,15]、(15,20]、(20,25]和(25,suprenum]：

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/af48c659-b62e-4dd7-b71b-0f2a24ee76d7)

接下来session B要执行 update t set c = 5 where c = 1这个语句，一样地可以拆成两步：

1. 插入(c=5, id=5)这个记录；

2. 删除(c=1, id=5)这个记录。

第一步试图在已经加了间隙锁的(1,10)中插入数据，所以就被堵住了。













