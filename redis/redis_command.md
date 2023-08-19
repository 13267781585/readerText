# Redis命令

## 查询key过期时间

```redis
ttl key  
没设过期时间返回 -1   
没有改key -2
```

## 设置字符串

```redis
setex key seconds value  
psetex key milliseconds value   
```

## 查询长度

```redis
字符串 strlen key (key不存在返回0，不能作为判断key是否存在的依据)   
链表 llen key
```

## 原子自增

```redis
incr key(若key不存在，创建初始化为0)  
decr key
```

## 设置查询哈希

```redis
hset key field value   
hmset key field1 value1[]  
(get同上)   
hgetall key 获取所有key和value 先显示key 再显示value一列一列显示
```

## 获取key

```redis
key pattern 获取redis中所有key   
hkeys key 获取对应hash表所有key   
hvals key 获取hash对应值  
scan cursor [match] [count] 返回一定数量的 key，可能重复，有匹配原则时没有找到返回空链表，返回下一次遍历的索引   
sscan cursor [match] [count] 集合的返回所有元素  
ZSCAN key cursor [MATCH pattern] [COUNT count]  
迭代有序集合中的元素（包括元素成员和元素分值） 
```

## 链表基本操作

```redis
brpop/blpop key1 [key2] timeout 限时弹出(阻塞)  
lindex key inde 索引获取  
rpop/lpop key  
rpush/lpush key value1 value2  
lrange key start stop 范围查询(-1表示全部)  
lset key index value 索引修改  
ltrim key start stop 修剪  
```

## 集合基本操作

```redis
sadd key value value1 添加元素  
scard key 集合大小  
smembers key 返回集合数据  
sismember key value 判断value 是否在集合中1是 0否  
spop key 随机移除元素  
sunion key1 [key2] 返回并集  
sunionstore dest key [key1]   
sinter key1 [key2] 返回交集  
sinterstore dest key [key1]   
sdiff key1 [key2] 第一个集合与其他集合差集  
sdiffstore dest key [key1] 
```

## 有序集合

```redis
zadd key score value score1 value1 添加元素  
zcard key 集合大小  
zcount key min max  
计算在有序集合中指定区间分数的成员数  
ZINCRBY key increment member  
有序集合中对指定成员的分数加上增量  
ZINTERSTORE destination numkeys key [key ...]  
计算给定的一个或多个有序集的交集并将结果集存储在新的有序集合   destination 中 numkeys为key的数量   
ZRANGE key start stop [WITHSCORES]  
查询范围内的元素(是否包含分数)  
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]  
通过分数返回有序集合指定区间内的成员  
ZRANGEBYSCORE zset (1 5(返回所有符合条件 1 < score <= 5 的成员)  
ZRANK key member   
返回有序集合中指定成员的索引  
ZREM key member [member ...]  
移除有序集合中的一个或多个成员  
ZSCORE key member  
返回有序集中，成员的分数值  
ZUNIONSTORE destination numkeys key [key ...]  
计算给定的一个或多个有序集的并集，并存储在新的 key 中  
```

## 查询客户端信息

```redis
client list
```

## 主从复制

```redis
slaveof ip port 成为ip:port的从服务器
info replication 查看同步信息
```

## 哨兵模式

```redis
redis-sentinel config_path 启动哨兵节点
redis-server config_path --sentinel
```

## 集群模式

```redis
cluster nodes 集群节点信息
cluster meet <ip> <port> 集群加入ip:port节点
cluster info 节点集群信息
cluster addslots [slots...]给节点增加槽
cluster keyslots <key> 查看key属于哪一个槽
cluster getkeysinslots <slot> <count> 最多返回count属于槽slot的数据
cluster replicate <node_id> 让接收到该命令的节点成为node_id节点的从节点
```

## 连接客户端

```redis
./redis-cli
docker exec -it <redis_name> redis-cli
```

## 查询key编码

```redis
object encoding <key>
```

## 查询key的上次访问时间

```redis
object idletime <key>
```

## 查询配置

```redis
config get <config_name>(可以使用通配符)
```

## 设置过期时间

```redis
expire <key> <ttl> ttl秒过期
pexpire <key> <ttl> ttl毫秒过期
expireat <key> <timestamp> timestamp过期(单位秒)
pexpireat <key> <timestamp> timestamp过期(单位毫米)
persist <key> 移除过期时间
```

## 延迟监控

```redis
CONFIG SET latency-monitor-threshold <milliseconds> 设置监控延迟阈值
latency doctor 延迟分析结果和应对方法建议
```



## 服务端命令监控

```redis
monitor 显示服务端接收到的命令
```


## 查询bigkeys和hotkeys

```redis
redis -cli --bigkeys
redis -cli --hotkeys
```


查询key的序列化信息

```
debug object <key>
```


## 检查修复持久化文件

```
redis-check-aof <-fix> <fileName> 检查/修复aof文件
redis-check-dump <fileName> 检查rdb文件
```

## 查询客户端连接

```redis
client list 查询客户端连接的信(连接的具体信息)
info clients 查询客户端的汇总信息

```


# Redis参数

```
maxclients 最大连接数
appendonly 是否开启aof
auto-aof-rewrite-percentage aof文件大小增长百分比触发重写
```
