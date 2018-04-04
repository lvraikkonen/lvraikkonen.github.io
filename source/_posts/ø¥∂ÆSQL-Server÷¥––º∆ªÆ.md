---
title: 看懂SQL Server执行计划
date: 2017-06-02 09:28:00
tags:
  - SQL Server

---


当我们写的SQL语句传到SQL Server的时候，查询分析器会将语句依次进行解析（Parse）、绑定（Bind）、查询优化（Optimization，有时候也被称为简化）、执行（Execution）。除去执行步骤外，前三个步骤之后就生成了执行计划，也就是SQL Server按照该计划获取物理数据方式，最后执行步骤按照执行计划执行查询从而获得结果。

## 执行计划

- 查询优化器对输入的 T-SQL 查询语句通过"计算"而选择出效率最高的一种执行方案，这个执行方案就是执行计划。
- 执行计划可以告诉你这个查询将会被如何执行或者已经被如何执行过，可以通过执行计划看到 SQL 代码中那些效率比较低的地方。
- 查看执行计划的方式我们可以通过图形化的界面，或者文本，或者XML格式查看，这样会比较方便理解执行计划要表达出来的意思。

<!-- more -->

当一个查询被提交到 SQL Server 后，服务器端很多进程实际上要做很多事情来确保数据出入的完整性。

对于 T-SQL 来说, 处理的主要有两个阶段：关系引擎阶段( `relational engine`)和存储引擎阶段( `storage engine`)。

- 关系引擎主要要做的事情就是首先确保 Query 语句能够正确的解析，然后交给查询优化并产生执行计划，然后执行计划就以二进制格式发到存储引擎来更新或者读取数据。

- 存储引擎主要处理的比如像锁、索引的维护和事务等

所以对于执行计划，重点的是关注关系引擎。

## 估算执行计划和实际执行计划

Estimated Execution Plans vs. Actual Execution Plans

它们之间的区别就是: **估算的执行计划** 是从查询优化器来的，是输入关系引擎的，它的执行步骤包括一些运算符等等都是通过一系列的逻辑分析出来的，是一种通过逻辑推算出来的计划，只能代表查询优化器的观点；**实际执行计划** 是真实的执行了"估算执行计划"后的一种真实的结果，是实实在在真实的执行反馈, 是属于存储引擎。


以上描述了关于执行计划的概念，下面以实际案例去解读一些基本语句， 例如`SELECT`, `UPDATE`,`INSERT`, `DELETE` 等查询的执行计划。

------------------------------------我是分隔符-----------------------------------

