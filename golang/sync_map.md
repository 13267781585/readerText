# sync.Map

## 设计思想

* 空间换时间，冗余了一份只读数据(没有增量数据，存量会修改)，提高读效率
* 只读数据double check的思想

## 优劣势

* 适合读多写少的场景，适合作为小型缓存容器使用
* 因为冗余了数据用于无锁读，不合适存放大量数据
* 不适合大数量容器使用，新增数据会刷新read数据到dirty中。
* key不适合大对象，read中删除数据只会将val置为nil，不会清除key。

## 触发读写数据置换条件

* 适合读多写少的场景，适合作为小型缓存容器使用。
* 因为冗余了数据用于无锁读，不合适存放大量数据。


[sync.Map源码解析](https://blog.csdn.net/u012785877/article/details/129832499)
[Golang - sync.map 设计思想和底层源码分析](https://blog.csdn.net/jiohfgj/article/details/130531564)
[踩了 Golang sync.Map 的一个坑](https://juejin.cn/post/6932780282479132685)