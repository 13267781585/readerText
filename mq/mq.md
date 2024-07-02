# MQ

## 对比
<img src=".\image\4.png" alt="4" />
### kafka
追求高吞吐量，不支持事务，对消息可靠性要求不高的场景，常用于大数据下日志收集等
消息会被持久化到日志中，消息的偏移量是唯一标识，消费者会搜索日志消费消息。
<img src=".\image\3.png" alt="3" />

### RabbitMQ
* 时延极低，性能高，吞吐较低
消息会被复制到队列中，消费完即删除
<img src=".\image\2.png" alt="2" />


### 消费模式
* Rabbitmq把消费复制给每个消费者对列，数据复制开销大
* Kafka会把消息保存到log，消费者查找log消费消息

### 顺序消费
* Rabbitmq单队列单消费者
* Kafka通过分区并发处理多个订单，吞吐大

### 消息匹配
* Rabbitmq有丰富路由匹配机制
* Kafka不提供匹配机制，只能由消费者自己实现

### 消息超时
* rabbitmq提供插件实现
* kafka没有提供


### 消息错误处理
* Rabbitmq提供弹性错误处理，重新入列或者死信队列
* Kafka单分区错误会停止消费

### 消息吞吐
* rabbitmq 万
* kafka 十万


### 维护和使用
* Rabbitmq使用简单，学习成本小，由erlang实现，维护成本大，不易扩展
* Kafka维护成本大，门槛高，


## 幂等处理
* 消息增加全局唯一标识符，消费去重
* 业务数据插入数据库表增加唯一主键
* 数据更新增加版本号，低版本不能覆盖高版本
* 业务前置校验，收到消息后反查数据，判断更新是否需要提交

## 延时消息实现方式
延时队列
1. 消息队列
* rocketmq延时消息
    * 缺点
        * 设置了18个等级延迟间隔，没办法自由设置
* rabbitmq自带死信队列
    * 缺点
        * 消息按照先进先出消费，即使后进的消息已经过期
* rabbitmq_delayed_message_exchange插件
Mnesia是集成在Erlang标准库中的关系型数据库，适合存储较小的数据集。
插件扩展了新的延迟exchange，并且建立了两张Mnesia数据表，一张索引表(order_set类型)，对消费时间做排序，一张数据表(bag类型)，存放消息数据，消息投递后，会根据设置的延迟时间计算消费的时间，并存入两张表，并且根据第一条需要处理的消息设置定时器，定时器到期后取出第一条消息投递到正常的exchang中进行消费。
    * 优点
        * 消息到期就执行
    * 缺点
        * 不支持生产者和exchang的publish confrim机制，消息不可靠
        * 单点问题，消息只投递到单节点，节点宕机，生产者切换节点耗时1s，节点重启耗时5s
        * 不支持横向扩展能力，Mnesia数据表不知道存放大数据，数据只存放在内存，消息量受到内存大小制约，不建议百万级别数据使用
        * Erlang支持设置最大49天的定时器，也就是消息过期时间不能大于49天
    * 性能测试-3节点4C8G-1000qps-总共33w消息-过期时间1s到1hour
        * 正常情况-无丢失消息-消息延迟消费误差ms
        * 叠加两次实例重启-丢失消息(500)-消息延迟消费误差30s

2. redis
* zet
    * 缺点
        * Redis持久化可能丢失数据
3. 数据结构
4. 定时任务
* Quartz
* Handfire
5. 时间轮片法
* Netty
* Kafka