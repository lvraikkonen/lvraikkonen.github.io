---
title: SQL-Server IO 统计
tags:
    - SQL Server
---


``` sql
SET STATISTICS IO ON

SELECT *
FROM Sales.SalesOrderDetail
```

在执行结果的消息里面显示如下信息：

```
Table 'SalesOrderDetail'.
Scan count 1,
logical reads 1246,
physical reads 0,
read-ahead reads 0,
lob logical reads 0,
lob physical reads 0,
lob read-ahead reads 0.
```

- 预读：在查询计划生成的过程中，用估计的信息去硬盘读取数据到缓存中，预读1242页，也就是从硬盘中读取了1242页放到了缓存中。
- 物理读：查询计划生成好以后，如果缓存缺少所需要的数据，再从硬盘里读取缺少的数据到缓存里。
- 逻辑读：从缓存中读取数据。逻辑读1240次，也就是从缓存中读取1240页数据。
