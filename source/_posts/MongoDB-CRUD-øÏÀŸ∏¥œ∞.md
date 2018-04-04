---
title: MongoDB CRUD 快速复习
date: 2017-06-28 11:08:32
tags:
    - MongoDB
---

提到数据库的基本操作，无论关系型还是非关系型，首先想到的肯定是数据的增删改查 (Create, Read, Update, Delete)，下面记录一下MongoDB里面的CRUD操作。

<!-- more -->

## INSERT 操作

![insertStatement](https://docs.mongodb.com/manual/_images/crud-annotated-mongodb-insertOne.bakedsvg.svg)

MongoDB里面数据叫做文档Document，其实就是JSON对象，常见的插入操作分为单条插入 `insertOne()` 和多条插入 `insertMany()` ，单条插入传入一个JSON对象，多条插入传入一个多个JSON对象的数组

## FIND 操作

![findStatement](https://docs.mongodb.com/manual/_images/crud-annotated-mongodb-find.bakedsvg.svg)

Query操作是这4中操作最重要的部分， 查询的语法如下：

- query criteria，可选。表示集合的查询条件。可指定条件查询参数，或一个空对象({})
- projection，可选。表示查询时所要返回的字段，省略此参数将返回全部字段。其格式如下：
{ field1: <boolean>, field2: <boolean> ... }

返回查询文档的游标，即：执行find()方法时其返回的文档，实际是对文档引用的一个游标。当指定projection参数时，返回值仅包含指定的字段和_id字段，也可以指定不返回_id字段

### 查询参数

下面这些在SQL的WHERE子句中的操作符，在MongoDB中都有实现。

SQL :

```
>, >=, <, <=, !=
And，OR，In，NotIn
```
MongoDB:

```
"$gt", "$gte", "$lt", "$lte", "$ne"
"$and", "$or", "$in"，"$nin"
```

举几个查询的例子：

``` shell
db.inventory.find( { status: { $in: [ "A", "D" ] } } )
# SELECT * FROM inventory WHERE status in ("A", "D")

db.inventory.find( $and: [ { status: "A", qty: { $lt: 30 } } ] )
# SELECT * FROM inventory WHERE status = "A" AND qty < 30

db.inventory.find( { $or: [ { status: "A" }, { qty: { $lt: 30 } } ] } )
# SELECT * FROM inventory WHERE status = "A" OR qty < 30

db.inventory.find( {
     status: "A",
     $or: [ { qty: { $lt: 30 } }, { item: /^p/ } ]
} )
# SELECT * FROM inventory WHERE status = "A" AND ( qty < 30 OR item LIKE "p%")
```

如果查询使用的是嵌套的文档的属性，那就使用 `"field.nestedField"`

## UPDATE 操作

UPDATE 的语法如下：

```
db.collection.update(
   <query>,
   <update>,
   {
     upsert: <boolean>,
     multi: <boolean>,
     writeConcern: <document>,
     collation: <document>
   }
)
```

更新操作分为整体更新和局部更新，整体更新是用一个新的文档完全替代匹配的文档。

**危险**： 使用替换更新时应当注意，如果查询条件匹配到多个文档，所有的文档都会被替换

下面主要说的是局部更新。

![updateStatement](https://docs.mongodb.com/manual/_images/crud-annotated-mongodb-updateMany.bakedsvg.svg)

### 修改器

现在有个文档products
``` json
{
  _id: 100,
  sku: "abc123",
  quantity: 250,
  instock: true,
  reorder: false,
  details: { model: "14Q2", make: "xyz" },
  tags: [ "apparel", "clothing" ],
  ratings: [ { by: "ijk", rating: 4 } ]
}
```

`$set`修改器

`$set`修改器用于指定一个字段的值，字段不存在时，则会创建字段。 修改错误或不在需要的字段，可以使用`$unset`方法将这个键删除

现在要把文档中 `details.make`字段的值更新为"zzz"

``` shell
db.products.update(
    { _id: 100},
    { $set: {"details.make": "zzz"},
      $currentDate: {lastModified: true}
    }
)
```

其中，$set操作符将details.make字段的值更新为zzz， $currentDate 操作符用来更新lastModified字段的值为当前时间，如果该字段不存在，$currentDate 操作符将会创建这个字段

`$inc`修改器

`$inc`修改器用于字段值的增加和减少

products文档如下：
``` JSON
{
  _id: 1,
  sku: "abc123",
  quantity: 10,
  metrics: {
    orders: 2,
    ratings: 3.5
  }
}
```

将quantity字段的值减2，并且将orders的值加1

``` shell
db.products.update(
    { sku: "abc123"},
    { $inc: {quiantity: -2, "metrics.orders": 1 }}
)
```

## DELETE 操作

**危险**：remove中如果不带参数将删除所有数据

![deleteStetement](https://docs.mongodb.com/manual/_images/crud-annotated-mongodb-deleteMany.bakedsvg.svg)

下面附的是 [MongoDB 的速查手册](http://7xkfga.com1.z0.glb.clouddn.com/MongoDBReferenceCards.pdf)
