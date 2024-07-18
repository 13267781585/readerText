# MySQL

## 事务表
在 MySQL 中，`innodb_trx` 表是 `information_schema` 数据库的一部分，它提供了当前正在进行的所有 InnoDB 事务的详细信息。这个表对于数据库管理员在监控和诊断事务相关问题时非常有用。下面是 `innodb_trx` 表中各个字段的详细解析：

### 字段列表及其描述

1. **trx_id**:
   - 类型: `VARCHAR(18)`
   - 描述: 事务的唯一标识符。这是一个内部生成的序列号，用于唯一标识当前实例中的事务。

2. **trx_state**:
   - 类型: `VARCHAR(13)`
   - 描述: 事务的当前状态。可能的值包括 'RUNNING', 'LOCK WAIT', 'ROLLING BACK' 和 'COMMITTING'。

3. **trx_started**:
   - 类型: `DATETIME`
   - 描述: 事务开始的时间。

4. **trx_requested_lock_id**:
   - 类型: `VARCHAR(81)`
   - 描述: 如果事务当前正在等待锁，这个字段显示等待的锁的标识符。

5. **trx_wait_started**:
   - 类型: `DATETIME`
   - 描述: 事务开始等待锁的时间。如果事务没有在等待锁，该字段为 NULL。

6. **trx_weight**:
   - 类型: `BIGINT`
   - 描述: 事务的权重，表示事务对资源的占用程度。权重越大，事务涉及的行更多，或者事务持有更多的锁。

7. **trx_mysql_thread_id**:
   - 类型: `BIGINT`
   - 描述: 执行事务的 MySQL 线程的 ID。

8. **trx_query**:
   - 类型: `LONGTEXT`
   - 描述: 事务中当前正在执行的 SQL 查询。如果没有查询正在执行，则为 NULL。

9. **trx_operation_state**:
   - 类型: `VARCHAR(64)`
   - 描述: 当前正在执行的事务操作的状态。这可以提供关于事务当前活动的更多细节。

10. **trx_tables_in_use**:
    - 类型: `BIGINT`
    - 描述: 当前在事务中使用的表的数量。

11. **trx_tables_locked**:
    - 类型: `BIGINT`
    - 描述: 当前在事务中锁定的表的数量。

12. **trx_lock_structs**:
    - 类型: `BIGINT`
    - 描述: 事务已经分配的锁结构的数量。

13. **trx_lock_memory_bytes**:
    - 类型: `BIGINT`
    - 描述: 事务分配给锁的内存量（以字节为单位）。

14. **trx_rows_locked**:
    - 类型: `BIGINT`
    - 描述: 事务当前锁定的行数。

15. **trx_rows_modified**:
    - 类型: `BIGINT`
    - 描述: 事务开始以来修改的行数。

16. **trx_concurrency_tickets**:
    - 类型: `BIGINT`
    - 描述: 事务在进入 InnoDB 内部队列之前可以进行的操作数。

17. **trx_isolation_level**:
    - 类型: `VARCHAR(16)`
    - 描述: 事务的隔离级别，如 'READ UNCOMMITTED', 'READ COMMITTED', 'REPEATABLE READ', 或 'SERIALIZABLE'。

18. **trx_unique_checks**:
    - 类型: `INT`
    - 描述: 是否启用了唯一性检查。

19. **trx_foreign_key_checks**:
    - 类型: `INT`
    - 描述: 是否启用了外键约束检查。

20. **trx_last_foreign_key_error**:
    - 类型: `VARCHAR(256)`
    - 描述: 最后一次外键错误的描述（如果有）。

21. **trx_adaptive_hash_latched**:
    - 类型: `INT`
    - 描述: 是否在自适应哈希索引上设置了闩锁。

22. **trx_adaptive_hash_timeout**:
    - 类型: `INT`
    - 描述: 自适应哈希索引的超时设置。

## 索引压缩

### mysiam
pack_keys 是一个与MyISAM存储引擎相关的表选项，用于控制MyISAM表的索引键值是否应该被压缩。

