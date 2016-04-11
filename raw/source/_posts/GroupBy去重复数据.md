---
title: GroupBy去重复数据
tags: [develop,MySQL]
date: 2016-04-11 18:25:08
---
# 去重的方式
我们经常会遇到查询的数据有重复的情况，常用的两种方式是**distinct** 和 **group by**


# MSSQL的限制
最初在接触SQLServer的时候，对于Group By的理解是比较严格的，因为MSSQL要求group by分组出现的字段必须在select的子句中，而其他**非group by的字段必须出现在聚合函数里**才行，否则报错。

# MySql的灵活
而MYSQL对于group by的约束相对就很宽泛了，没有要求其他字段必须出现在聚合函数中，因此，使用group by进行去重就是非常简单的事情了。

# MySql示例
```sql
select id, name from table group by name
```

P.S. MSSql去重
http://www.cnblogs.com/dannywang/p/3315063.html