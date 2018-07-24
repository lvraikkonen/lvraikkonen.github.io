---
title: ETL增量数据的捕获
date: 2018-05-07 18:59:15
tags:
- SQL Server
- ETL
---

在日常的数据操作中，有种很常见的需求，就是要捕获上游数据源的变化，来对增量数据进行处理，如何捕捉变化的数据就变成了ETL过程的主要问题，一般对捕获过程有两个要求：1. 要按照一定的频率准确捕获到增量数据， 2. 不能对业务系统造成太大压力。

常见的捕获增量数据的方法有以下几种：

1. 触发器
2. 时间戳
3. 全表对比
4. 日志监控

下面逐个分析一下

<!-- more -->

## 触发器

在被抽取的源表上建立插入、修改、删除三种触发器，当源表中数据发生变化，相应的触发器将变化的数据写入一个增量日志表，日志表只存储源表名称、更新关键字值和操作类型（insert，update，delete）。ETL先从日志表取源表名称和关键字值，再去源表抽取完整记录，根据操作类型对目标表做处理。这样的缺点很明显，会对业务系统的性能造成极大的影响。

## 时间戳

在源表上增加一个时间戳字段，系统中更新修改表数据的时候，同时修改时间戳字段的值。当进行数据抽取时，通过比较系统时间与时间戳字段的值来决定抽取哪些数据。对不支持时间戳的自动更新的数据库，需要要求业务系统进行额外的更新时间戳操作，业务系统的人可能会不愿意，并且时间戳的方式无法捕获delete操作(可以要求业务系统实现软删除来避免)

## 全表对比

典型的全表比对的方式是采用MD5校验码。ETL工具事先为要抽取的表建立一个结构类似的MD5临时表，该临时表记录源表主键以及根据所有字段的数据计算出来的MD5校验码。每次进行数据抽取时，对源表和MD5临时表进行MD5校验码的比对，从而决定源表中的数据是新增、修改还是删除，同时更新MD5校验码。这种方式性能较差，并且对于没有主键或者有重复数据的表，准确性也较差。

## 日志对比

数据的增删改在数据库中是要记录log的，例如对于MySQL，可以通过Canel，Databus，Puma等工具读binlog来获取MySQL的增删改等操作来获取增量数据。SQL Server2008之后，提供了CDC(Change Data Capture)功能来实现对增量数据的捕获，CDC可以以异步进程读取事务日志进行捕获数据变更，这样对业务系统的影响比较小。

下面用一个例子来实际操作一下CDC

## 使用CDC捕捉增量数据

### 启用CDC

``` SQL
--把dbowner设为sa，否则会提示权限不足
EXEC sp_changedbowner 'sa'
GO

--查看数据库是否启用CDC
SELECT name, is_cdc_enabled FROM sys.databases WHERE name = 'AdventureWorks2012'

--启用数据库CDC
USE AdventureWorks2012
GO
EXECUTE sys.sp_cdc_enable_db;
GO

--检查启用是否成功
SELECT is_cdc_enabled,CASE WHEN is_cdc_enabled=0 THEN 'CDC功能禁用' ELSE 'CDC功能启用' END 描述
FROM sys.databases
WHERE NAME = 'AdventureWorks2012'
```