有大约78个执行计划中的操作符，可以去 [MSDN Book Online](https://msdn.microsoft.com/en-us/library/ms175913.aspx) 随时查

下表表示一下常见的执行计划元素

|  |   |   |
|---|---|---|
|**Select (Result)** | **Sort** | Spool                                     |
|**Clustered Index Scan** | **Key Lookup** | Eager Spool             |
|**NonClustered Index Scan** | **Compute Scalar** | Stream Aggregate |
|**Clustered Index Seek** | Constant Scan | Distribute Streams   |
|**NonClustered Index Seek** | **Table Scan** | Repartition Streams  |
|**Hash Match** | **RID Lookup** | Gather Streams                    |
|**Nested Loops** | **Filter** | Bitmap                              |
|**Merge Join** | Lazy Spool | Split                             |


操作符分为阻断式 `blocking` 和非阻断式`non-blocking`

## 常见操作符的执行计划解释

### Table Scan 表扫描

<img src='https://i-msdn.sec.s-msft.com/dynimg/IC121534.gif' atl="Table Scan" align="left"/><br>


当表中没有聚集索引，又没有合适索引的情况下，会出现这个操作。这个操作是很耗性能的，他的出现也意味着优化器要遍历整张表去查找你所需要的数据

### Clustered Index Scan / Index Scan 聚集索引扫描/非聚集索引扫描

<img src='https://i-msdn.sec.s-msft.com/dynimg/IC72069.gif' atl="Clustered Index Scan" align="left"/><br>

这个图标两个操作都可以使用，一个聚集索引扫描，一个是非聚集索引扫描。

- 聚集索引扫描：聚集索引的数据体积实际是就是表本身，也就是说表有多少行多少列，聚集所有就有多少行多少列，那么聚集索引扫描就跟表扫描差不多，也要进行全表扫描，遍历所有表数据，查找出你想要的数据。

- 非聚集索引扫描：非聚集索引的体积是根据你的索引创建情况而定的，可以只包含你要查询的列。那么进行非聚集索引扫描，便是你非聚集中包含的列的所有行进行遍历，查找出你想要的数据。

看下面这个查询

``` sql
SELECT ct.*
FROM Person.ContactType AS ct;
```

这个表有一个聚簇索引PK_ContactType_ContactTypeID，聚簇索引的叶子结点是存储数据的，所以对于这个聚簇索引的扫描和全表扫面基本类似，基本也是一行一行地进行扫描来满足查询。

![IndexScan](http://7xkfga.com1.z0.glb.clouddn.com/IndexScan.png)


如果在执行计划中遇到索引扫描，说明查询有可能返回比需要更多的行，这时候建议使用 `WHERE`语句去优化查询，确保只是需要的那些行被返回。

### Clustered Index Seek / Index Seek 聚集索引查找/非聚集索引查找

<img src='https://i-msdn.sec.s-msft.com/dynimg/IC322.gif' atl="Clustered Index Scan" align="left"/><br>

聚集索引查找和非聚集索引查找都是使用该图标。

- 聚集索引查找：聚集索引包含整个表的数据，也就是在聚集索引的数据上根据键值取数据。

- 非聚集索引查找：非聚集索引包含创建索引时所包含列的数据，在这些非聚集索引的数据上根据键值取数据。

``` sql
SELECT ct.*
FROM Person.ContactType AS ct
WHERE ct.ContactTypeID = 7
```

![IndexSeek](http://7xkfga.com1.z0.glb.clouddn.com/ClusterIndexSeek.png)

这个表有一个聚簇索引PK_ContactType_ContactTypeID，建在ContactTypeID字段上，查询使用了这个聚簇索引来查找指定的数据。

索引查找和索引扫描不同，使用查找可以让优化器准确地通过键值找到索引的位置。

以上几种查询的性能对比：

- [Table Scan] 表扫描（最慢）：对表记录逐行进行检查
- [Clustered Index Scan] 聚集索引扫描（较慢）：按聚集索引对记录逐行进行检查
- [Index Scan] 索引扫描（普通）：根据索引滤出部分数据在进行逐行检查
- [Index Seek] 索引查找（较快）：根据索引定位记录所在位置再取出记录
- [Clustered Index Seek] 聚集索引查找（最快）：直接根据聚集索引获取记录

### Key Lookup 键值查找

<img src='https://i-msdn.sec.s-msft.com/dynimg/IC165506.gif' atl="Key Lookup" align="left"/><br>

首先需要说的是查找，查找与扫描在性能上完全不是一个级别的，扫描需要遍历整张表，而查找只需要通过键值直接提取数据，返回结果，性能要好。

当你查找的列没有完全被非聚集索引包含，就需要使用键值查找在聚集索引上查找非聚集索引不包含的列。

### RID Lookup RID查找

<img src='https://i-msdn.sec.s-msft.com/dynimg/IC57221.gif' atl="RID Lookup" align="left"/><br>

跟键值查找类似，只不过RID查找，是需要查找的列没有完全被非聚集索引包含，而剩余的列所在的表又不存在聚集索引，不能键值查找，只能根据行表示Rid来查询数据。


``` sql
SELECT p.BusinessEntityID
     , p.LastName
     , p.FirstName
     , p.NameStyle
FROM Person.Person AS p
WHERE p.LastName LIKE 'Jaf%';
```

![keyLookup](http://7xkfga.com1.z0.glb.clouddn.com/keyLookup.png)

`Person.Person` 表有非聚簇索引 `IX_Person_LastName_FirstName_MiddleName`作用在LastName、FirstName和MiddleName列上面，而列 `NameStyle`并没有被非聚集索引所包含，所以需要使用 `KeyLookUp`在聚集索引上查找不包含的列。如果这个列所在的表不存在聚集索引，那就只能通过RId，也就是行号在查询了。

### Sort

对数据集合进行排序，需要注意的是，有些数据集合在索引扫描后是自带排序的。

### Filter

根据出现在having之后的操作运算符，进行筛选

### Computer Scalar

在需要查询的列中需要自定义列，比如count(\*) as cnt , select name+''+age 等会出现此符号。

## JOIN 连接查询

当多表连接时，SQL Server会采用三类不同的连接方式：散列连接，循环嵌套连接，合并连接


### Hash Join

<img src='https://i-msdn.sec.s-msft.com/dynimg/IC113753.gif' atl="Hash Match" align="left"/><br>

这个图标有两种地方用到，一种是表关联，一种是数据聚合运算时 (`GROUP BY`)

下面有两个概念：

> Hashing：在数据库中根据每一行的数据内容，转换成唯一符号格式，存放到临时哈希表中，当需要原始数据时，可以给还原回来。类似加密解密技术，但是他能更有效的支持数据查询。

> Hash Table：通过hashing处理，把数据以key/value的形式存储在表格中，在数据库中他被放在tempdb中。

Hash Join是做大数据集连接时的常用方式，优化器使用两个表中较小（相对较小）的表利用Join Key在内存中建立散列表 `Hash Table`，然后扫描较大的表并探测散列表，找出与Hash表匹配的行。这种方式适用于较小的表完全可以放于内存中的情况

如果在执行计划中见到Hash Match Join，也许应该检查一下是不是缺少或者没有使用索引、没有用到WHERE等等。

### Nested Loops Join

<img src='https://i-msdn.sec.s-msft.com/dynimg/IC138581.gif' atl="Nested Loop Join" align="left"/><br>

这个操作符号，把两个不同列的数据集汇总到一张表中。提示信息中的Output List中有两个数据集，下面的数据集（inner set）会一一扫描与上面的数据集（out set），直到扫描完为止，这个操作才算是完成。

对于被连接的数据子集较小的情况，嵌套循环连接是个较好的选择。在嵌套循环中，内表被外表驱动，外表返回的每一行都要在内表中检索找到与它匹配的行，因此整个查询返回的结果集不能太大

### Merge Join

<img src='https://i-msdn.sec.s-msft.com/dynimg/IC173813.gif' atl="Merge Join" align="left"/><br>

这种关联算法是对两个已经排过序的集合进行合并。如果两个聚合是无序的则将先给集合排序再进行一一合并，由于是排过序的集合，左右两个集合自上而下合并效率是相当快的。

通常情况下散列连接的效果都比排序合并连接要好，然而如果行源已经被排过序，在执行排序合并连接时不需要再排序了，这时排序合并连接的性能会优于散列连接。Merge join 用在没有索引，并且数据已经排序的情况。

``` sql
SELECT c.CustomerID
FROM Sales.SalesOrderDetail od
JOIN Sales.SalesOrderHeader oh
ON od.SalesOrderID = oh.SalesOrderID
JOIN Sales.Customer c ON oh.CustomerID = c.CustomerID
```

![joinPlan](http://7xkfga.com1.z0.glb.clouddn.com/MergeJoin.png)

由于没有使用WHERE语句，所以优化器对Customer表使用聚集索引扫描，对SalesOrderHeader表使用非聚集索引扫描

Customer表和SalesOrderHeader表使用Merge Join操作符进行关联，关联字段是CustomerID字段，这个字段在上面的索引扫描之后都是有序的。如果不是有序的，优化器会在前面进行排序或者是直接将两个表进行Hash Join连接。

## 根据执行计划细节要做的优化操作

1. 如果select * 通常情况下聚集索引会比非聚集索引更优。
2. 如果出现Nested Loops，需要查下是否需要聚集索引，非聚集索引是否可以包含所有需要的列。
3. Hash Match连接操作更适合于需要做Hashing算法集合很小的连接。
4. Merge Join时需要检查下原有的集合是否已经有排序，如果没有排序，使用索引能否解决。
5. 出现表扫描，聚集索引扫描，非聚集索引扫描时，考虑语句是否可以加where限制，select * 是否可以去除不必要的列。
6. 出现Rid查找时，是否可以加索引优化解决。
7. 在计划中看到不是你想要的索引时，看能否在语句中强制使用你想用的索引解决问题，强制使用索引的办法Select CluName1,CluName2 from Table with(index=IndexName)。
8. 看到不是你想要的连接算法时，尝试强制使用你想要的算法解决问题。强制使用连接算法的语句：select * from t1 left join t2 on t1.id=t2.id option(Hash/Loop/Merge Join)
9. 看到不是你想要的聚合算法是，尝试强制使用你想要的聚合算法。强制使用聚合算法的语句示例：select  age ,count(age) as cnt from t1 group by age  option(order/hash group)
10. 看到不是你想要的解析执行顺序是，或这解析顺序耗时过大时，尝试强制使用你定的执行顺序。option（force order）
11. 看到有多个线程来合并执行你的sql语句而影响到性能时，尝试强制是不并行操作。option（maxdop 1）
12. 在存储过程中，由于参数不同导致执行计划不同，也影响啦性能时尝试指定参数来优化。option（optiomize for（@name='zlh'））
13. 不操作多余的列，多余的行，不做务必要的聚合，排序。
