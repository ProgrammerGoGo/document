







`redo log` 和 `binlog` 日志的不同点：

* `redo log` 是InnoDB引擎特有的；`binlog` 是MySQL的Server层实现的，所有引擎都可以使用。

* `redo log` 是物理日志，记录的是“在某个数据页上做了什么修改”；`binlog` 是逻辑日志，记录的是这个语句的原始逻辑，比如“给ID=2这一行的c字段加1 ”。

* `redo log` 是循环写的，空间固定会用完；`binlog` 是可以追加写入的。“追加写”是指 `binlog` 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。






