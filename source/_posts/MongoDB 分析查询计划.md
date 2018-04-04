---
title: MongoDB 分析查询计划
tags:
  - MongoDB
date: 2017-06-29 10:18:51
---


## 查询计划

和传统的关系型数据库的执行计划类似，MongoDB也提供了查询计划。MongoDB的查询优化器处理查询语句，并且从生成的执行计划中选择最优的来执行查询过程。下图显示了MongoDB查询计划的逻辑步骤：

![QueryPlannerLogic](https://docs.mongodb.com/manual/_images/query-planner-diagram.bakedsvg.svg)

## 分析查询的性能

MongoDB 提供了一个 explain 命令让我们获知系统如何处理查询请求。利用 explain 命令，我们可以很好地观察系统如何使用索引来加快检索，同时可以针对性优化索引。

<!-- more -->

## 分析实例

### 准备数据

``` shell
db.inventory.insertMany([
{ "_id" : 1, "item" : "f1", type: "food", quantity: 500 },
{ "_id" : 2, "item" : "f2", type: "food", quantity: 100 },
{ "_id" : 3, "item" : "p1", type: "paper", quantity: 200 },
{ "_id" : 4, "item" : "p2", type: "paper", quantity: 150 },
{ "_id" : 5, "item" : "f3", type: "food", quantity: 300 },
{ "_id" : 6, "item" : "t1", type: "toys", quantity: 500 },
{ "_id" : 7, "item" : "a1", type: "apparel", quantity: 250 },
{ "_id" : 8, "item" : "a2", type: "apparel", quantity: 400 },
{ "_id" : 9, "item" : "t2", type: "toys", quantity: 50 },
{ "_id" : 10, "item" : "f4", type: "food", quantity: 75 }
])
```

### 无索引的查询

在这个collection没有索引的情况下，写一个查询，查询quantity字段的值在100和200之间的文档

``` shell
db.inventory.find({ quantity: { $gte: 100, $lte: 200}})
```

这时候，我们想知道这个查询语句所选择的查询计划是怎么样的：

``` shell
db.inventory.find(
    { quantity: {$gte: 100, $lte: 200}}
).explain("executionStats")
```

下图为返回的结果：

![executionStats](http://7xkfga.com1.z0.glb.clouddn.com/MongoDBexecutionStats.png)

- `queryPlanner.winningPlan.stage` 显示的是 COLLSCAN集合扫描，也就是关系型数据库的全表扫描，看到这个说明性能肯定不好
- `nReturned`为3，符合的条件的返回为3条
- `totalKeysExamined`为0，没有使用index。
- `totalDocsExamined`为10，扫描了所有记录。

优化的方向也很明显， 就是如何减少检查的文档数量。

### 创建索引

在quantity字段上创建索引：

``` shell
db.inventory.createIndex( {quantity: 1})
```

再次执行上面的命令查看查询计划

![executionStatsAfterIndex](http://7xkfga.com1.z0.glb.clouddn.com/MongoDBexecutionStatsAfterIndex.png)

- `queryPlanner.winningPlan.inputStage.stage` 显示IXSCAN说明使用了索引
- `executionStats.nReturned` 有3条文档符合条件返回
- `executionStats.totalKeysExamined` 扫描了3个索引
- `executionStats.totalDocsExamined` 一共扫描了3个文档




参考资料：

> [MongoDB Documentation: Analyze Query Performance ](https://docs.mongodb.com/manual/tutorial/analyze-query-plan/)
