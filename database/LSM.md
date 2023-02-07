### Log-Structured Merge-Tree 日志结构合并树
`牺牲部分磁盘读性能获得高吞吐写` `HBase` `RocksDB` `NoSql`

#### 高吞吐写的原因
* Batch Write：批量随机写操作转化为顺序写
* Multi-Page Block：因为数据在磁盘上是连续存放的，合并过程中，可以顺序读出多个页面处理完后在连续写入，实现一次I/O完成多个页面的读写。   
[LSM-Tree 与 LevelDB 的原理和实现](https://wingsxdu.com/posts/database/leveldb/#memtable)   
[LSM树详解](https://zhuanlan.zhihu.com/p/181498475)

#### 实现场景
* LevelDB
* [HBase](https://help.aliyun.com/document_detail/49503.html)
* [RocksDB](https://www.jianshu.com/p/3302be5542c7)
* [Cassandra](https://developer.aliyun.com/article/713847)

