---
title: Group By 按日期统计脚本
tags: [develop,MySQL]
date: 2016-05-06 18:54:47
---

# Group By 按日期统计脚本
```sql
SELECT DATE_FORMAT( created, "%Y-%m-%d" ) , COUNT( * ) 
FROM ios_activation_logs
where created>'2016-04-26' 
GROUP BY DATE_FORMAT( created, "%Y-%m-%d" ) 
```