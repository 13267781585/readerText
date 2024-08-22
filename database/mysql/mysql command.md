# mysql command

## 登录客户端

```sql
mysql -u (用户名) -h (mysql服务所在地址) -P (可选默认3306 指定端口号) -p 数据库
```

## docker登录mysql

docker ps 查看docker运行的程序
启动mysql
docker exec -it {docker_mysql_name} bash
mysql -u root -p

## 运行进程

```sql
show processlist;
```
 
## 修改表
```sql
-- 添加字段
alter table table_name add col_name col_type;

-- 修改字段类型
alter table table_name modify col_name col_type;

-- 修改字段名称类型
alter table table_name change old_col new_col col_type;

-- 删除字段
alter table table_name drop col_name;

-- 修改表名
alter table old_table_name rename new_table_name;

-- 增加删除索引
alter table table_name add/drop index/primary index/unique index  index_name(col_name,...);
```

## 表的信息

```sql
show create table table_name; -- 查看建表语句

--表的基本信息
desc table_name;

-- 数据库中表的统计信息
show table status;

-- 索引消息
show index from table_name;
```

## mysql变量配置

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

-- 警告条数
warning_count

-- 警告开关
sql_notes

-- mysql版本
version

-- 临时内存表的大小限制
tmp_table_size
max_heap_table_size
```
