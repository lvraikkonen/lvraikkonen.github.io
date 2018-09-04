---
title: Spark应用运行流程(转)
date: 2018-08-17 12:26:51
tags: 
- Big Data
- Spark
categories:
- Big Data
- Spark
---

[Spark核心技术原理透视一（Spark运行原理）](https://cloud.tencent.com/developer/article/1066566)


## Spark专业术语定义

### Spark应用程序

指的是用户编写的Spark应用程序，包含了Driver功能代码和分布在集群中多个节点上运行的Executor代码。Spark应用程序，由一个或多个作业JOB组成，如下图所示。

![](https://ask.qcloudimg.com/http-save/1484446/w7r82hdm8t.jpeg)

<!-- more -->

### Driver：驱动程序

Spark 中的 Driver 即运行上述 Application 的 Main() 函数并且创建 SparkContext，其中创建 SparkContext 的目的是为了准备 Spark 应用程序的运行环境。在 Spark 中由 SparkContext 负责和 ClusterManager 通信，进行资源的申请、任务的分配和监控等；当 Executor 部分运行完毕后，Driver 负责将 SparkContext 关闭。通常 SparkContext 代表 Driver，如下图所示。

![](https://ask.qcloudimg.com/http-save/1484446/g48nwyw3ps.jpeg)

### Cluster Manager：资源管理器

指的是在集群上获取资源的外部服务，常用的有：Standalone，Spark 原生的资源管理器，由 Master 负责资源的分配；Haddop Yarn，由 Yarn 中的 ResearchManager 负责资源的分配；Messos，由 Messos 中的 Messos Master 负责资源管理，如下图所示。

![](https://ask.qcloudimg.com/http-save/1484446/fu6xmh4z9r.jpeg)

### Executor：执行器

Application 运行在 Worker 节点上的一个进程，该进程负责运行 Task，并且负责将数据存在内存或者磁盘上，每个 Application 都有各自独立的一批 Executor，如下图所示。

![](https://ask.qcloudimg.com/http-save/1484446/v1swunbo1i.jpeg)

### Worker：计算节点

集群中任何可以运行 Application 代码的节点，类似于 Yarn 中的 NodeManager 节点。在Standalone模式中指的就是通过Slave文件配置的Worker节点，在Spark on Yarn模式中指的就是NodeManager节点，在Spark on Messos模式中指的就是Messos Slave节点，如下图所示。

![](https://ask.qcloudimg.com/http-save/1484446/ht7ncbjdde.jpeg)

### RDD：弹性分布式数据集

Resillient Distributed Dataset，Spark的基本计算单元，可以通过一系列算子进行操作（主要有Transformation和Action操作），如下图所示。

![](https://ask.qcloudimg.com/http-save/1484446/0728ga2gim.jpeg)

### 窄依赖

父RDD每一个分区最多被一个子RDD的分区所用；表现为一个父RDD的分区对应于一个子RDD的分区，或两个父RDD的分区对应于一个子RDD 的分区。如图所示。

![](https://ask.qcloudimg.com/http-save/1484446/ttjqzrfdoa.jpeg)

### 宽依赖

父RDD的每个分区都可能被多个子RDD分区所使用，子RDD分区通常对应所有的父RDD分区。如图所示。

![](https://ask.qcloudimg.com/http-save/1484446/d12pinmy21.jpeg)

常见的窄依赖有：map、filter、union、mapPartitions、mapValues、join（父RDD是hash-partitioned ：如果JoinAPI之前被调用的RDD API是宽依赖(存在shuffle), 而且两个join的RDD的分区数量一致，join结果的rdd分区数量也一样，这个时候join api是窄依赖）。

常见的宽依赖有groupByKey、partitionBy、reduceByKey、join（父RDD不是hash-partitioned ：除此之外的，rdd 的join api是宽依赖）。

### DAG：有向无环图

Directed Acycle graph，反应RDD之间的依赖关系，如图所示。

![](https://ask.qcloudimg.com/http-save/1484446/2zg6a5alzx.jpeg)

### DAGScheduler：有向无环图调度器

基于 DAG 划分 Stage 并以 TaskSet 的形势把 Stage 提交给 TaskScheduler；负责将作业拆分成不同阶段的具有依赖关系的多批任务；最重要的任务之一就是：计算作业和任务的依赖关系，制定调度逻辑。在 SparkContext 初始化的过程中被实例化，一个 SparkContext 对应创建一个 DAGScheduler。

![](https://ask.qcloudimg.com/http-save/1484446/frls702hq6.jpeg)

### TaskScheduler：任务调度器

将 Taskset 提交给 worker（集群）运行并回报结果；负责每个具体任务的实际物理调度。如图所示。

![](https://ask.qcloudimg.com/http-save/1484446/kx8exuewzr.jpeg)

### Job：作业

由一个或多个调度阶段所组成的一次计算作业；包含多个Task组成的并行计算，往往由Spark Action催生，一个JOB包含多个RDD及作用于相应RDD上的各种Operation。如图所示。

![](https://ask.qcloudimg.com/http-save/1484446/6c0mqrdqef.jpeg)

### Stage：调度阶段

一个任务集对应的调度阶段；每个Job会被拆分很多组Task，每组任务被称为Stage，也可称TaskSet，一个作业分为多个阶段；Stage分成两种类型ShuffleMapStage、ResultStage。如图所示。

![](https://ask.qcloudimg.com/http-save/1484446/2iz91ugpry.jpeg)

### TaskSet：任务集

由一组关联的，但相互之间没有Shuffle依赖关系的任务所组成的任务集。如图所示。

![](https://ask.qcloudimg.com/http-save/1484446/st71xnk864.jpeg)

- 一个Stage创建一个TaskSet；
- 为Stage的每个Rdd分区创建一个Task,多个Task封装成TaskSet

### Task：任务

被送到某个Executor上的工作任务；单个分区数据集上的最小处理流程单元。如图所示

![](https://ask.qcloudimg.com/http-save/1484446/l0g1qmri60.jpeg)

总体如图所示：

![](https://ask.qcloudimg.com/http-save/1484446/dmbxjaa1wp.jpeg)

## Spark运行基本流程

![](https://ask.qcloudimg.com/http-save/1484446/qf0l01mxtl.jpeg)

![](https://ask.qcloudimg.com/http-save/1484446/wyia35n4le.jpeg)

## Spark运行架构特点

## Spark核心原理透视

### 计算流程

![](https://ask.qcloudimg.com/http-save/1484446/ulabuvzkus.jpeg)

### 从代码构建DAG图

``` scala
val lines1 = sc.textFile(inputPath1).map(···)).map(···)
val lines2 = sc.textFile(inputPath2).map(···)
val lines3 = sc.textFile(inputPath3)
val dtinone1 = lines2.union(lines3)
val dtinone = lines1.join(dtinone1)

dtinone.saveAsTextFile(···)
dtinone.filter(···).foreach(···)
```

Spark的计算发生在RDD的Action操作，而对Action之前的所有Transformation，Spark只是记录下RDD生成的轨迹，而不会触发真正的计算。

Spark内核会在需要计算发生的时刻绘制一张关于计算路径的有向无环图，也就是DAG。

![](https://ask.qcloudimg.com/http-save/1484446/q44dfuw706.jpeg)

### 将DAG划分为Stage核心算法

Application多个job多个Stage：Spark  Application中可以因为不同的Action触发众多的job，一个Application中可以有很多的job，每个job是由一个或者多个Stage构成的，后面的Stage依赖于前面的Stage，也就是说只有前面依赖的Stage计算完毕后，后面的Stage才会运行。

划分依据：Stage划分的依据就是宽依赖，何时产生宽依赖，reduceByKey, groupByKey等算子，会导致宽依赖的产生。

核心算法：从后往前回溯，遇到窄依赖加入本stage，遇见宽依赖进行Stage切分。Spark内核会从触发Action操作的那个RDD开始从后往前推，首先会为最后一个RDD创建一个stage，然后继续倒推，如果发现对某个RDD是宽依赖，那么就会将宽依赖的那个RDD创建一个新的stage，那个RDD就是新的stage的最后一个RDD。然后依次类推，继续继续倒推，根据窄依赖或者宽依赖进行stage的划分，直到所有的RDD全部遍历完成为止。

### 将DAG划分为Stage剖析

从HDFS中读入数据生成3个不同的RDD，通过一系列transformation操作后再将计算结果保存回HDFS。可以看到这个DAG中只有join操作是一个宽依赖，Spark内核会以此为边界将其前后划分成不同的Stage.   同时我们可以注意到，在图中Stage2中，从map到union都是窄依赖，这两步操作可以形成一个流水线操作，通过map操作生成的partition可以不用等待整个RDD计算结束，而是继续进行union操作，这样大大提高了计算的效率。

![](https://ask.qcloudimg.com/http-save/1484446/8412ipj61a.jpeg)

