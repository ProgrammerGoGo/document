

# redo log

> 注意：redo log 文件是Innodb引擎特有的，不属于MySQL的server层。

如果每一次的更新操作都需要写进磁盘，然后磁盘也要找到对应的那条记录，然后再更新，整个过程IO成本、查找成本都很高。为了解决这个问题，MySQL的设计者就用了类似酒店掌柜粉板的思路来提升更新效率。

MySQL里经常说到的WAL技术，WAL的全称是Write-Ahead Logging，它的关键点就是先写日志，再写磁盘。  

具体来说，当有一条记录需要更新的时候，InnoDB引擎就会先把记录写到`redo log`里面，并更新内存，这个时候更新就算完成了。同时，InnoDB引擎会在适当的时候，将这个操作记录更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做。

InnoDB的`redo log`是固定大小的，比如可以配置为一组4个文件，每个文件的大小是1GB，那么`redo log`总共就可以记录4GB的操作。从头开始写，写到末尾就又回到开头循环写，如下面这个图所示。

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/c7238637-de01-49eb-9ce7-29d15d2c30b9)

`write pos`是当前记录的位置，一边写一边后移，写到第3号文件末尾后就回到0号文件开头。`checkpoint`是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件。

`write pos`和`checkpoint`之间的是`redo log`还空着的部分，可以用来记录新的操作。如果`write pos`追上`checkpoint`，表示`redo log`满了，这时候不能再执行新的更新，得停下来先擦掉一些记录（即刷脏页，flush），把`checkpoint`推进一下。

有了`redo log`，InnoDB就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为`crash-safe`。

# binlog

> binglog 是MySQL的server层的日志文件。

因为最开始MySQL里并没有InnoDB引擎。MySQL自带的引擎是MyISAM，但是MyISAM没有crash-safe的能力，`binlog`日志只能用于归档。而InnoDB是另一个公司以插件形式引入MySQL的，既然只依靠`binlog`是没有`crash-safe`能力的，所以InnoDB使用另外一套日志系统——也就是`redo log`来实现`crash-safe`能力。

# redo log 和 binlog 日志的不同点：
`redo log` 和 `binlog` 日志的不同点：

* `redo log` 是InnoDB引擎特有的；`binlog` 是MySQL的Server层实现的，所有引擎都可以使用。

* `redo log` 是物理日志，记录的是“在某个数据页上做了什么修改”；`binlog` 是逻辑日志，记录的是这个语句的原始逻辑，比如“给ID=2这一行的c字段加1 ”。

* `redo log` 是循环写的，空间固定会用完；`binlog` 是可以追加写入的。“追加写”是指 `binlog` 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。






