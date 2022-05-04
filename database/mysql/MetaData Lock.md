## MetaData Lock
在执行DDL语句之前，需要获取 MetaData Lock 锁，获取到锁的条件是该表的事务全部完成，该锁会阻塞后续对于该表的所有操作，包括读操作。

#### 解决的问题
MetaData Lock是在5.5引入的，用于解决如下问题：
* RR隔离级别下不可重读问题
在RR级别下，使用MVCC和ReadView解决记录可重复读的问题，但是当事务中使用alter增加字段时，重复读取会有不同的字段问题。引入MetaData Lock后，DDL语句需要先获取字典锁，获取字典锁条件是在等待该锁中事务全部完成后，并且会阻塞后续所有操作至字典锁释放。

* 主从复制不一致
在5.5版本之前事务中涉及到DDL的语句，不会被其他事务阻塞，在运行过程中会和其他事务交叉运行，同步到从库会产生数据不一致的情况。
```sql
-- 事务一
begin;
select * from table_name1;

-- 事务二
begin;
rename table table_name1 to table_name2;
commit;

-- 同步到从库，表结构修改完成

-- 事务一
select * from table_name1;
-- 报错，主从复制中断
```

##### MDL的分类
![25](.\image\25.jpg)
使用show processlist可以得到等待锁信息：
* Waiting for global read lock 
* Waiting for commit lock
* Waiting for schema metadata lock
* Waiting for table metadata lock
* Waiting for stored function metadata lock
* Waiting for stored procedure metadata lock
* Waiting for trigger metadata lock
* Waiting for event metadata lock


##### MDL注意事项
* DDL耗时语句应在低峰时段运行，防止阻塞住其他的操作造成连接数量升高(连接操作在一定时间内没有返回会重开连接执行)。
[记一次truncate导致的锁表处理](https://blog.csdn.net/qian_xiaoqian/article/details/53813333)   
[truncate 的问题](https://houbb.github.io/2017/02/27/mysql-truncate)


摘抄自
[关于mysql Meta Lock 机制详细讲解 ](http://blog.itpub.net/29896444/viewspace-2101567/)
[MySQL表结构变更你不可不知的Metadata Lock详解](https://www.jb51.net/article/145599.htm)