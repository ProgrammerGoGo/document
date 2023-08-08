
# 什么是幻读？

建表语句：

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

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/87c48f76-0c52-4cfd-b935-cbfd38539a82)





# 如何解决幻读？

间隙锁（Gap Lock）





