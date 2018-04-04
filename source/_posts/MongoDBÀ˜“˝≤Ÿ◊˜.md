---
title: MongoDB索引操作
tags:
  - MongoDB
date: 2017-06-29 15:19:20
---


上面那篇MongoDB Documentation关于查询优化的案例，数据只有10条，看不出来性能有多少提升，这篇再尝试一个例子。

<!-- more -->

## 准备数据

首先先插入1千万条测试数据

``` shell
use test

for(var i = 0; i < 10000000; i++){
    var rand = parseInt(i * Math.random());
    db.person.insert({name: "hxc"+i, age: i})
}
```

![prepareData](http://7xkfga.com1.z0.glb.clouddn.com/prepareData.png)

数据插入需要好久，这个在以后也是个需要优化的地方。

## 使用 explain函数分析查询性能

现在在person这个collection上面没有创建任何索引，这里使用MongoDB提供的 explain工具分析一下一个查询的性能

``` shell
db.person.find({ name: "hxc"+10000}).explain()
```

![initialPerformance](http://7xkfga.com1.z0.glb.clouddn.com/initialExecutionStatus.png)

上面查询使用的是COLLSCAN，就是表扫描，totalDocsExamined为全部1千万，nReturn为1，最终返回一个文档， executionTimeMillisEstimate为4637，预计耗时4637毫秒。

## 创建索引

试一试在name字段上创建一个索引呢？

``` shell
db.person.ensureIndex({ name: 1})
db.person.find({ name: "hxc"+10000}).explain()
```

![executionStatusAfterIndex](http://7xkfga.com1.z0.glb.clouddn.com/indexScan.png)

再来看看查询的性能，这回使用的是IDXSCAN，mongodb在后台使用B树结构来存放索引，这里使用的索引名字是name_1，只浏览了一个文档就返回这一个文档，executionTimeMillisEstimate预计耗时0毫秒。

note: 在创建索引的时候，会占用一个写锁，如果数据量很大的话，会产生很多问题，所以建议用background方式为大表创建索引。

``` shell
db.person.ensureIndex({ name: 1}, {background: 1})
```


## 创建组和索引

有时候查询不是单条件的，可能是多条件，这时候可以创建组合索引来加速查询

``` shell
db.person.ensureIndex({ name: 1, birthday: 1})
db.person.ensureIndex({ birthday: 1, name: 1})

db.person.getIndexes()
```

下面就是创建好的所有索引

![showIndexes](http://7xkfga.com1.z0.glb.clouddn.com/allIndexes.png)

其中第一个索引是在创建collection的时候系统自动创建的一个唯一性索引，key值为 `_id`。 最后两个是刚才所创建的组合索引，这两个组和索引使用的字段虽然是一样的，但是这是两个完全不同的索引

下面分析一下下面的查询使用的到底是哪个索引

``` shell
db.person.find({ birthday: "1989-05-01", name: "mary"}).explain()
```

![chooseIndex](http://7xkfga.com1.z0.glb.clouddn.com/winingPlan.png)

可以看出，最终优化器选择了使用name_1_birthday_1这个索引。查询优化器会使用我们建立的这些索引来创建查询方案，优化器会从中选择最优的执行查询，同时也可以看到被优化器拒绝的查询计划。当然如果非要用自己指定的查询方案，这也是可以的，在mongodb中给我们提供了`hint`方法让我们可以暴力执行。
