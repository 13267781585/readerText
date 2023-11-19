# Binlog

## 业界binlog同步组件

* linkedin的databus
* 阿里巴巴的canal
* 美团点评的puma
* 字节drc

## 基于binlog复制多机房容灾

### 业界解决方案

* 字节->drc(Data Replication Center)
* 阿里->otter <https://github.com/alibaba/otter>

### 数据回环

数据在A、B机房同步链路形成回环(数据发生多次变更，A机房数据V0->V1，然后快速V1->V0，B机房执行V0->V1，又执行V1->V0，同步binlog给A机房)

#### 解决方案

* 辅助表(两次带宽消耗)
别的机房同步数据，利用事务机制，写入数据+辅助表流水，在同步数据前，检查辅助表是否同步过该记录。
* GTID(一次带宽消耗)
通过判断gtid是否被执行过来判断。
* 字段判断数据源
依赖业务或proxy/client
数据冲突

### 冲突解决策略

* 写入额外表，业务自行处理
* 以记录时间戳为准
* 覆盖写
* 业务自定义处理
