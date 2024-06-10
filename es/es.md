# ES

## 查询上下文和过滤上下文
### 查询上下文
检查文档是否符合条件+计算相关性得分，通常用于全文搜索，主要关心的是文档的匹配程度。query是查询上下文。

### 过滤上下文
检查文档是否符合条件，不会计算相关性得分，通常用于范围、等值查询等。filter是过滤上下文，频繁调用的filter结果会被缓存用于提升速度。

## 不同版本重要特性
### 6.0
1. 稀疏Doc Values
es使用列式存储，在数据排序、聚合性能好，稀疏Doc values对文档没有某个字段情况进行优化，减少存储这种情况带来的开销。
```es
PUT /my_index
{
  "mappings": {
    "properties": {
      "my_field": {
        "type": "keyword",
        "doc_values": false
      }
    }
  }
}
```


2. index sorting
在索引数据时进行排序，适合经常需要做排序、聚合操作的字段，写入有额外开销。
```es
PUT /my_index
{
  "settings": {
    "index": {
      "sort.field": ["price", "date"],  // 指定排序字段
      "sort.order": ["asc", "desc"]     // 指定每个字段的排序顺序
    }
  },
  "mappings": {
    "properties": {
      "price": {
        "type": "double"
      },
      "date": {
        "type": "date"
      }
    }
  }
}
```

3. 不支持一个index中多个type
type被类比为数据库模型中的表，但是同index中不同type底层不是独立存储，es会把不同type的字段合并存储，就会造成以下影响：
* 字段爆炸，每个文档随着type增多而增多，造成字段稀疏问题
* 小type和大type之间查询和写入相互影响
* 不同type相同名称类型必须相同

4. 基于负载均衡
从轮训匀请求路由，改为基于机器负载路由，会考虑机器cpu、内存、磁盘等情况。

### 7.0
1. 去除type
2. 配置默认值修改
* 默认分片数 5->1

3. 引入Zen 2集群协调机制
减少处理网络分区和节点故障带来"脑裂"问题。
* 选举主节点和主节点发布更改(增删节点)都需要其他节点半数以上确认
* 优化故障检测机制和支持管理员灵活的故障配置

4. 安全特性免费使用
5. Lucene 9.0
6. 内存断路器
通过监控内存使用情况，并在达到某个阈值时阻断进一步的操作，从而保护节点稳定性。

7. Weak-AND算法思想
Weak-AND 算法是在传统的 AND 和 OR 查询操作之间提供了一种折中的方法。在标准的 AND 查询中，只有包含所有查询词的文档才会被检索出来。而在 OR 查询中，包含任何一个查询词的文档都会被检索。算法通过设置一个阈值来工作，这个阈值决定了一个文档必须包含多少查询词才能被认为是相关的。
```es
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "field1": "keyword1" }},
        { "match": { "field2": "keyword2" }},
        { "match": { "field3": "keyword3" }},
        { "match": { "field4": "keyword4" }}
      ],
      "minimum_should_match": 3  // 至少需要匹配的条件数
    }
  }
}
```

8. 间隔查询
适用于需要精确控制词项之间距离和顺序的全文搜索场景，如法律文档、学术文章或任何结构化文本的复杂文本分析。
```es
// 找到包含“Elasticsearch”后面紧跟“search”（中间最多有三个其他词）的文档,any_of 来指定任何一个条件满足即可，或使用 not_containing 来排除某些词的出现
GET /books/_search
{
  "query": {
    "intervals" : {
      "content" : {
        "all_of" : {
          "intervals" : [
            {
              "match" : {
                "query" : "Elasticsearch",
                "max_gaps" : 0, -- 指定词项之间允许的最大间隔数
                "ordered" : true  -- 指定词项是否需要按照给定的顺序出现
              }
            },
            {
              "match" : {
                "query" : "search",
                "max_gaps" : 3,
                "ordered" : true
              }
            }
          ]
        }
      }
    }
  }
}
```


