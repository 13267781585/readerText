# MySQL 

## 排名内置函数   
* RANK()  
 并列跳跃排名，并列即相同的值，相同的值保留重复名次，遇到下一个不同值时，跳跃到总共的排名。 1 2 2 4 5  
* DENSE_RANK()   
并列连续排序，并列即相同的值，相同的值保留重复名次，遇到下一个不同值时，依然按照连续数字排名。 1 2 2 3 4   
* ROW_NUMBER()   
连续排名，即使相同的值，依旧按照连续数字进行排名。 1 2 3 4

## 日期函数   
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

## 精度函数   
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

## mysql并发数高
大多数情况下，开发会使用多线程并行处理方法加快任务进度，减少DB执行时间，但是很多人都错估了DB承受能力导致mysql经常有running high情况，而running high 服务器最显著的表示CPU load、idle 异常，并发数设置过高并不能给程序带来任何收益，反而恰恰相反，增加了执行时间。

并发数高影响：

      1.并发线程数太高，线程调度会严重消耗资源，当运行线程数超过32个（innodb_thread_concurrency）时，请求线程会在mysql innodb层等待，处于waiting for innodb queue状态，同时线程会有大量上下文切换，消耗CPU。

       2.不同线程访问相同资源，会出现资源竞争或者mysql事务锁，从而出现SQL拥塞现象。mysql中会有mutex来保护共享资源，当有线程需要使用这些共享资源时，会请求获得mutex量，请求成功的线程进入临界区，而请求失败的线程只能等待它释放这个mutex，所以并发线程越多，mutex的竞争越激烈；当某些并发线程访问更新一表中数据时，而恰巧mysql 隔离级别是REPEATABLE READ，会有间隙锁，线程高，锁竞争多，SQL拥塞，执行时间较长。

资源是有限，开启过多进程并不会提升性能，反而会下降。在满足业务要求下，尽可能降低资源消耗，需求时间和性能资源必须有取舍，取舍优先级必须性能资源大于时间消耗。

在确定任务完成时间需求，开发合理分解任务后，可以这样计算并发数：

     情况一、业务要求时间是T1，单线程运行完是T2，T1>T2，那单线程运行就额可以满足要求，没有必要采用多线程并发处理；

     情况二、如果T2>T1,取N=T2/T1,以这个为基准，测试N个并发下运行时间是否小于T1，如果N个并发还不能满足业务时间要求，则加大一个并发进行再次测试，找到一个满足业务需求的最小并发数。

## mysql从库延迟
mysql db从库延迟一直是困扰着我们noc dba，几乎每天都会有某个域的延迟告警，大多数情况下，延迟原因为以下两种：   1.批量数据增删改 2.大表加索引操作3.表无主键，一次更新多条记录，对于某些域DB而言，开发可能认为从库没业务延迟可以忽略，但是无规则不成方圆，只有我们按照规范约束数据库行为才能提供更好运维环境。

1.批量数据增删改操作可以拆分，按照每次处理1000条，sleep 0.3s模式循环操作，目前我们线上mysql DB都是5.5.44版本，复制操作是单线程，普通情况一个单线程每秒最大操作数据行数在8k-10k行左右

2.对于BI系统而言为了谋求插入速度，常常会对表先insert再创建索引，几百万数据表创建索引延迟数据是非常可观的，所以先创索引再插入，保证线上DB稳定。

3.表无主键，一次操作100+条记录时，而这条SQL是全表扫描/低效的索引扫描，从库复制时会产生延迟，因为我们线上DB是采用row模式binlog复制，主库更新100条记录，从库就需执行100次update语句，而每个SQL效率极其低下。



## mysql服务端/客户端通信协议
半双工 只有一端向另一端发送数据，只有数据全部接受完才能响应(max_allowed_packet客户端和服务端每次通讯最大数据包大小)
* TCP/IP