![enableCDC](http://7xkfga.com1.z0.glb.clouddn.com/ce57a9f8978812c058addc3af6d422ab.jpg)

这时候发现数据库的用户多了一个叫cdc的用户，并且多了一个cdc的schema

![cdcSchema](http://7xkfga.com1.z0.glb.clouddn.com/089c2b1ace195a7a4f0fbff062ce2b43.jpg)

### 对目标表启用CDC

``` SQL
--创建测试表
USE CDC_DB
GO
CREATE TABLE [dbo].[Department](
    [DepartmentID] [smallint] IDENTITY(1,1) NOT NULL,
    [Name] [nvarchar](200) NULL,
    [GroupName] [nvarchar](50) NOT NULL,
    [ModifiedDate] [datetime] NOT NULL,
    [AddName] [nvarchar](120) NULL,
 CONSTRAINT [PK_Department_DepartmentID] PRIMARY KEY CLUSTERED 
(
    [DepartmentID] ASC
) ON [PRIMARY]
) ON [PRIMARY]
GO

--对表启用捕获
EXEC sys.sp_cdc_enable_table 
    @source_schema= 'dbo',
    @source_name = 'Department',
    @role_name = N'cdc_Admin',
    @capture_instance = DEFAULT,
    @supports_net_changes = 1,
    @index_name = NULL,
    @captured_column_list = NULL,
    @filegroup_name = DEFAULT

--检查是否成功
SELECT name, is_tracked_by_cdc ,
    CASE WHEN is_tracked_by_cdc = 0 THEN 'CDC功能禁用' ELSE 'CDC功能启用' END 描述
FROM sys.tables
WHERE OBJECT_ID= OBJECT_ID('dbo.Department')

--返回某个表的变更捕获配置信息
EXEC sys.sp_cdc_help_change_data_capture 'dbo', 'Department'
```

创建一个测试表，对表行变更启用捕获，为表`Department`启用CDC，首先会在系统表中创建[cdc].[dbo_Department_CT]，会在Agent中创建两个作业，cdc.CDC_DB_capture和cdc.CDC_DB_cleanup，启用表变更捕获需要开启SQL Server Agent服务，不然会报错。每对一个表启用捕获就会生成一个向对应的记录表。

![AGENTJOB](http://7xkfga.com1.z0.glb.clouddn.com/ea84721e1b548dec7bc1c50224ae2d03.jpg)


### 测试插入、更新、删除

``` SQL
--测试插入数据
INSERT  INTO dbo.Department(
    Name ,
    GroupName ,
    ModifiedDate
)VALUES('Marketing','Sales and Marketing',GETDATE())

--测试更新数据
UPDATE dbo.Department SET Name = 'Marketing Group',ModifiedDate = GETDATE()
WHERE Name = 'Marketing'

--测试删除数据
DELETE FROM dbo.Department WHERE Name='Marketing Group'

--查询捕获数据
SELECT * FROM cdc.dbo_Department_CT
```

![RESULT](http://7xkfga.com1.z0.glb.clouddn.com/c3ac87a11081f22773c89f7ff0fc5592.jpg)

对于insert/delete操作，会有对应的一行记录，而对于update，会有两行记录。`__$operation`列：
- 1 = 删除
- 2 = 插入
- 3 = 更新（旧值）
- 4 = 更新（新值）

可以从结果中看出：刚才的语句插入了一条、更新前后的数据、删除一条数据。

### ETL查询指定时间范围的增量数据

``` SQL
SELECT sys.fn_cdc_map_time_to_lsn
('smallest greater than or equal', '2018-05-07 09:00:30') AS BeginLSN

SELECT sys.fn_cdc_map_time_to_lsn
('largest less than or equal', '2018-05-08 23:59:59') AS EndLSN


/******* 查看某时间段所有CDC记录*******/
DECLARE @FromLSN binary(10) =
sys.fn_cdc_map_time_to_lsn
('smallest greater than or equal' , '2018-05-07 09:00:30')

DECLARE @ToLSN binary(10) =
sys.fn_cdc_map_time_to_lsn
('largest less than or equal' , '2018-05-08 23:59:59')

SELECT CASE [__$operation]
    WHEN 1 THEN 'DELETE'
    WHEN 2 THEN 'INSERT'
    WHEN 3 THEN 'Before UPDATE'
    WHEN 4 THEN 'After UPDATE'
    END Operation,[__$operation],[__$update_mask],DepartmentId,Name,GroupName,ModifiedDate,AddName
FROM [cdc].[fn_cdc_get_all_changes_dbo_Department]
(@FromLSN, @ToLSN,  N'all update old')
/*
all 其中的update，只包含新值
all update old 包含新值和旧值
*/
```

![INCREMENTALDATA](http://7xkfga.com1.z0.glb.clouddn.com/07a3719c920d411a372cc48235f973b5.jpg)


## 使用SSIS的CDC控件实现增量数据处理

### 准备

首先创建一个测试表`dbo.DimCustomer_CDC`以CustomerKey作为主键，并在表中插入500条数据，这个表作为源表。

``` SQL
SELECT * into [TestDB].dbo.DimCustomer_CDC
FROM AdventureWorksDW2014.[dbo].[DimCustomer]
WHERE CustomerKey < 11500

IF NOT EXISTS (SELECT * FROM sys.indexes WHERE object_id = OBJECT_ID(N'[dbo].[DimCustomer_CDC]') AND name = N'PK_DimCustomer_CDC')
  ALTER TABLE [dbo].[DimCustomer_CDC] ADD CONSTRAINT [PK_DimCustomer_CDC] PRIMARY KEY CLUSTERED
(
    [CustomerKey] ASC
)
GO

```

然后在这个源表上开启CDC

``` SQL
EXEC sp_changedbowner 'sa'
GO

USE TestDB
GO
EXECUTE sys.sp_cdc_enable_db;
GO

-- Check
SELECT is_cdc_enabled,CASE WHEN is_cdc_enabled=0 THEN N'CDC功能禁用' ELSE N'CDC功能启用' END 描述
FROM sys.databases
WHERE NAME = 'TestDB'

 
EXEC sys.sp_cdc_enable_table
@source_schema = N'dbo',
@source_name = N'DimCustomer_CDC',
@role_name = N'cdc_admin',
@supports_net_changes = 1
 
GO
```

![](http://7xkfga.com1.z0.glb.clouddn.com/bec94a82e08467410419acd004474fa3.jpg)

![](http://7xkfga.com1.z0.glb.clouddn.com/22cd00f2283a5d3de67f1d6c2177bfa6.jpg)

创建一个目标表`dbo.DimCustomer_Destination`

``` sql
SELECT TOP 0 * 
INTO DimCustomer_Destination
FROM DimCustomer_CDC
```

到此， CDC的监控准备工作就做好了

### 全量加载

这个包只需要执行一次，将源表的所有数据加载到目标表，并且记录下起始/结束LSN

使用CDC Control Task标记起始LSN，并将CDC状态写入到`cdc_state`表中

![](http://7xkfga.com1.z0.glb.clouddn.com/79a6e3912453fe6b499634125379e09f.jpg)

然后创建数据流任务，将源表数据全量导入目标表

最后，创建新的CDC Control Task标记结束LSN

![](http://7xkfga.com1.z0.glb.clouddn.com/f1867110c85c023d404cf789fdde19fb.jpg)

运行这个包，发现源表数据已经全部导入到目标表，并且在`cdc_state`表中存储了当前CDC的状态

![](http://7xkfga.com1.z0.glb.clouddn.com/82b60123268013e38f6656e800b8f2c8.jpg)

### 增量加载

这个包可以随时加载源表中的增量数据，每次运行这个包都会记录每次的CDC状态。

首先，创建两个stage临时表缓存更新数据和已删的数据

然后创建CDC Control Task查询上次的CDC状态

![](http://7xkfga.com1.z0.glb.clouddn.com/e3ccbd495ff50767379a81e95a3d18d2.jpg)

接下来增量数据流任务里面，查找出增量数据

![](http://7xkfga.com1.z0.glb.clouddn.com/ce6f3e3fc5169a9451986575b1400a8e.jpg)

CDC Source本质上是去CDC创建的系统表`cdc.dbo_DimCustomer_CDC_CT`中查询变化数据，其中`__$operation`列：
- 1 = 删除
- 2 = 插入
- 3 = 更新（旧值）
- 4 = 更新（新值）

CDC Split能够根据上面的`__$operation`列值，自动分出Insert、Update和Delete，下面就是将这三个output放入指定的表即可。

接下来，通过缓存表来更新或者删除目标表

``` sql
-- batch update
UPDATE dest
SET
    dest.FirstName = stg.FirstName,
    dest.MiddleName = stg.MiddleName,
    dest.LastName = stg.LastName,
    dest.YearlyIncome = stg.YearlyIncome
FROM
    [DimCustomer_Destination] dest,
    [stg_DimCustomer_UPDATES] stg
WHERE
    stg.[CustomerKey] = dest.[CustomerKey]
 
-- batch delete
DELETE FROM [DimCustomer_Destination]
  WHERE[CustomerKey] IN
(
    SELECT [CustomerKey]
    FROM [dbo].[stg_DimCustomer_DELETES]
)
```

接下来创建一个新的CDC Control Task来标记这次增量抽取的范围

最终，增量加载的包如下：

![](http://7xkfga.com1.z0.glb.clouddn.com/f233e2a0442aed2e6199427c8d74b251.jpg)

### 运行增量加载

首先，将源表的数据进行新增和更新

``` SQL
-- Transfer the remaining customer rows
INSERT INTO DimCustomer_CDC
SELECT *
FROM AdventureWorksDW2014.[dbo].[DimCustomer]
WHERE CustomerKey >= 11500
 
-- give 10 people a raise
UPDATE DimCustomer_CDC
SET
    YearlyIncome = YearlyIncome + 10
WHERE
    CustomerKey > 11000 AND CustomerKey <= 11010

-- delete 10 customer
DELETE DimCustomer_CDC
WHERE CustomerKey > 11110 AND CustomerKey <= 11120
```

此时运行增量加载的包，可以发现包已经获取到了新增、更新以及删除的数据：

![](http://7xkfga.com1.z0.glb.clouddn.com/40d2299494ea2a48ef8fb3414be6cfef.jpg)

并且CDC状态也已经更新

![](http://7xkfga.com1.z0.glb.clouddn.com/7c3ca7b6cc10a65b0b3a4b049e3b3964.jpg)

_NOTE: [CDC_STATE的含义](https://docs.microsoft.com/zh-cn/sql/integration-services/data-flow/define-a-state-variable?view=sql-server-2017)_