## 容量评估
### 分片数
容量建议10-20G

### 总存储
磁盘大小 = （数据量 * 副本数）/ 预计的利用率
30G数据，打算存储4主1备，磁盘利用率按50%算，那么需要申请的总存储空间为 30G * 8 / 50% = 480G

### 内存大小
总内存大小  = （数据量 * 副本数） / N(离线场景 N=20； 高性能在线场景 N=2) 
* 性能要求不高：480G/20=24G
* 要求高：480G/2=240G

### 实例个数
* 写速度
单机最高50MB/s写入，实例数=每秒写入pqs * 一条数据大小 * 分片数 / 单机最高写入

* 内存
单节点内存上限128G，实例数=总内存/单节点内存

* 存储
单节点存储最高30T

### 实例规格
* 内存=总内存/节点数(cpu:mem=1:2/1:4)
* 存储规格=总存储/节点数

## 集群
* 主从架构

### 节点
集群由多个Node节点组成，一个Node节点有多个Shard分片，一个Shard分片就是一个Lucene 实例
#### 节点类型
* Master Node 主节点
控制整个集群的元数据。只有Master Node节点可以修改节点状态信息及元数据(metadata)的处理，比如索引的新增、删除、分片路由分配、所有索引和相关 Mapping 、Setting 配置等等。

* Master Eligible Node 合格节点
Master节点的备份，该节点只是与集群保持心跳，判断Master是否存活，如果Master故障则参加新一轮的Master选举。

* Data Node 数据节点
存储数据的节点

* Coordinating Node 协调节点
协调客户端请求，路由到对应数据节点上，查询数据返回，并做数据的排序、过滤、合并，最后返回给客户端。

### Shard和Replic
* 分片，数据水平扩展能力
* 副本，容灾
* 一个主分片上只会有一个相同的doc
* Shard和Replic不会在同Node上
* Shard被确定后无法改变，Replic可以

## 路由
路由决定了一个文档会被存储到哪个 Shard 上：
shard = hash(routing) % number_of_primary_shards
routing 默认为文档的 _id

## 运营
### disk使用率
* 大于85%，无法进行索引创建/shard移动
* 大于95%，只度不写

## 重建索引
### reindex
异步查询数据复制到新索引，无法跟踪过程中修改的数据，需要听写。

### 理想方案
提供像binlog一样机制，复制数据发送mq，并监听变更，写入新索引，当新老索引数据差达到设置阈值，通过别名切换。如果有部分更新，切换前可以做短暂停写，mq数据消费完成后再切。

## Q&A
### text和keyword的区别
* text全文索引，在写入时会被分析、转化，存储时不是原来文本，使用用于模糊匹配
* keyword不会被拆解，适合用于过滤、排序、聚合操作

### nested和object的区别
1. objec  
es中任意字段可以看作object类型，object被用于处理object对象/数组，会把对象扁平化，例如 {"user":{"first":"huang","name":"1"}} 打平为 user.first:"huang" 和 user.name:"1"，如果是数字，打平为数组，user.first:["huang","chen"] 和 user.name:["1","2"]。
* 对象之间的字段会被打平成为索引的字段，查询和正常字段一样
* 问题是会散失对象字段之间相关性

2. nested
lucene引擎没有nested的类型，只能处理扁平化文档。nested类型是es的封装实现，嵌套的结构当作独立的文档存储，保留结构字段的相关性。
* 查询需要嵌套在nested中
```es
{
  "query": {
    "nested": {
      "path": "user",
      "query": {
        "match": {"user.first" : "John"}
      }
    }
  }
}
```
* 可用于存放动态kv
* 写入性能低
主子文档被单独存储，在修改的时候需要把主子文档都查询出来做覆盖更新
* 查询性能低
  * 需要把子文档和夫文档关联起来
  * 当查询包含nested时，es需要针对每一个嵌套结构做单独查询