* 命名管道和共享内存(仅适合单机)
1. 命名管道
服务器 --enable-named-pipe  客户端 --protocol=pipe
2. 共享内存
服务器 --shared-memory 客户端 --protocol=memory

* Unix域套接字(仅适合unix单机)  

* 为什么mysql通信协议要选用半双工的形式呢？   
1. 实现简单，容易维护
2. 数据库查询的机制是一个sql请求一个数据回复，半双工就可以满足要求，数据库的性能瓶颈是在sql的执行时间，和数据库的底层实现等有关，和通讯机制关系不大。(个人理解)

## mysql无法在同一个sql中同时对一张表进行查询和更新
```sql
update table_name set a = (select b from table_name where id = 1);  -- 执行报错
-- 可以通过临时表的方式绕过限制(查询被当做临时表)
update table_name inner join (select b from table_name where id = 1) table_name1 on c set table_name.a = table_name1.b;
```
## on where having的区别
* on 是在多表连接作为条件去判断两条记录是否匹配的条件
* where 是在返回结果之前对数据进行过滤的条件，不可以使用聚合函数，可以使用所有列
* having 一般搭配 group by 使用(也可以单独使用)，对返回的数据集进行每一个组的过滤，可以使用聚合函数，因为是对返回的集合的操作，所以只能操作数据集中有的列
* 执行顺序 on -> where -> having

## BufferPool缓冲池
https://blog.csdn.net/wuhenyouyuyouyu/article/details/93377605  
https://blog.csdn.net/m0_37892044/article/details/121795586

## MVCC 
https://blog.csdn.net/qq_38538733/article/details/88902979


## 行数据的最大值
一条记录占用的最大存储空间是有限制的，除了 BLOB 或者 TEXT 类型的列之外(根据具体行格式判断，Compact存放前768字节，Dynamic等不存放)，其他所有的列（不包括隐藏列和记录头信息）占用的字节长度加起来不能超过 65535 个字节(数据页大小16k)。
https://zhuanlan.zhihu.com/p/53413773

## mysql统计表和实际不符
统计表采用定期同步的方式去落盘数据，会发生统计表(information_schema.tables/show table status等)和实际情况不符的情况。
* 解决方法：将时间设置为实时更新
```sql
-- 全局设置实时更新
SET GLOBAL information_schema_stats_expiry=0;
SET @@GLOBAL.information_schema_stats_expiry=0;

-- 会话实时更新
SET SESSION information_schema_stats_expiry=0;
SET @@SESSION.information_schema_stats_expiry=0;
```

## 删除数据的方式
* delete from table_name
1. delete 是 DML 语句，只删除数据不删除结构，通过一行一行删除并且记录事务日志，可以回滚。
2. 执行会触发trigger
3. 删除数据后innodb和myisam都不会立刻释放空间，只会标记记录为删除状态，删除的空间可重用(可使用optmize table table_name 整理空间碎片)。
    * myisam
    删除前
    ![21](.\image\21.jpg)
    删除后
    ![22](.\image\22.jpg)
    可以看出虽然行数和平均的行数长度都为0，但是数据的长度没有改变，只是作为空间碎片的数据重复使用，并没有释放磁盘空间。
    * innodb
    删除前
    ![23](.\image\23.jpg)
    删除后
    ![24](.\image\24.jpg)
    可以看出虽然行数和平均的行数长度都为0，但是数据的长度没有改变(删除后的空间在innodb引擎不作为空间碎片)。
4. 执行后不重置 auto_increment

* truncate table table_name
1. truncate 是 DDL 语句，速度快，执行后无法回滚。
2. 不触发trigger
3. innodb和myisam引擎都一致，执行后立即释放磁盘空间
4. 执行后重置 auto_increment

* drop table table_name
1. drop 是 DDL 语句
2. 执行后立即释放磁盘空间
3. 执行后删除表的数据、结构、触发器等，依赖于该表的存储过程/函数状态变为 invalid 