#### `pack_keys` 的设置选项

`pack_keys` 参数可以有以下几种设置：

- `pack_keys=0`：禁用键压缩。这是默认设置，意味着索引键不会被压缩。
- `pack_keys=1`：启用键压缩。在这种模式下，MyISAM会尝试压缩所有索引键，除了那些用作行指针的部分。
- `pack_keys=DEFAULT`：这将只压缩字符串类型的键（CHAR, VARCHAR, TEXT），而不压缩数值类型的键（如INT, DECIMAL等）。

#### 工作原理

当`pack_keys`设置为压缩时（`pack_keys=1`或`DEFAULT`），对于字符串类型的键，MyISAM通过只存储字符串的变化部分来实现压缩。这类似于前缀压缩，其中只记录与前一个键值不同的部分。

#### 性能影响

- **空间节省**：启用`pack_keys`可以减少索引的存储空间，这对于磁盘空间有限的系统可能非常有用。
- **性能**：因为每个值都依赖前面的值，所以查找的时候无法使用二分法只能从头开始扫描，还有倒序扫描速度下降，因此等值查询平均都需要扫描半个索引块。

#### 使用场景

- **大量文本数据**：对于包含大量文本字段的表，启用`pack_keys`可能会显著减少索引的大小。
- **读密集型应用**：如果应用主要是读取操作，并且磁盘I/O是性能瓶颈，压缩索引可能有助于提高性能。
- **空间敏感型应用**：在磁盘空间非常宝贵的环境中，压缩索引可以是一个有价值的优化。

### 设置方法

在创建或修改MyISAM表时，可以通过SQL语句设置`pack_keys`：

```sql
CREATE TABLE my_table (
    id INT,
    data VARCHAR(255),
    INDEX data_index (data)
) ENGINE=MyISAM PACK_KEYS=1;

ALTER TABLE my_table PACK_KEYS=0;
```

### innodb
在InnoDB存储引擎中，使用`ROW_FORMAT=COMPRESSED`选项可以启用表数据和索引的压缩。这种压缩方式旨在减少数据在磁盘上的存储空间，从而可能减少I/O操作，提高数据加载的速度。下面详细解析`ROW_FORMAT=COMPRESSED`是如何实现压缩的：

#### 压缩机制

1. **页压缩**：
   - InnoDB存储数据和索引在磁盘上的基本单位是“页”，默认大小为16KB。
   - 当表设置为`COMPRESSED`行格式时，InnoDB会尝试将这些16KB的页压缩到更小的页中，如8KB、4KB或更小，具体取决于`KEY_BLOCK_SIZE`的设置。
   - 压缩使用的是zlib库，这是一个广泛使用的数据压缩库。

2. **压缩过程**：
   - 当数据页（包括索引页）首次写入磁盘前，InnoDB会尝试将其压缩到指定的`KEY_BLOCK_SIZE`。
   - 如果压缩成功，压缩后的页将被写入磁盘。如果压缩失败（即数据无法压缩到小于或等于指定的`KEY_BLOCK_SIZE`），则原始的16KB页将被写入磁盘。

3. **压缩页的管理**：
   - 每个压缩页在内存中都有一个对应的未压缩的16KB的“修改缓冲区”页。
   - 当压缩页被读取进内存时，它会被解压缩回16KB的格式，以便进行处理。
   - 当对页进行修改时，修改首先应用于内存中的未压缩页。之后，在将页写回磁盘之前，页将再次被压缩。

#### 性能和存储考虑

- **存储效率**：使用`ROW_FORMAT=COMPRESSED`可以显著减少数据和索引占用的磁盘空间，特别是对于包含大量可压缩数据（如文本或冗余数据）的表。
- **CPU开销**：压缩和解压缩数据需要额外的CPU资源，这可能会影响到系统的CPU性能，特别是在高负载情况下。
- **I/O性能**：虽然压缩减少了磁盘上的数据量，从而可能减少了读取数据所需的I/O操作，但频繁的压缩和解压缩可能会抵消这一优势。

