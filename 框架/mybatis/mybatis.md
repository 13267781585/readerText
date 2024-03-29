## Mybatis 

### mybatis如何防止sql注入
mybatis关于sql动态参数设置有两种方式，`#{}`，`${}`，其中#{}采用预编译的方式，参数的设置采用占位符替换的方式，一般用于参数的设置，可以有效sql注入，${}采用sql拼接的方式，一般用于动态表名、列名的设置，不能防止sql注入。


### mybatis一二级缓存  
* 一级缓存(默认开启) 
1. 一级缓存是基于sqlSession连接的(数据库连接？)，会话在被关闭或者执行了更新操作时会清除缓存，一级缓存采用hashmap的结构存放查询过的数据，再次提交相同的查询时会从缓存中返回数据，不会再执行一次sql语句。
2. 判别两次sql相同的条件 
* statementid
* sql语句
* 传入的参数

*二级缓存(默认不开启)
1. 二级缓存是全局的缓存，被所有sqlSession共享，在执行更新操作时缓存会被刷新，默认采用最近最少使用(LRU)来回收。

* 优先级顺序 
二级缓存>一级缓存>数据库查询

* 为什么一级缓存默认开启，二级缓存默认关闭？
因为一级缓存的范围是会话级别的，占用的空间小，可以在一定的查询中提升效率。而二级缓存是全局的，所有会话共享，占用的内存可能比较大，且如果个别会话频繁更新，缓存会被清除，导致效率更低，所以二级缓存默认关闭，开发者在根据自己的需求去开启。

### mapper怎么和xml的sql语句联系起来的   
1. xml的解析
mybatis在初始化的时候会去指定路径下解析xml文件，把sql标签的信息解析到 SqlSource 对象中，会用id(全限定类名+方法名)标识组成一个MappedStatement对象，然后将这些解析出来的信息存放到Configuration对象中。

2. Spring在实例化Mapper对象的时候，使用jdk代理的方式生成一个Mapper对应的代理类，实现接口的方法，方法会根据 全限定类名+方法名 去查找对应的MappedStatement对象然后执行sql语句。

```java
public class DefaultSqlSession implements SqlSession {
 
    public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
        try {
            MappedStatement ms = configuration.getMappedStatement(statement);
            return executor.query(ms,
                wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
        }
    }
}
```
转载：https://www.cnblogs.com/626zch/p/10776985.html