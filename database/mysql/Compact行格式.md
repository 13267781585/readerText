## Compact行格式

#### 行格式分类

mysql行的格式有Compact，Redundant，Dynamic，Compressed四种，mysql5.5以上的默认compact格式

#### 查询行格式

```sql
show variables like 'innodb_%format';
```

<img src=".\image\4.jpg" alt="4" />

#### 变长列表

InnoDB 中，利用 变长字段长度列表 来解决上面的问题：

* 变长字段长度列表记录每一个变长字段值的长度，存储的长度是十六进制的。
* 如果有多个变长字段，那么变长字段长度列表是按逆序存储的。
* CHAR(M)
M代表字符的数量，对于这类数据类型的字段，是否需要记录到变长长度表，取决于表采用的编码格式，若使用的是ascii编码，字符的编码只需要1字节，是定长的；若是gbk(1-2字节)或者utf-8(1-3字节)，则需要记录到变长表中。
<img src=".\image\5.jpg" alt="5" />

#### NULL值列表

1. InnoDB 中，利用 NULL值列表 来解决上面的问题：

* NULL 值列表记录可为 NULL 的字段的情况。
* 用二进制bit位来标识字段值是否为 NULL。1为 NULL，0 不为 NULL。
* 如果有多个可为 NULL 的字段，那么 NULL 值列表也是按照逆序存储的。
* 而且 NULL 值列表的位数必须是 8bit 的N倍。例如：列表仅仅只有4个bit，则往高位补0，补到 8个bit。
<img src=".\image\6.jpg" alt="6" />

2. 采用 NULL值列表 和 直接存储“NULL”字符串相比，有多大的存储差距？

* NULL值列表 一个 bit
* NULL值 四个byte

#### 数据头

40 bit
<img src=".\image\7.jpg" alt="7" />

<img src=".\image\8.jpg" alt="8" />

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

数据页最大为16kb，mysql数据类型varchar，text可以远远超过这个值，mysql规定当字段数据大于 768byte 时，只存放数据前 768byte 的数据还有包含一个指向其他数据页的指针。这样可以保证一个数据页可以存放足够多的行，增大索引树的效率。
<img src=".\image\9.jpg" alt="9" />

#### 其他行格式

* Redunmant
设计粗糙，占用空间大，非紧凑的形式，导致查询的效率低
* Dynamic和Compressed
和Compact相差不大，在处理行溢出数据不一样，直接采用20字节指针记录存放数据的页面，不会存放前768字节的数据，Compressed还可以对页面进行压缩，可以极大的节省内存空间，但是在查询时需要先解压页面，消耗CPU资源节省磁盘资源。

转载自
<https://www.cnblogs.com/Howinfun/p/12370415.html>
<https://blog.csdn.net/XueyinGuo/article/details/119222878>
<https://zhuanlan.zhihu.com/p/312089234>
