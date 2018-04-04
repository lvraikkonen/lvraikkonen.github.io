---
title: MongoDB 基础知识
tags:
  - MongoDB
date: 2017-06-27 13:20:00
---


MongoDB 是一个基于 分布式文件存储 的数据库。由 C++ 语言编写。

## MongoDB 相关概念

mongodb中基本的概念是文档、集合、数据库，和传统关系型数据库相关概念的对应关系如下表：

|  RDBMS       |  MongoDB    |  描述      |
|------------- | ----------- |------------|
| database     |	database   |	数据库         |
| table        |	collection |	数据库表/集合  |
| row          |	document   |	数据记录行/文档|
| column       |	field	     | 数据字段/域    |
| index        |	index	     | 索引          |
| table	joins	|              |MongoDB可以使用DbRef或者$lookup实现|
| primary key |	primary key |	主键,MongoDB自动将_id字段设置为主键|

<!-- more -->

### 数据存储————文档

文档是一个键值(key-value)对(即BSON)类似JSON对象。MongoDB 的文档不需要设置相同的字段，并且相同的字段不需要相同的数据类型，这与关系型数据库有很大的区别，也是 MongoDB 非常突出的特点。

### MongoDB的主键 `_id`

MongoDB默认会为每个document生成一个 `_id` 属性，作为默认主键，且默认值为ObjectId,可以更改 `_id` 的值(可为空字符串)，但每个document必须拥有 `_id` 属性。这个字段是为了保证文档的唯一性。

### MongoDB系统保留数据库

- admin
- local
- config


### 时间数据类型

MongoDB中的时间类型默认是MongoDate，MongoDate默认是按照UTC（世界标准时间）来存储。例如下面的两种使用方式:

``` shell
db.col.insert({"date": new Date(), num: 1})
db.col.insert({"date": new Date().toLocaleString(), num: 2})

db.col.find()
{
    "_id" : ObjectId("539944b14a696442d95eaf08"),
    "date" : ISODate("2014-06-12T06:12:01.500Z"),
    "num" : 1
}
{
    "_id" : ObjectId("539944b14a696442d95eaf09"),
    "date" : "Thu Jun 12 14:12:01 2014",
    "num" : 2
}
```

note: 第一条数据存储的是一个Date类型，第二条存储存储的是String类型。两条数据的时间相差大约8个小时（忽略操作时间），第一条数据MongoDB是按照UTC时间来进行存储。

### MongoDB中的一对多、多对多关系

MongoDB的基本单元是Document（文档），通过文档的嵌套（关联）来组织、描述数据之间的关系。
例如我们要表示一对多关系，在关系型数据库中我们通常会设计两张表A（一）、B（多），然后在B表中存入A的主键，以此做关联关系。然后查询的时候需要从两张表分别取数据。MongoDB中的Document是通过嵌套来描述数据之间的关系，例如：

``` JSON
{
    _id:ObjectId("akdjfiou23o4iu23oi5jktlksdjfa")
    teacherName: "foo",
    students: [
        {
            stuName: "foo",
            totalScore：100，
            otherInfo :[]
            ...
        },{
            stuName: "bar",
            totalScore：90，
            otherInfo :[]
            ...
        }
    ]
}
```
一次查询便可得到所有老师和同学的对应关系。


### 内嵌文档查询

在MongoDB中文档的查询是与顺序有关的。例如：

``` json
{
    "address" : {
        "province" : "河北省",
        "city" : "石家庄"
    },
    "number" : 2640613
}
```

要搜索province为“河北省”、city为“石家庄”可以这样:

``` shell
db.col.find(
    {
        "address":{
            "city" : "石家庄",
            "province" : "河北省"
        }
    }
)
```

然而这样什么都不会查询到。事实上，这样的查询MongoDB会当做全量匹配查询，即document中所有属性与查询条件全部一致时才会被返回。当然这里的“全部一致”也包括属性的顺序。那么，上面的查询如果想搜索到之前的应该先补充number属性，然后更改address属性下的顺序。

在实际应用中我们当然不会这么来查询文档，尤其是需要查询内嵌文档的时候。MongoDB中提供"."（点）表示法来查询内嵌文档。因此，上面的查询可以这样写：

``` shell
db.col.find(
    {
        "address.privince":"河北省"
    }
)
```
