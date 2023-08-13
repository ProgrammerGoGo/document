
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


# 如何解决幻读？

间隙锁（Gap Lock）





