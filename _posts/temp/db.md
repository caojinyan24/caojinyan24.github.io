
解决方法：

1、查看数据库当前的进程，看一下有无正在执行的慢SQL记录线程。

mysql> show  processlist;
2、查看当前的事务

当前运行的所有事务

mysql> SELECT * FROM information_schema.INNODB_TRX;
当前出现的锁

mysql> SELECT * FROM information_schema.INNODB_LOCKs;
锁等待的对应关系

mysql> SELECT * FROM information_schema.INNODB_LOCK_waits;
解释：看事务表INNODB_TRX，里面是否有正在锁定的事务线程，看看ID是否在show processlist里面的sleep线程中，如果是，就证明这个sleep的线程事务一直没有commit或者rollback而是卡住了，我们需要手动kill掉。

搜索的结果是在事务表发现了很多任务，这时候最好都kill掉。

3、批量删除事务表中的事务

我这里用的方法是：通过information_schema.processlist表中的连接信息生成需要处理掉的MySQL连接的语句临时文件，然后执行临时文件中生成的指令。

mysql>  select concat('KILL ',id,';') from information_schema.processlist where user='cms_bokong';
+------------------------+
| concat('KILL ',id,';') |
+------------------------+
| KILL 10508;            |
| KILL 10521;            |
| KILL 10297;            |
+------------------------+
18 rows in set (0.00 sec)
当然结果不可能只有3个，这里我只是举例子。参考链接上是建议导出到一个文本，然后执行文本。而我是直接copy到记事本处理掉 ‘|’，粘贴到命令行执行了。都可以。

kill掉以后再执行SELECT * FROM information_schema.INNODB_TRX; 就是空了。

这时候系统就正常了

故障排查

mysql都是autocommit配置mysql> select @@autocommit;
+--------------+
| @@autocommit |
+--------------+
|            0 |
+--------------+
1 row in set (0.00 sec)
如果是0 ，则改为1
mysql> set global autocommit=1;

mysql的引擎检查，可以检查一下数据库引擎是不是InnoDB（mysql5.5.5以前默认是MyISAM，mysql5.5.5以后默认是InnoDB）show ENGINES； #检查命令
如果不是的话改为 InnoDB :
查看表使用的存储引擎

show table status from db_name where name='table_name';

修改表的存储引擎

alter table table_name engine=innodb;