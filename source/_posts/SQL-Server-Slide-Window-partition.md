---
title: SQL Server Slide Window partition
date: 2017-06-22 11:06:12
tags:
- SQL Server
- Database
- 表分区
---


[上篇文章](https://lvraikkonen.github.io/2017/06/21/SQL-Server%E8%A1%A8%E5%88%86%E5%8C%BA/)详细介绍了表分区的概念和如何创建分区表。


一个分区表创建好之后，随着数据量的逐渐增长，历史数据越来越不受待见：查询和更新频率越来越低，那么把这些历史数据进行归档就十分必要了。如果使用常规的SELECT、INSERT、DELETE进行历史数据归档的话，会出现以下的一系列问题：

- DELETE效率太低

  由于DELETE单个语句是一个事务性的语句，要么全部成功，要么全部失败。那么可想如果删除的是亿级别的数据，那么日志增长,IO负荷非常的大。

- 迁移过程表会被锁住，这样可能会出现死锁，一旦迁移失败，又会造成更大的IO问题。

这时候，在SQL Server中分区表有一个非常实用的语句 `ALTER TABLE …SWITCH`，这个DDL可以快速的将同文件组的表的某个分区迅速的转移到另外的表。这个是利用数据的位置偏移量的指针的转移到新表的方法来实现的。这种方案转移数据非常快。

<!-- more -->

## 滑动窗口 Sliding Window

滑动窗口是为了在一个分区表上维持固定分区的方法，当新数据来的时候，创建新的数据分区、老的分区被移出分区表被删除或者被归档。由于滑动窗口操作在SQL Server中是元数据操作，所以速度会非常快。

下面以AdventureWorks数据库中表FactResellerSales出发，从0开始将该表进行分区，并且以滑动窗口的方式进行动态分区的维护并归档历史数据。

## Table Partition实践

### 创建测试数据库

这个测试数据库有两个文件组，PRIMARY文件组在E盘，SECONDARY文件组在D盘，用于归档数据用。

``` sql
CREATE DATABASE [PartitionDB]
CONTAINMENT = NONE
ON  PRIMARY
(NAME = N'PartitionDB', FILENAME = N'E:\SQLServerData\PartitionDB.mdf', SIZE = 10240KB, FILEGROWTH = 1024KB ),
FILEGROUP [SECONDARY]
(NAME = N'PartitionDBArchive', FILENAME = N'D:\SQLServerData\PartitionDBArchive.ndf', SIZE = 4096KB, FILEGROWTH = 1024KB )
LOG ON
(NAME = N'PartitionDB_log', FILENAME = N'E:\SQLServerData\PartitionDB_log.ldf', SIZE = 1024KB, FILEGROWTH = 10%)
GO

USE [PartitionDB]
GO
IF NOT EXISTS (SELECT name FROM sys.filegroups
                WHERE is_default=1 AND name = N'PRIMARY')
    ALTER DATABASE [PartitionDB] MODIFY FILEGROUP [PRIMARY] DEFAULT
GO
```

![DatabaseFileGroups](http://7xkfga.com1.z0.glb.clouddn.com/DBFileGroups.png)

### 创建测试数据表

以AdventureWorks的FactResellerSales表为例

``` sql
Select * INTO FactResellerSales
FROM [AdventureWorksDW2014].[dbo].[FactResellerSales]
```

在测试表数据导入之后，查看一下当前的表分区情况

``` sql
SELECT o.name objectname,i.name indexname, partition_id, partition_number, [rows]
FROM sys.partitions p
INNER JOIN sys.objects o ON o.object_id=p.object_id
INNER JOIN sys.indexes i ON i.object_id=p.object_id and p.index_id=i.index_id
WHERE o.name LIKE '%FactResellerSales%'
```

如下图：

![startPartition](http://7xkfga.com1.z0.glb.clouddn.com/startPartitionStatus.png)

可以看见，现在这个表有一个默认的分区，并且全部数据都在这个分区里面。

### 在现有表上创建分区

现在要在这个表上创建两个分区，根据OrderDate字段分为两个分区，一个小于2016-01-01，另外分区的数据大于等于这个时间(2016-01-01的数据分配到右面的分区)。按照上面文章写的步骤创建分区

步骤1： 创建分区函数

``` sql
CREATE PARTITION FUNCTION [myPartitionRange] (DATETIME)
    AS RANGE RIGHT FOR VALUES ('2016-01-01')
GO
```

步骤2：创建分区方案

在上面的两个文件组(PRIMARY, SECONDARY)上创建分区方案

``` sql
CREATE PARTITION SCHEME myPartitionScheme
    AS PARTITION [myPartitionRange]
    TO ([SECONDARY],[PRIMARY])
```

步骤3：在表上创建聚集索引并将分区方案作用在该字段上

这里由于要根据字段 `OrderDate` 进行分区并归档，所以在该字段上创建聚集索引。

``` sql
CREATE CLUSTERED INDEX IX_FactResellerSales_OrderDate
  ON FactResellerSales (OrderDate)
  WITH (STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF,
         ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON)
  ON myPartitionScheme(OrderDate) -- AssignPartitionScheme
```

在创建好索引并应用分区方案后，再查看一下现在的表分区状态

``` sql
SELECT o.name objectname,i.name indexname, partition_id, partition_number, [rows]
FROM sys.partitions p
INNER JOIN sys.objects o ON o.object_id=p.object_id
INNER JOIN sys.indexes i ON i.object_id=p.object_id and p.index_id=i.index_id
WHERE o.name LIKE '%FactResellerSales%'
```

![newPartitionStatus](http://7xkfga.com1.z0.glb.clouddn.com/newPartitionStatus.png)

可以看见，现在有两个分区：1、2，其中全部数据都在第一个分区里面，一共有60855条，也就是说60855条销售数据的OrderDate < '2016-01-01'

### 测试新数据

现在分区已经创建好，向这个表插入三条新的2016-01-01之后的数据

``` sql
INSERT INTO dbo.FactResellerSales
(ProductKey, OrderDateKey, DueDateKey, ShipDateKey, ResellerKey, EmployeeKey, PromotionKey, CurrencyKey ,SalesTerritoryKey
 , SalesOrderNumber, SalesOrderLineNumber, RevisionNumber, OrderQuantity, UnitPrice, ExtendedAmount, UnitPriceDiscountPct, DiscountAmount
 , ProductStandardCost, TotalProductCost, SalesAmount, TaxAmt, Freight, CarrierTrackingNumber, CustomerPONumber, OrderDate, DueDate, ShipDate)
VALUES
(592, 20160101, 20160101, 20160101, 490, 281, 16, 100, 4, 'SO71952', 42, 1, 3, 20, 60, 0, 0, 50, 60, 2, 0, 0, '9490-4552-81', 'PO9715163911', '2016-01-01 00:00:00.000',  
'2016-01-01 00:00:00.000', '2016-01-01 00:00:00.000')
, (592, 20160101, 20160101, 20160101, 490, 281, 16, 100, 4, 'SO71952', 42, 1, 3, 20, 60, 0, 0, 50, 60, 2, 0, 0, '9490-4552-81', 'PO9715163911', '2016-01-02 00:00:00.000',  
'2016-01-02 00:00:00.000', '2016-01-02 00:00:00.000')
, (592, 20160101, 20160101, 20160101, 490, 281, 16, 100, 4, 'SO71952', 42, 1, 3, 20, 60, 0, 0, 50, 60, 2, 0, 0, '9490-4552-81', 'PO9715163911', '2016-01-03 00:00:00.000',  
'2016-01-03 00:00:00.000', '2016-01-03 00:00:00.000')
```


在观察一下分区的情况，发现新插入的3条数据都在第二个分区里面

![newPartitionData](http://7xkfga.com1.z0.glb.clouddn.com/newPartitionData.png)

### 滑动窗口

假设销售数据到了2017年，现在的任务就是把现有的两个分区合并，2016年以及以前的数据放到老的分区里面(D盘上面的那个文件组中)，2017年的数据插入到新的分区里面。

在split partition之前，必须使用alter partition scheme 指定一个NEXT USED FileGroup。如果Partiton Scheme没有指定 next used filegroup，那么alter partition function split range command 执行失败

``` sql
DECLARE @CurrentYear DATETIME='2017-01-01'
DECLARE @PrevMax DATETIME=(SELECT CONVERT(DATETIME,Value) FROM sys.partition_functions f
                           INNER JOIN sys.partition_range_values r   
                           ON f.function_id = r.function_id
                           WHERE f.name = 'myPartitionRange')
IF @PrevMax<@CurrentYear
BEGIN TRY
BEGIN TRAN
-- Merge Old Partitions
ALTER PARTITION FUNCTION myPartitionRange()
MERGE RANGE (@PrevMax)

-- Assign NEXT USED filegroup
ALTER PARTITION SCHEME myPartitionScheme
NEXT USED [PRIMARY]

-- Split partition
ALTER PARTITION FUNCTION myPartitionRange()
SPLIT RANGE (@CurrentYear)

COMMIT TRAN
PRINT 'COMITIINGGGGGG'
END TRY

BEGIN CATCH
IF @@TRANCOUNT>0
ROLLBACK TRAN
END CATCH
```

执行完上面的步骤后，你会发现，2016年的那3条数据也被合并到第一个分区里面了，新的分区2数据为空，留给2017年

![mergePartition](http://7xkfga.com1.z0.glb.clouddn.com/mergePartitionData.png)

再插入3条2017年的数据试验一下

``` sql
INSERT INTO dbo.FactResellerSales
(ProductKey, OrderDateKey, DueDateKey, ShipDateKey, ResellerKey, EmployeeKey, PromotionKey, CurrencyKey, SalesTerritoryKey,
SalesOrderNumber, SalesOrderLineNumber, RevisionNumber, OrderQuantity, UnitPrice, ExtendedAmount, UnitPriceDiscountPct, DiscountAmount
, ProductStandardCost, TotalProductCost, SalesAmount, TaxAmt, Freight, CarrierTrackingNumber, CustomerPONumber, OrderDate, DueDate, ShipDate)
VALUES
(592, 20170101, 20170101, 20170101, 490, 281, 16, 100, 4, 'SO71952', 42, 1, 3, 20, 60, 0, 0, 50, 60, 2, 0, 0, '9490-4552-81', 'PO9715163911', '2017-01-01 00:00:00.000',  
'2017-01-01 00:00:00.000', '2017-01-01 00:00:00.000')
, (592, 20170102, 20170102, 20170102, 490, 281, 16, 100, 4, 'SO71952', 42, 1, 3, 20, 60, 0, 0, 50, 60, 2, 0, 0, '9490-4552-81', 'PO9715163911', '2017-01-02 00:00:00.000',  
'2017-01-02 00:00:00.000', '2017-01-02 00:00:00.000')
, (592, 20170103, 20170103, 20170103, 490, 281, 16, 100, 4, 'SO71952', 42, 1, 3, 20, 60, 0, 0, 50, 60, 2, 0, 0, '9490-4552-81', 'PO9715163911', '2017-01-03 00:00:00.000',  
'2017-01-03 00:00:00.000', '2017-01-03 00:00:00.000')
```

![2017PartitionData](http://7xkfga.com1.z0.glb.clouddn.com/2017PartitionData.png)


现在，2016年以及更久之前的数据保存在分区1中，文件组存储在D盘上，而2017年最新的数据在分区2中，存储在E盘上，这样，就把新老数据在存储上分开，利用更好存储设备的性能查询和更新操作等。而对于查询或者操作这个表的用户来说，逻辑上这还是一张表。


当然，还有另外一种方式进行老数据的归档操作，那就是两张物理表，一张仅存储最新数据，另一张存储归档数据
