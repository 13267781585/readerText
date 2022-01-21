## MySQL 

### 排名内置函数   
* RANK()  
 并列跳跃排名，并列即相同的值，相同的值保留重复名次，遇到下一个不同值时，跳跃到总共的排名。 1 2 2 4 5  
* DENSE_RANK()   
并列连续排序，并列即相同的值，相同的值保留重复名次，遇到下一个不同值时，依然按照连续数字排名。 1 2 2 3 4   
* ROW_NUMBER()   
连续排名，即使相同的值，依旧按照连续数字进行排名。 1 2 3 4

### 日期函数   
* date_add()   
* date_sub()
```sql
set @dt = now();

select date_add(@dt, interval 1 day); -- add 1 day
select date_add(@dt, interval 1 hour); -- add 1 hour
select date_add(@dt, interval 1 minute); -- ...
select date_add(@dt, interval 1 second);
select date_add(@dt, interval 1 microsecond);
select date_add(@dt, interval 1 week);
select date_add(@dt, interval 1 month);
select date_add(@dt, interval 1 quarter);
select date_add(@dt, interval 1 year);

select date_add(@dt, interval -1 day); -- sub 1 day

 select date_add(@dt, interval '01:15:30' hour_second);
 select date_add(@dt, interval '1 01:15:30' day_second);
```
* datediff(date1,date2)   
```sql
select datediff('2008-08-08', '2008-08-01'); -- 7
select datediff('2008-08-01', '2008-08-08'); -- -7
```

### 精度函数   
* round(number,num_digits)四舍五入   
```sql
    number：需要四舍五入的数字
    num_digits：按此位数对 number 参数进行四舍五入
    0 表示取整
    1 表示保留一位小数，2 表示保留两位小数，依此类推
    -1 表示十位数取整，-2 表示百位数取整，依次类推

    SELECT ROUND(100.3465,2),ROUND(100,2),ROUND(0.6,2),ROUND(114.6,-1);

    100.35,100，0.6,110
```

### mysql并发数高
大多数情况下，开发会使用多线程并行处理方法加快任务进度，减少DB执行时间，但是很多人都错估了DB承受能力导致mysql经常有running high情况，而running high 服务器最显著的表示CPU load、idle 异常，并发数设置过高并不能给程序带来任何收益，反而恰恰相反，增加了执行时间。

并发数高影响：

      1.并发线程数太高，线程调度会严重消耗资源，当运行线程数超过32个（innodb_thread_concurrency）时，请求线程会在mysql innodb层等待，处于waiting for innodb queue状态，同时线程会有大量上下文切换，消耗CPU。

       2.不同线程访问相同资源，会出现资源竞争或者mysql事务锁，从而出现SQL拥塞现象。mysql中会有mutex来保护共享资源，当有线程需要使用这些共享资源时，会请求获得mutex量，请求成功的线程进入临界区，而请求失败的线程只能等待它释放这个mutex，所以并发线程越多，mutex的竞争越激烈；当某些并发线程访问更新一表中数据时，而恰巧mysql 隔离级别是REPEATABLE READ，会有间隙锁，线程高，锁竞争多，SQL拥塞，执行时间较长。

资源是有限，开启过多进程并不会提升性能，反而会下降。在满足业务要求下，尽可能降低资源消耗，需求时间和性能资源必须有取舍，取舍优先级必须性能资源大于时间消耗。

在确定任务完成时间需求，开发合理分解任务后，可以这样计算并发数：

     情况一、业务要求时间是T1，单线程运行完是T2，T1>T2，那单线程运行就额可以满足要求，没有必要采用多线程并发处理；

     情况二、如果T2>T1,取N=T2/T1,以这个为基准，测试N个并发下运行时间是否小于T1，如果N个并发还不能满足业务时间要求，则加大一个并发进行再次测试，找到一个满足业务需求的最小并发数。

### mysql从库延迟
mysql db从库延迟一直是困扰着我们noc dba，几乎每天都会有某个域的延迟告警，大多数情况下，延迟原因为以下两种：1.批量数据增删改 2.大表加索引操作3.表无主键，一次更新多条记录，对于某些域DB而言，开发可能认为从库没业务延迟可以忽略，但是无规则不成方圆，只有我们按照规范约束数据库行为才能提供更好运维环境。

1.批量数据增删改操作可以拆分，按照每次处理1000条，sleep 0.3s模式循环操作，目前我们线上mysql DB都是5.5.44版本，复制操作是单线程，普通情况一个单线程每秒最大操作数据行数在8k-10k行左右

2.对于BI系统而言为了谋求插入速度，常常会对表先insert再创建索引，几百万数据表创建索引延迟数据是非常可观的，所以先创索引再插入，保证线上DB稳定。

3.表无主键，一次操作100+条记录时，而这条SQL是全表扫描/低效的索引扫描，从库复制时会产生延迟，因为我们线上DB是采用row模式binlog复制，主库更新100条记录，从库就需执行100次update语句，而每个SQL效率极其低下。



### mysql服务端/客户端通信协议
* 半双工 只有一端向另一端发送数据，只有数据全部接受完才能响应
 

### mysql无法在同一个sql中同时对一张表进行查询和更新
```sql
update table_name set a = (select b from table_name where id = 1);  -- 执行报错
-- 可以通过临时表的方式绕过限制(查询被当做临时表)
update table_name inner join (select b from table_name where id = 1) table_name1 on c set table_name.a = table_name1.b;
```
### on where having的区别
* on 是在多表连接作为条件去判断两条记录是否匹配的条件
* where 是在返回结果之前对数据进行过滤的条件，不可以使用聚合函数，可以使用所有列
* having 一般搭配 group by 使用(也可以单独使用)，对返回的数据集进行每一个组的过滤，可以使用聚合函数，因为是对返回的集合的操作，所以只能操作数据集中有的列
* 执行顺序 on -> where -> having