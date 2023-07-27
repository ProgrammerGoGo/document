


[02讲日志系统：一条SQL更新语句是如何执行的](https://funnylog.gitee.io/mysql45/02%E8%AE%B2%E6%97%A5%E5%BF%97%E7%B3%BB%E7%BB%9F%EF%BC%9A%E4%B8%80%E6%9D%A1SQL%E6%9B%B4%E6%96%B0%E8%AF%AD%E5%8F%A5%E6%98%AF%E5%A6%82%E4%BD%95%E6%89%A7%E8%A1%8C%E7%9A%84.html)


SQL更新语句执行流程和 [一条SQL查询语句的执行流程](https://github.com/ProgrammerGoGo/document/blob/main/MySQL/%E4%B8%80%E6%9D%A1SQL%E6%9F%A5%E8%AF%A2%E8%AF%AD%E5%8F%A5%E7%9A%84%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.md) 相似，不同点主要在“执行器”和“存储引擎”的操作流程上。

要想了解更新语句 `update T set c=c+1 where ID=2;` 的执行流程，需要先了解 [redo log和binlog日志分别是什么](https://github.com/ProgrammerGoGo/document/blob/main/MySQL/redo%20log%20%E5%92%8C%20binlog%20%E6%97%A5%E5%BF%97%E5%88%86%E5%88%AB%E6%98%AF%E4%BB%80%E4%B9%88.md) 。

有了对这两个日志的概念性理解，我们再来看执行器和InnoDB引擎在执行这个简单的update语句时的内部流程。

1、执行器先找引擎取ID=2这一行。ID是主键，引擎直接用索引树搜索找到这一行。如果ID=2这一行数据所在的 **数据页** 本来就在存储引擎的内存中，就直接返回给执行器；否则，需要先从磁盘读入存储引擎内存，然后再返回。

2、执行器拿到引擎给的行数据，把这个值加上1，比如原来是N，现在就是N+1，得到新的一行数据，再调用引擎接口写入这行新数据。

3、引擎将这行新数据更新到存储引擎内存中，同时将这个更新操作记录到 `redo log` 里面，此时 `redo log` 处于prepare状态。然后告知执行器执行完成了，随时可以提交事务。

4、执行器生成这个操作的 `binlog`，并把 `binlog` 写入磁盘。

5、执行器调用引擎的提交事务接口，引擎把刚刚写入的 `redo log` 改成提交（commit）状态，更新完成。

这里我给出这个update语句的执行流程图，图中浅色框表示是在InnoDB内部执行的，深色框表示是在执行器中执行的。