#### 使用场景

- **适用于读多写少的场景**：如果应用主要进行读操作，压缩可以减少读取数据所需的I/O，提高性能。
- **数据冷存储**：对于不经常访问的数据，使用压缩可以节省存储成本。

## hint
用于向数据库查询优化器提供额外的信息，以影响查询执行计划的选择。通过使用hint，开发者可以在不修改数据库本身统计信息和结构的情况下，指导优化器采取特定的行动。这可以帮助优化复杂的查询，特别是在优化器未能选择最优查询计划的情况下。
### 索引选择
* USE INDEX：指定查询应该使用的索引。
* FORCE INDEX：强制查询使用特定的索引，即使MySQL认为不使用索引或使用其他索引更优。
* IGNORE INDEX：告诉优化器忽略一个或多个索引。

### 查询执行方式
* STRAIGHT_JOIN：强制MySQL按照FROM子句中表的顺序来连接表。
* SQL_SMALL_RESULT：指示优化器查询结果集较小，优化器可以选择更适合处理小结果集的查询计划。
* SQL_BIG_RESULT：指示优化器查询结果集较大，优化器可能选择不同的查询计划以处理大量数据。

### 其他
* SQL_NO_CACHE：指示MySQL不要将查询结果缓存，这对于一次性查询很有用。
* SQL_CALC_FOUND_ROWS：告诉MySQL计算符合条件的总行数，即使查询本身有LIMIT限制。

##  MySQL单表数据达到什么程度需要分表分库
MYSQL单表数据达2000万性能严重下降，经验得出大部分单条数据的大小在1k左右，所以一个页可以存储的数据大约为16条。Innodb使用B+树来进行索引存储，主键bigint占8字节，指针占6字节，那么根节点可以存储主键+指针数就是16*1024/14 = 1170.
两层索引可以存储16*1170=18720条数据
三层索引可以存储16*1170*1170 = 21902400 ～约等于2000w条数据。
所以超过2000w条数据之后理论上就需要四层索引来处理，那么为什么超过之后性能会下降严重呢？
* 因为读取一次索引就是一次IO操作，那么为什么四次IO就会比三次IO要性能下降很多呢？
* 因为InnoDB使用innodb_buffer_pool_size来设置缓存，会存储索引和数据，一般innodb_buffer_pool_size的大小默认为128M，大概可以存储2层缓存，所以一般三层索引只需要执行一次IO即可，四层索引需要执行2次IO，性能才会严重下降。
所以在数据量比较大的情况下，可以通过分表来降低单表的存储数据，来加快数据的查询速度

## 排名内置函数

`8.0以上版本使用`

* RANK()  
 并列跳跃排名，并列即相同的值，相同的值保留重复名次，遇到下一个不同值时，跳跃到总共的排名。 1 2 2 4 5  
* DENSE_RANK()
并列连续排序，并列即相同的值，相同的值保留重复名次，遇到下一个不同值时，依然按照连续数字排名。 1 2 2 3 4
* ROW_NUMBER()
连续排名，即使相同的值，依旧按照连续数字进行排名。 1 2 3 4

```sql
// 需要配合 over 使用，partition by分区，order by是排序
select a,b,rank() over(partition by a order by a desc) as 'rank' from table_name
```

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

<https://blog.csdn.net/wuhenyouyuyouyu/article/details/93377605>  
<https://blog.csdn.net/m0_37892044/article/details/121795586>

## MVCC

<https://blog.csdn.net/qq_38538733/article/details/88902979>

## 行数据的最大值

一条记录占用的最大存储空间是有限制的，除了 BLOB 或者 TEXT 类型的列之外(根据具体行格式判断，Compact存放前768字节，Dynamic等不存放)，其他所有的列（不包括隐藏列和记录头信息）占用的字节长度加起来不能超过 65535 个字节(数据页大小16k)。
<https://zhuanlan.zhihu.com/p/53413773>

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
    <img src=".\image\21.jpg" alt="21" />
    删除后
    <img src=".\image\22.jpg" alt="22" />
    可以看出虽然行数和平均的行数长度都为0，但是数据的长度没有改变，只是作为空间碎片的数据重复使用，并没有释放磁盘空间。
    * innodb
    删除前
    <img src=".\image\23.jpg" alt="23" />
    删除后
    <img src=".\image\24.jpg" alt="24" />
    可以看出虽然行数和平均的行数长度都为0，但是数据的长度没有改变(删除后的空间在innodb引擎不作为空间碎片)。
