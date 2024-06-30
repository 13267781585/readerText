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

数据在A、B机房同步链路形成回环(数据发生多次变更，A机房数据V0->V1，产生binlog发送给B机房，B机房执行V0->V1，产生binlog发送给A机房，在A机房执行B回传binlog之前又把V1->v0，产生新的binlog，就会一直循环下去)

#### 解决方案
* GTID
事务在同步过程中gtid不变，binlog从A->B->A过程中gtid是同一个，通过判断gtid是否被执行过来判断。
* 字段判断数据源
通过记录集群/服务器id等系统/业务字段来区分

### 冲突解决策略

* 写入额外表，业务自行处理
* 以记录时间戳为准
* 覆盖写
* 业务自定义处理
