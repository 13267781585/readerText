# Redo Log

<img src=".\image\92.jpg" alt="92" />

## redolog作用

### 宕机恢复

因为bin log是追加写，宕机重启后无法判断哪些数据有没有刷盘，而redo log是循环写，只会存放没有刷盘的事务，宕机后只需要把redo log加载到内存中就行。

### 提高数据刷盘效率

* 数据页随机io->redolog顺序io
* redolog数据小
* 降低刷盘频率

## 触发redo log刷盘时机

mysql通过参数innodb_flush_log_at_trx_commit控制redolog刷盘策略：

* 1：每次提交事务刷盘
* 2：只写入内存，刷盘时机由操作系统决定
* 0：定期刷新(1s)
因此可能发生刷盘时机如下：
* Redo log buffer空间不足
* 事务提交
* 后台线程
后台存在master线程定时(1秒)把redolog刷新到磁盘
* checkpoint
定期把redolog日志和脏页刷新到磁盘，减少宕机恢复的时间
* mysql实例关闭

## Q&A

### 为什么用redolog同步数据延迟比binlog小？

因为事务中只要修改了数据就会记录redolog，binlog只有在事务提交后才记录，使用redolog做主从同步，在数据修改后就可以进行同步，不需要等到事务提交后，所以主从不一致延迟大大减少。
<https://ost.51cto.com/posts/20428>