4. 执行后不重置 auto_increment

* truncate table table_name

1. truncate 是 DDL 语句，速度快(不记录undo日志)，执行后无法回滚。
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

<img src=".\image\66.jpg" alt="66" />

## 锁错误排查方式

<img src=".\image\68.jpg" alt="68" />

## 基本概念

* 聚簇索引
* 索引覆盖
* 索引下推
* MRR
<img src=".\image\87.jpg" alt="87" />
<img src=".\image\88.jpg" alt="88" />

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
<img src=".\image\80.jpg" alt="80" />
<img src=".\image\81.jpg" alt="81" />
<img src=".\image\82.jpg" alt="82" />
<img src=".\image\83.jpg" alt="83" />
<img src=".\image\84.jpg" alt="84" />
<img src=".\image\85.jpg" alt="85" />
<img src=".\image\86.jpg" alt="86" />

## 查询语句执行过程

<img src=".\image\89.jpg" alt="89" />

## 主从同步

<img src=".\image\90.jpg" alt="90" />

## 正排索引和倒排索引

* 正排索引以文档为关键字，倒排索引以字或词为索引

## 重复插入数据

### ON DUPLICATE KEY UPDATE(mysql独有语法)

插入数据时，如果表中存在对应主键数据，会执行 ON DUPLICATE KEY UPDATE后续更新操作，若后续字段有发生更新，返回2(用于区分不同情况)，若覆盖更新的字段和原字段数据一致，返回0；如果表中没有该主键，插入数据返回1。

```sql
-- 需要使用主键判断
INSERT INTO table_name (`id`, `create_time`, `update_time`) VALUES (2, 0, 0) on DUPLICATE KEY UPDATE create_time=2 , update_time=1;
```

[ON DUPLICATE KEY UPDATE 用法与说明](https://blog.csdn.net/zyb2017/article/details/78449910)

### REPLACE

插入数据时，如果表中存在对应主键数据，会先删除记录，再插入。
[MySQL replace语句](https://www.yiibai.com/mysql/replace.html)

## GTID(Global Transaction Identifier)

事务全局标识符，分为两部分：服务器UUID唯一标识符+ID，在row或者mixed模式生效，事务提交后会写入binlog递增生成ID，用于全局(多服务器)唯一表示事务。在单源头服务器中，gtid是递增的，多源头服务器中，因为在同步数据时会按源数据的gtid写入从数据库，所以gtid是乱序的。

### 作用

#### 主从故障切换

* 选取新主
传统模式下通过日志的位置和日志的偏移量(那个文件同步到哪里)衡量从数据库数据同步完整性，这个过程复杂且易错，gtid可以简化这个过程。
* 指定从服务器复制位置
传统模式下需要为每个从服务器设置同步新位置，gtid模式下，只需要同步executed_gtid_set没有执行过的事务即可。

#### 主从数据一致性

gtid_executed->执行过的gtid，通过取主从服务器集合差集，判断主从数据一致性情况

#### 多机房同步数据回环问题

通过executed_gtid_set判断事务是否执行过。

## NULL

表示数据不存在，意味着这个字段没有值，和类型的空值不相同。

* NULL !=NULL
* 会被汇总函数忽略
* 排序时，根据 SQL_MODE 变量(NULLS_FIRST||NULLS_LAST)排序

## datetime vs timestamp

* datetime需要5-8字节，timestamp需要4字节
* timestamp范围 1970-2027，datetime没有限制
* timestamp使用utc时区格式存储，使用时根据不同时区转换，datetime不跟随时区变化
