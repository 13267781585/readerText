
1. 查询key过期时间   
ttl key    
没设过期时间返回 -1   
没有改key -2

2. 设置字符串和过期时间  
setex key seconds value  
psetex key milliseconds value   
expire key seconds 设置过期时间


3. 查询长度  
字符串 strlen key (key不存在返回0，不能作为判断key是否存在的依据)   
链表 llen key

4. 原子自增  
incr key(若key不存在，创建初始化为0)  
decr key

5. 设置查询哈希  
hset key field value   
hmset key field1 value1[]  
(get同上)   
hgetall key 获取所有key和value 先显示key 再显示value一列一列显示

6. 获取key   
key pattern 获取redis中所有key   
hkeys key 获取对应hash表所有key   
hvals key 获取hash对应值  
scan cursor [match] [count] 返回一定数量的 key，可能重复，有匹配原则时没有找到返回空链表，返回下一次遍历的索引   
sscan cursor [match] [count] 集合的返回所有元素  
ZSCAN key cursor [MATCH pattern] [COUNT count]  
迭代有序集合中的元素（包括元素成员和元素分值）  

7. 链表基本操作  
brpop/blpop key1 [key2] timeout 限时弹出(阻塞)  
lindex key inde 索引获取  
rpop/lpop key  
rpush/lpush key value1 value2  
lrange key start stop 范围查询(-1表示全部)  
lset key index value 索引修改  
ltrim key start stop 修剪  

8. 集合基本操作    
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

9. 有序集合   
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