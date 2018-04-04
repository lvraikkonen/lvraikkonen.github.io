---
title: T-SQL查询语句执行顺序
date: 2017-06-06 10:43:35
tags:
  - SQL

---


数据库查询语句可以说是基础中的基础，查询语句是后面查询性能优化的基础，但是很多人并不能很准确的说出数据库查询的逻辑流程。

## SQL 语句中元素

逻辑查询处理，指的是标准SQL定义的如何处理查询和返回最终结果的概念性路径。和其他的语言代码执行顺序不同，SQL 查询不是按照代码顺序来进行逻辑查询。下面这段SQL 查询

``` sql
SELECT empid
     , YEAR(orderdate) AS OrderYear
     , COUNT(*) AS NumOrders
FROM Sales.Orders
WHERE custid = 71
GROUP BY empid, YEAR(orderdate)
HAVING COUNT(*) > 1
ORDER BY empid, OrderYear
```

这段SQL 语句其实是按照下面的顺序进行的逻辑处理：

1. **FROM**
2. **WHERE**
3. **GROUP BY**
4. **HAVING**
5. **SELECT**
6. **ORDER BY**

<!-- more -->

`FROM` 子句指定要查询的表名称和进行多表运算的表运算符(JOIN)； `WHERE` 子句可以指定一个谓词或者逻辑表达式来筛选由 `FROM`阶段返回的行； `GROUP BY`阶段允许用户把前面阶段返回的行排列到组中； `HAVING` 子句可以指定一个谓词来筛选前面GROUP出的组，而不是筛选单个行； `SELECT` 子句用户指定要返回到查询结果表中属性(列)； `ORDER BY` 子句允许对输出行进行排序。

## 流程图

这里引用 [Itzik Ben-Gan](http://tsql.solidq.com/) 的流程图

![SQLLogicFlowchart](http://7xkfga.com1.z0.glb.clouddn.com/208801.jfif)

## 分步分析


### FROM 子句

FROM阶段标识出查询的来源表，并处理表运算符。在涉及到联接运算的查询中（各种join），主要有以下几个步骤：

1. 求笛卡尔积。不论是什么类型的联接运算，首先都是执行交叉连接（cross join），求笛卡儿积，生成虚拟表VT1-J1。
2. ON筛选器。这个阶段对上个步骤生成的VT1-J1进行筛选，根据ON子句中出现的谓词进行筛选，让谓词取值为true的行通过了考验，插入到VT1-J2。
3. 添加外部行。如果指定了outer join，还需要将VT1-J2中没有找到匹配的行，作为外部行添加到VT1-J2中，生成VT1-J3。

经过以上步骤，FROM阶段就完成了。概括地讲，FROM阶段就是进行预处理的，根据提供的运算符对语句中提到的各个表进行处理（除了join，还有apply，pivot，unpivot）

### WHERE 子句

WHERE阶段是根据<where_predicate>中条件对VT1中的行进行筛选，让条件成立的行才会插入到VT2中。

### GROUP BY阶段

GROUP阶段按照指定的列名列表，将VT2中的行进行分组，生成VT3。最后每个分组只有一行。

### HAVING阶段

该阶段根据HAVING子句中出现的谓词对VT3的分组进行筛选，并将符合条件的组插入到VT4中。

### SELECT阶段

这个阶段是投影的过程，处理SELECT子句提到的元素，产生VT5。这个步骤一般按下列顺序进行

1. 计算SELECT列表中的表达式，生成VT5-1。
2. 若有DISTINCT，则删除VT5-1中的重复行，生成VT5-2
3. 若有TOP，则根据ORDER BY子句定义的逻辑顺序，从VT5-2中选择签名指定数量或者百分比的行，生成VT5-3

### ORDER BY阶段

根据ORDER BY子句中指定的列明列表，对VT5-3中的行，进行排序，生成游标VC6.

当然SQL SERVER在实际的查询过程中，有查询优化器来生成实际的工作计划。以何种顺序来访问表，使用什么方法和索引，应用哪种联接方法，都是由查询优化器来决定的。优化器一般会生成多个工作计划，从中选择开销最小的那个去执行。逻辑查询处理都有非常特定的顺序，但是优化器常常会走捷径。