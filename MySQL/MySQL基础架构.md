MySQL基础架构

<img src="D:\workplace\temp\iamge\mysql基础架构.png" alt="mysql基础架构" style="zoom:50%;" />

连接器：

负责身份认证和权限相关认证

查询缓存：

执行查询语句的时候，会先查询缓存

分析器：

没有命中缓存的话，SQL语句就会经过分析器，分析器主要是用来分析SQL语句是来干嘛的，分析器分为两步

- 词法分析，提取SQL语句中的关键字，如select，需要查询的表，字段名，查询条件等
- 语法分析，分析SQL是否符合MySQL语法

优化器

经过分析器之后，SQL是符合MySQL语法规范的，优化器的作用就是以它认为最优的执行方法去执行SQL，如走索引还是走全表扫描

执行器

当确定了执行方案之后，会去调用存储引擎的接口，拿到数据

日志模块

redo log

在MySQL中如果每一次的更新操作都需要写进磁盘，然后磁盘也要找到对应的那条记录，然后再更新，整个过程IO成本过高，所以为了解决这个问题引入了redo log。

具体来说，当有一条记录需要更新的时候，InnoDB引擎就会先把记录写到redo log里，然后再更新内存，这个时候更新内存就算完成了。同时，InnoDB引擎会在适当的时候，将这个操作记录更新到磁盘里面，这个更新往往是在系统比较空闲的时候做的。

Write-Ahead Logging

当有一条记录需要更新的时候，InnoDB引擎会先写redo log，并更新内存的数据，这个时候更新就算完成。同时，InnoDB引擎会在适当的时候，将操作记录更新到磁盘里面。

<img src="C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210819105304634.png" alt="image-20210819105304634" style="zoom:80%;" />

InnoDB的redo log可以配置好几个文件，write_pos是当前记录的位置，checkpoint是当前已经更新到磁盘中的位置，当write_pos追上checkpoint时，需要停下来让checkpoint往后移。

两阶段提交

深绿色执行器做的，浅绿色是存储引擎做的。

<img src="C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210819104756702.png" alt="image-20210819104756702" style="zoom: 67%;" />

1. 更新内存
2. 先写redolog，处于prepare阶段

2. 然后在写binlog
3. 最后让redolog提交。

如果在1-2，2-3之间崩溃，可以放弃事务，2-3崩溃了，可以通过log进行恢复事务







