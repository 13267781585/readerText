## mysql command

##### 查询mysql版本
```sql
select version();
```

##### 运行进程
```sql
show processlist;
```

##### 表的数据统计
```sql
show table status
```

##### mysql变量配置
```sql
-- 使用index dive的限制 -- 查询区间个数过多 精确统计或者估算  
eq_range_index_dive_limit

-- innodb 锁等待时间
innodb_lock_wait_timeout

-- meta data lock 锁和表锁等待时间
lock_wait_timeout

-- 自动提交
autocommit

-- 事务隔离级别 8.0
transaction_isolation

-- 设置统计表实时更新
-- 全局
SET GLOBAL information_schema_stats_expiry=0;
SET @@GLOBAL.information_schema_stats_expiry=0;

-- 会话
SET SESSION information_schema_stats_expiry=0;
SET @@SESSION.information_schema_stats_expiry=0;
```