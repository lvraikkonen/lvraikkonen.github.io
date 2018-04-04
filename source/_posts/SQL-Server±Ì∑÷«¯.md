---
title: SQL Server表分区
date: 2017-06-21 17:53:27
tags:
  - SQL Server
  - Database
  - 表分区

---

某种特定业务下，部分业务数据可能只保留比较短的时间，例如每日的流量日志数据，可能每天亿级别的数据增长，随着数据量的逐渐增长，你会发现数据库能性能越来越慢，查询速度会明显变慢，而这时要想提高数据库的查询速度，你肯定会想到索引这种方式，但是随着索引的引入，数据的插入和更新也会变慢，因为在数据插入的时候，索引也是需要重建的。那怎么办呢？ (其实我觉得这种应用场景更好的解决方法是用流式处理的方式)

一个最简单的解决方法就是把一个大表拆分成多个小表，这个就叫做表分区，表分区有两种：

- 水平分区 (行级)
- 垂直分区 (列级)

下面主要说的是水平分区。

表分区有以下优点：

1. 改善查询性能：对分区对象的查询可以仅搜索自己关心的分区，提高检索速度。
2. 增强可用性：如果表的某个分区出现故障，表在其他分区的数据仍然可用；
3. 维护方便：如果表的某个分区出现故障，需要修复数据，只修复该分区即可；
4. 均衡I/O：可以把不同的分区映射到磁盘以平衡I/O，改善整个系统性能。

分区表是把数据按某种标准划分成区域存储在不同的文件组中，使用分区可以快速而有效地管理和访问数据子集，从而使大型表或索引更易于管理。合理的使用分区会很大程度上提高数据库的性能。

创建分区需要如下个步骤：
1. 创建文件组
2. 创建分区函数
3. 创建分区方案
4. 创建或者修改使用分区方案的表


<!-- more -->

举一个按照时间分区的案例：

确定分区键列的类型(DATETIME)以及分区的边界值:
- 2011-01-01
- 2012-01-01
- 2013-01-01

N个边界值确定 N+1 个分区