## 自适应哈希索引(AHI)
自适应哈希索引是mysql内部一种加快查询的优化措施，由innodb引擎自己判断构建，只能针对等值查询。   
[MySQL AHI 实现解析](https://cloud.tencent.com/developer/article/1004516)

## 其他索引
## 多维索引
* [空间填充曲线](https://www.cnblogs.com/tgzhu/p/8286616.html)
* 网格文件
* 分段散列
* 多键索引
* kd-树
* 四叉树
* R-树
* 位图索引   
[【高级数据库】第二章 第04讲 多维索引](https://blog.csdn.net/qq_36426650/article/details/103324224)

## 行式存储 VS 列式存储 ? [OLTP VS OLAP](https://github.com/Vonng/ddia/blob/master/ch3.md#%E4%BA%8B%E5%8A%A1%E5%A4%84%E7%90%86%E8%BF%98%E6%98%AF%E5%88%86%E6%9E%90)
## 行式存储(OLTP)
* 需要频繁更新和插入数据
* 记录列不多，没有复杂的分析场景

在高层次上，我们看到存储引擎分为两大类：针对 事务处理（OLTP） 优化的存储引擎和针对 在线分析（OLAP） 优化的存储引擎。这两类使用场景的访问模式之间有很大的区别：  
* OLTP 系统通常面向最终用户，这意味着系统可能会收到大量的请求。为了处理负载，应用程序在每个查询中通常只访问少量的记录。应用程序使用某种键来请求记录，存储引擎使用索引来查找所请求的键的数据。硬盘查找时间往往是这里的瓶颈。
* 数据仓库和类似的分析系统会少见一些，因为它们主要由业务分析人员使用，而不是最终用户。它们的查询量要比 OLTP 系统少得多，但通常每个查询开销高昂，需要在短时间内扫描数百万条记录。硬盘带宽（而不是查找时间）往往是瓶颈，列式存储是针对这种工作负载的日益流行的解决方案。

## [列式存储(OLAP)](https://github.com/Vonng/ddia/blob/master/ch3.md#%E4%BA%8B%E5%8A%A1%E5%A4%84%E7%90%86%E8%BF%98%E6%98%AF%E5%88%86%E6%9E%90)
## 优势
* 针对不同数据类型压缩数据
* 不读取无效数据，适用于列非常多的场景
* 提高有效数据读取效率，减少不需要数据的读取
* 对数据按不同纬度排序处理应对不同场景的分析操作+数据备份   

## 中间件
* Hbase
* Druid
* Cassandra
* clickhouse   
[列存数据库，不只是列式存储](https://cn.kyligence.io/blog/%E5%88%97%E5%AD%98%E6%95%B0%E6%8D%AE%E5%BA%93%EF%BC%8C%E4%B8%8D%E5%8F%AA%E6%98%AF%E5%88%97%E5%BC%8F%E5%AD%98%E5%82%A8/)

## SQL语句执行顺序
![66](.\image\66.jpg)

## 锁错误排查方式
![68](.\image\68.jpg)

## 基本概念
* 聚簇索引
* 索引覆盖
* 索引下推
* MMR
![87](.\image\87.jpg)
![88](.\image\88.jpg)


## mysql问题合集
* char和varchar
* datetime和timestamp
* uuid适合作为主键吗？
* 索引原则
* mysql索引数据结果选型
* 为什么用自增id作为主键
* 平衡二叉树和红黑色
* 数据库连接使用后不关闭的弊端？
* 短索引
* SQL注入
* Statement和PrepareStatement
* mysql常见引擎
* 三大范式
![80](.\image\80.jpg)
![81](.\image\81.jpg)
![82](.\image\82.jpg)
![83](.\image\83.jpg)
![84](.\image\84.jpg)
![85](.\image\85.jpg)
![86](.\image\86.jpg)

## 查询语句执行过程
![89](.\image\89.jpg)

## 主从同步
![90](.\image\90.jpg)
