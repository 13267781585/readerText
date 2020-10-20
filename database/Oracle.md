## Oracle 笔记

* DISTINCT   
i.只能用于 SELECT 语句   
ii.重复数据被过滤的条件是 查询的n列数据同时相同   

* ORDER BY   
i.ASC 升序  DESC 降序   
ii.多字段排序按字段顺序排序，后序的字段排序在不改变前一个字段排序顺序中进行(后序排序在前一个字段相同数据中进行)

* FETCH 
```sql
[ OFFSET offset ROWS]
 FETCH  NEXT [  row_count | percent PERCENT  ] ROWS  [ ONLY | WITH TIES ]
```
i.FETCH子句指定要返回的行数或百分比。

ii.
OFFSET子句指定在行限制开始之前要跳过行数。OFFSET子句是可选的。 如果跳过它，则偏移量为0，行限制从第一行开始计算。

偏移量必须是一个数字或一个表达式，其值为一个数字。偏移量遵守以下规则：

    如果偏移量是负值，则将其视为0。
    如果偏移量为NULL或大于查询返回的行数，则不返回任何行。
    如果偏移量包含一个分数，则分数部分被截断。

iii.ONLY | WITH TIES选项

仅返回FETCH NEXT(或FIRST)后的行数或行数的百分比。

WITH TIES返回与最后一行相同的排序键。请注意，如果使用WITH TIES，则必须在查询中指定一个ORDER BY子句。如果不这样做，查询将不会返回额外的行。

