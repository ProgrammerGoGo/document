
[01讲基础架构：一条SQL查询语句是如何执行的](https://funnylog.gitee.io/mysql45/01%E8%AE%B2%E5%9F%BA%E7%A1%80%E6%9E%B6%E6%9E%84%EF%BC%9A%E4%B8%80%E6%9D%A1SQL%E6%9F%A5%E8%AF%A2%E8%AF%AD%E5%8F%A5%E6%98%AF%E5%A6%82%E4%BD%95%E6%89%A7%E8%A1%8C%E7%9A%84.html)

```这里默认MySQL使用innodb存储引擎```

首先，MySQL可以分为Server层和存储引擎层两部分。Server层包括连接器、查询缓存（MySQL8.0已删除该模块）、分析器、优化器、执行器等。

* 第一步（连接器），需要先连接到这个数据库上，连接器负责跟客户端建立连接、获取权限、维持和管理连接。通过输入连接命令`mysql -h$ip -P$port -u$user -p
`，在完成经典的TCP握手后，连接器就要开始认证登录人的身份，这个时候用的就是命令中输入的用户名和密码。

如果用户名或密码不对，你就会收到一个"Access denied for user"的错误，然后客户端程序结束执行。如果用户名密码认证通过，连接器会到权限表里面查出你拥有的权限。之后，这个连接里面的权限判断逻辑，都将依赖于此时读到的权限。这就意味着，一个用户成功建立连接后，即使你用管理员账号对这个用户的权限做了修改，也不会影响已经存在连接的权限。修改完成后，只有再新建的连接才会使用新的权限设置。（需要注意的是，客户端如果太长时间没动静，连接器就会自动将它断开）

* 第二步（查询缓存），会在查询缓存中查看是否有命中相同的SQL语句，如果命中则直接返回缓存中的结果，否则向下继续执行。（因为MySQL8.0已删除查询缓存，所以这里我们不考虑查询缓存步骤）

* 第三步（优化器），首先，MySQL需要知道你要做什么，因此需要对SQL语句做解析，包括“词法分析”和“语法分析”。

分析器先会做“词法分析”。你输入的是由多个字符串和空格组成的一条SQL语句，MySQL需要识别出里面的字符串分别是什么，代表什么。MySQL从你输入的"select"这个关键字识别出来，这是一个查询语句。它也要把字符串“T”识别成“表名T”，把字符串“ID”识别成“列ID”。做完了这些识别以后，就要做“语法分析”。根据词法分析的结果，语法分析器会根据语法规则，判断你输入的这个SQL语句是否满足MySQL语法。如果你的语句不对，就会收到“You have an error in your SQL syntax”的错误提醒，比如查询语句中的select关键字少打了开头的字母“s”，语法分析就会拦截报错。

* 第四步（执行器），MySQL通过分析器知道了你要做什么，通过优化器知道了该怎么做，于是就进入了执行器阶段，开始执行语句。

开始执行的时候，要先判断一下登录人对这个表有没有执行查询的权限，如果没有，就会返回没有权限的错误。如果有权限，就打开表继续执行。打开表的时候，执行器就会根据表的引擎定义，去使用这个引擎提供的接口。

如果查询的字段没有索引，执行流程如下：

1、调用InnoDB引擎接口取这个表的第一行，判断查询条件是否符合，如果不符合则跳过，如果符合则将这行存在 **结果集** 中；

2、调用引擎接口取“下一行”，重复相同的判断逻辑，直到取到这个表的最后一行。

3、执行器将上述遍历过程中所有满足条件的行组成的记录集作为结果集返回给客户端。

对于有索引的表，执行的逻辑也差不多。第一次调用的是“取满足条件的第一行”这个接口（通过索引树），之后循环取“满足条件的下一行”这个接口，这些接口都是引擎中已经定义好的。


![SQL执行流程](https://github.com/ProgrammerGoGo/document/blob/main/MySQL/image/MySQL%E7%9A%84%E5%9F%BA%E6%9C%AC%E6%9E%B6%E6%9E%84%E7%A4%BA%E6%84%8F%E5%9B%BE.png)










