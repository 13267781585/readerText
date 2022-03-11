## Compact行格式

#### 查询行格式
```sql
show variables like 'innodb_%format';
```

![4](.\image\4.jpg)

#### 变长列表
InnoDB 中，利用 变长字段长度列表 来解决上面的问题：

* 变长字段长度列表记录每一个变长字段值的长度，存储的长度是十六进制的。
* 如果有多个变长字段，那么变长字段长度列表是按逆序存储的。

![5](.\image\5.jpg)


#### NULL值列表
1. InnoDB 中，利用 NULL值列表 来解决上面的问题：
* NULL 值列表记录可为 NULL 的字段的情况。
* 用二进制bit位来标识字段值是否为 NULL。1为 NULL，0 不为 NULL。
* 如果有多个可为 NULL 的字段，那么 NULL 值列表也是按照逆序存储的。
* 而且 NULL 值列表的位数必须是 8bit 的N倍。例如：列表仅仅只有4个bit，则往高位补0，补到 8个bit。
![6](.\image\6.jpg)

2. 采用 NULL值列表 和 直接存储“NULL”字符串相比，有多大的存储差距？
* NULL值列表 一个 bit
* NULL值 四个byte

#### 数据头 
40 bit
![7](.\image\7.jpg)

![8](.\image\8.jpg)

#### 隐藏字段

除了变长字段长度列表、NULL值列表、40个bit位的数据头和真实数据，其实还包含了一些隐藏字段：

* DB_ROW_ID 字段：如果我们没有指定主键和unique key唯一索引的时候，他就内部自动加一个ROW_ID作为主键。
* DB_TRX_ID 字段：事务 ID，标识这是哪个事务更新的数据
* DB_ROLL_PTR 字段：回滚指针，用来进行事务回滚的

加上隐藏字段后，上面的例子的实际存储可能就是：

```java
0x06 0x08 00000101 0000000000000000000010000000000000011001 00000000094C（DB_ROW_ID）00000000032D（DB_TRX_ID） EA000010078E（DB_ROL_PTR） 616161 636320 6262626262(编码数据)
```


#### 行溢出
数据页最大为16kb，mysql数据类型vchar，text可以远远超过这个值，mysql规定当字段数据大于 768byte 时，只存放数据前 768byte 的数据还有包含一个指向其他数据页的指针。这样可以保证一个数据页可以存放足够多的行，增大索引树的效率。
![9](.\image\9.jpg)



转载自
https://www.cnblogs.com/Howinfun/p/12370415.html    
https://blog.csdn.net/XueyinGuo/article/details/119222878   
https://zhuanlan.zhihu.com/p/312089234