![BorderValue](http://7xkfga.com1.z0.glb.clouddn.com/borderValue.png)

## 创建文件组

T-SQL语法

```
alter database <数据库名> add filegroup <文件组名>
```

下面创建4个分区文件组

``` sql
USE AdventureWorksDW2014;  
GO  

-- Adds four new filegroups to the AdventureWorksDW2014 database  
ALTER DATABASE AdventureWorksDW2014  
ADD FILEGROUP test1fg;  
GO  
ALTER DATABASE AdventureWorksDW2014  
ADD FILEGROUP test2fg;  
GO  
ALTER DATABASE AdventureWorksDW2014  
ADD FILEGROUP test3fg;  
GO  
ALTER DATABASE AdventureWorksDW2014  
ADD FILEGROUP test4fg;   

-- Adds one file for each filegroup.  
ALTER DATABASE AdventureWorksDW2014   
ADD FILE   
(  
    NAME = test1dat1,  
    FILENAME = 'E:\FileGroupsData\t1dat1.ndf',  
    SIZE = 5MB,  
    MAXSIZE = 100MB,  
    FILEGROWTH = 5MB  
)  
TO FILEGROUP test1fg;  
ALTER DATABASE AdventureWorksDW2014   
ADD FILE   
(  
    NAME = test2dat2,  
    FILENAME = 'E:\FileGroupsData\t2dat2.ndf',  
    SIZE = 5MB,  
    MAXSIZE = 100MB,  
    FILEGROWTH = 5MB  
)  
TO FILEGROUP test2fg;  
GO  
ALTER DATABASE AdventureWorksDW2014   
ADD FILE   
(  
    NAME = test3dat3,  
    FILENAME = 'E:\FileGroupsData\t3dat3.ndf',  
    SIZE = 5MB,  
    MAXSIZE = 100MB,  
    FILEGROWTH = 5MB  
)  
TO FILEGROUP test3fg;  
GO  
ALTER DATABASE AdventureWorksDW2014   
ADD FILE   
(  
    NAME = test4dat4,  
    FILENAME = 'E:\FileGroupsData\t4dat4.ndf',  
    SIZE = 5MB,  
    MAXSIZE = 100MB,  
    FILEGROWTH = 5MB  
)  
TO FILEGROUP test4fg;  
GO  
```

执行上面的脚本，可以创建4个文件组

![fileGroups](http://7xkfga.com1.z0.glb.clouddn.com/createFileGroup.png)

使用样例数据库的FactResellerSales表做一个分区的实验

``` sql
IF OBJECT_ID('dbo.SalesOrders')IS NOT NULL
DROP TABLE dbo.SalesOrders
GO

SELECT * INTO SalesOrders
FROM [AdventureWorksDW2014].[dbo].[FactResellerSales]
```

## 创建分区函数

指定分依据区列（依据列唯一），分区数据范围规则，分区数量，然后将数据映射到一组分区上。

创建语法：

```
create partition function 分区函数名(<分区列类型>) as range [left/right]
for values (每个分区的边界值,....)
```

``` sql
CREATE PARTITION FUNCTION PF_Orders_OrderDateRange(DATETIME)
    AS RANGE RIGHT FOR VALUES
    (
       '2011-01-01',
       '2012-01-01',
       '2013-01-01'
    )
```

左边界/右边界：就是把临界值划分给上一个分区还是下一个分区。这里2013-01-01就属于下一个分区

> 注意：只有没有应用到分区方案中的分区函数才能被删除。

## 创建分区方案

指定分区对应的文件组。

创建语法：

```
-- 创建分区方案语法
create partition scheme <分区方案名称> as partition <分区函数名称> [all]to (文件组名称,....)
```

``` sql
CREATE PARTITION SCHEME PS_Orders
    AS PARTITION PF_Orders_OrderDateRange
    TO (test1fg, test2fg, test3fg, test4fg) ;
GO   
```

> 注意：只有没有分区表，或索引使用该分区方案是，才能对其删除。

## 创建使用分区方案的表

创建语法：

```
--创建分区表语法
create table <表名> (
  <列定义>
)on<分区方案名>(分区列名)
```

下面在已有的表SalesOrders上面应用分区方案，并在OrderDate字段上创建聚集索引

``` sql
CREATE CLUSTERED INDEX IX_FactResellerSales_OrderDate
  ON dbo.SalesOrders (OrderDate)
  WITH (STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF,
         ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON)
  ON PS_Orders(OrderDate) -- AssignPartitionScheme  
-- 这里使用[PS_Orders]分区方案，根据OrderDate列进行分区
```




**在创建分区表后，需要创建聚集分区索引**

根据订单表Orders 查询时经常使用OrderDate 范围条件来查询的特点，我们最好在Orders.OrderDate 列上建立聚集索引（clustered index）。为了便于进行分区切换（partition swtich)。
大多数情况下，建议在分区表上建立分区索引。


## 查询分区表信息

### 查看分区依据列的指定值所在的分区

``` sql
-- 查询分区依据列为'2013-01-01'的数据在哪个分区上
SELECT $partition.PF_Orders_OrderDateRange('2013-01-01')  
-- 返回值是3，表示此值存在第2个分区
```

### 查询每个非空分区存在的行数

``` sql
-- 查看分区表中，每个非空分区存在的行数
SELECT $partition.PF_Orders_OrderDateRange(OrderDate) as partitionNum
     , count(*) as recordCount
FROM dbo.SalesOrders
GROUP BY $partition.PF_Orders_OrderDateRange(OrderDate)
```

### 查询各个分区的数据信息

``` sql
SELECT PARTITION = $PARTITION.PF_Orders_OrderDateRange(OrderDate),
       ROWS      = COUNT(*),
       MinVal    = MIN(OrderDate),
       MaxVal    = MAX(OrderDate)
FROM [dbo].[SalesOrders]
GROUP BY $PARTITION.PF_Orders_OrderDateRange(OrderDate)
ORDER BY PARTITION
```

![partitionStatus](http://7xkfga.com1.z0.glb.clouddn.com/SalesOrderPartitionStatus.png)

## 分区的拆分合并

### 拆分分区

在分区函数中新增一个边界值，即可将一个分区变为2个。

``` sql
--分区拆分
alter partition function PF_Orders_OrderDateRange()
split range('2014-01-01')
```

> 注意：如果分区函数已经指定了分区方案，则分区数需要和分区方案中指定的文件组个数保持对应一致。


### 合并分区

与拆分分区相反，去除一个边界值即可。

``` sql
--合并分区
alter partition function PF_Orders_OrderDateRange()
merge range('2011-01-01')
```

## 分区数据移动

### 分区数据移动

可以使用 `ALTER TABLE ....... SWITCH` 语句快速有效地移动数据子集：

- 将某个表中的数据移动到另一个表中；
- 将某个表作为分区添加到现存的已分区表中；
- 将分区从一个已分区表切换到另一个已分区表；
- 删除分区以形成单个表。


### 切换分区表的一个分区到普通数据表

创建普通表SalesOrder_2012用于存放订单日期为2012年的所有数据。从分区到普通表的切换，最好满足以下条件：

1. **普通表必须建立在分区表切换出的分区所在的文件组上**
2. **普通表的表结构和分区表一致**
3. **普通表上的索引要和分区表一致，包括聚集索引和非聚集索引**
4. **普通表必须是空表**

切换分区为3的数据从分区表到归档表

``` sql
ALTER TABLE dbo.SalesOrders SWITCH PARTITION 3
TO dbo.SalesOrders_2012
```

现在再查看一下分区表的分区状态，分区为3的数据已经不在了

![partitionStatusAfterMoveOneOut](http://7xkfga.com1.z0.glb.clouddn.com/partitionStatusAfterOneMoved.png)

### 切换普通表数据到分区表的一个分区中

下面要把上面已经归档的2012年的数据切换回来

按照上面的那种写法先试一下：

```sql
ALTER TABLE dbo.SalesOrders_2012 SWITCH TO
dbo.SalesOrders PARTITION 3
```

这时候会遇到错误

```
Msg 4982, Level 16, State 1, Line 1
ALTER TABLE SWITCH statement failed.
Check constraints of source table 'AdventureWorksDW2014.dbo.SalesOrders_2012'
allow values that are not allowed by range defined by partition 3 on target table 'AdventureWorksDW2014.dbo.SalesOrders'.
```

这是因为表dbo.SalesOrders 的数据经过分区函数的分区列定义, 各个分区的数据实际上已经经过
了数据约束检查，符合分区边界范围(Range)的数据才会录入到各个分区中。
但是在存档表dbo.SalesOrders_2012中的数据实际上是没有边界约束的，比如完全可以手动的插入一条其他年的数据，所以进行SWITCH时肯定是不会成功的，这时候需要增加一个数据约束检查

``` sql
ALTER TABLE dbo.SalesOrders_2012 ADD CONSTRAINT CK_SalesOrders_OrderDate
CHECK(OrderDate>='2012-01-01' AND OrderDate<'2013-01-01')
```

这时候再SWITCH，2012年扥分区数据就会到了分区表中。

### 切换分区表数据到分区表

新的存档分区表在结构上和源分区表是一致的，包括分区函数和分区方案，
但是需要重新创建，不能简单地直接使用dbo.SalesOrders 表上的分区函和分区方案，因为他们之间有绑定关系

创建分区函数和分区方案

``` sql
IF EXISTS (SELECT * FROM sys.partition_schemes WHERE name = 'PS_SalesOrdersArchive')
DROP PARTITION SCHEME PS_SalesOrdersArchive
GO

IF EXISTS (SELECT * FROM sys.partition_functions WHERE name = 'PF_SalesOrdersArchive_OrderDateRange')
DROP PARTITION FUNCTION PF_SalesOrdersArchive_OrderDateRange
GO

CREATE PARTITION FUNCTION PF_SalesOrdersArchive_OrderDateRange(DATETIME)
AS RANGE RIGHT FOR VALUES
(
   '2011-01-01',
   '2012-01-01',
   '2013-01-01'
)
GO

CREATE PARTITION SCHEME PS_SalesOrdersArchive
AS PARTITION PF_SalesOrdersArchive_OrderDateRange
TO (test1fg, test2fg, test3fg, test4fg)
GO
```

创建归档表

``` sql
CREATE TABLE [dbo].[SalesOrdersArchive](
	[ProductKey] [int] NOT NULL,
	[OrderDateKey] [int] NOT NULL,
	...
	[OrderDate] [datetime] NOT NULL,
	[DueDate] [datetime] NULL,
	[ShipDate] [datetime] NULL
) ON PS_SalesOrdersArchive(OrderDate)


CREATE CLUSTERED INDEX IXC_SalesOrdersArchive_OrderDate ON dbo.SalesOrdersArchive(OrderDate)
```

切换分区到归档表

```sql
ALTER TABLE dbo.SalesOrders SWITCH PARTITION 1 TO dbo.SalesOrdersArchive PARTITION 1
ALTER TABLE dbo.SalesOrders SWITCH PARTITION 2 TO dbo.SalesOrdersArchive PARTITION 2
ALTER TABLE dbo.SalesOrders SWITCH PARTITION 3 TO dbo.SalesOrdersArchive PARTITION 3
```

切换完成后，观察一下原表和归档表的分区数据状况：

原表：

![OriginTablePartitionStatus](http://7xkfga.com1.z0.glb.clouddn.com/OriginTablePatitionStatus.png)

归档表：

![ArchieveTablePatitionStatus](http://7xkfga.com1.z0.glb.clouddn.com/ArchiveTablePatition.png)

## 总结

分区表分区切换并没有真正去移动数据,而是SQL Server 在系统底层改变了表的元数据。因此分区表分区切换是高效、快速、灵活的。利用分区表的分区切换功能，我们可以快速加载数据到分区表、卸载分区数据到普通表，然后TRUNCATE普通表，以实现快速删除分区表数据，快速归档不活跃数据到历史表。

表分区的相关概念和实际操作就介绍到这儿，下一篇重点介绍一下如何实现表分区随着时间窗口的移动而自动维护。
