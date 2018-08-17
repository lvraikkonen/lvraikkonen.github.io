---
title: RDD API 实操
date: 2018-08-15 18:05:34
tags: 
- Big Data
- Spark
categories:
- Big Data
- Spark
---

RDD是Spark最核心的数据抽象，全称叫Resilient Distributed Datasets(弹性分布式数据集)。`org.apache.spark.rdd.RDD`是一个抽象类，定义了RDD的基本操作和属性

``` scala
// 计算某个分区的数据，返回Iterator
def compute(split: Partition, context: TaskContext): Iterator[T]

// 获取RDD的分区列表
protected def getPartitions: Array[Partition]

// 获取RDD的依赖列表
protected def getDependencies: Seq[Dependency[_]] = deps

// 获取某一个分区数据所在的机器
protected def getPreferredLocations(split: Partition): Seq[String] = Nil

// 分区器
@transient val partitioner: Option[Partitioner] = None
```

<!-- more -->

## 创建RDD

RDD可以有三种创建方式：
1. 从存储系统中创建，例如HDFS、文件等等
2. 从已经存在的RDD中创建，即使用Transform操作
3. 从内存中的列表数据创建

下面是在内存中创建RDD的例子

``` scala
val rdd01 = sc.makeRDD(List(1, 2, 3, 4, 5, 6))

// 两个分区
val rdd02 = sc.parallelize(List(1, 2, 3, 4, 5, 6), 2)
```

RDD支持两类操作：
- 转换(Transform)
- 行动(Action)

当RDD执行转换操作时候，实际计算并没有被执行，只有当RDD执行行动操作时候才会触发计算任务提交，执行相应的计算操作。

首先准备RDD

``` scala
val rddInt:RDD[Int] = sc.makeRDD(List(1,2,3,4,5,6,2,5,1))
val rddStr:RDD[String] = sc.parallelize(Array("a","b","c","d","b","a"), 2)
val rddFile:RDD[String] = sc.textFile("word_count.text", 2)
val rdd01:RDD[Int] = sc.makeRDD(List(1,3,5,3))
val rdd02:RDD[Int] = sc.makeRDD(List(2,4,5,1))
```

## 转换Transform

下面是几个简单的Transform操作

``` scala
/* map操作 */
rddInt.map(x => x + 1).collect()
/* mapPartitions操作 */
rddInt.mapPartitions操作(x => x + 1).collect()
/* filter操作 */
rddInt.filter(x => x > 4).collect()
/* flatMap操作 */
rddFile.flatMap { x => x.split(" ") }.take(5)
/* distinct去重操作 */
rddInt.distinct().collect()
rddStr.distinct().collect()
/* union操作 */
rdd01.union(rdd02).collect()
/* intersection操作 */
rdd01.intersection(rdd02).collect()
/* subtract操作 */
println(rdd01.subtract(rdd02).collect()
/* cartesian操作 */
rdd01.cartesian(rdd02).collect()
```

### `map(func)`

将函数应用于 RDD 中的每个元素，返回值构成新的 RDD

### `flatMap(func)`

类似于 map，但是每个输入项可以映射为0个输出项或更多输出项（打散）

### `filter(func)`

将函数应用于 RDD 中的每个元素，返回 func 函数的值为true的元素形成一个新的 RDD

### `distinct()` 去重

### `union(otherRDD)` 并集

生成一个包含两个 RDD 中所有元素的 RDD。如果输入的 RDD 中有重复数据，union() 操作也会包含这些重复的数据．

### `intersection(otherRDD)`

求两个 RDD 共同的元素的 RDD。 intersection() 在运行时也会去掉所有重复的元素

### `subtract(otherRDD)` 差集

subtract 接受另一个 RDD 作为参数，返回一个由只存在第一个 RDD 中而不存在第二个 RDD 中的所有元素组成的 RDD

下面的Transform操作应用在Key-Value的RDD上。

### `groupByKey()` 分组

根据键值对 key 进行分组。 在（K，V）键值对的数据集上调用时，返回（K，Iterable ）键值对的数据集

note: 基于`combineByKeyWithClassTag`

``` scala
val rdd:RDD[(String,Int)] = sc.makeRDD(List(("k01",3),("k02",6),("k03",2),("k01",26)))
val other:RDD[(String,Int)] = sc.parallelize(List(("k01",29)), 1)

val rddGroup:RDD[(String,Iterable[Int])] = rdd.groupByKey()
// (k01,CompactBuffer(3, 26)),(k03,CompactBuffer(2)),(k02,CompactBuffer(6))
```

> **note:** 如果分组是为了在每个 key 上执行聚合（如求总和或平均值）则使用 reduceByKey 或 aggregateByKey 会有更好的性能。

### `reduceByKey(func, [numTasks])` 根据key聚合

当在（K，V）键值对的数据集上调用时，返回（K，V）键值对的数据集，使用给定的reduce函数 func 聚合每个键的值，该函数类型必须是（V，V）=> V

note: 基于`combineByKeyWithClassTag`

``` scala
val rddReduce:RDD[(String,Int)] = rdd.reduceByKey((x,y) => x + y)
// (k01,29),(k03,2),(k02,6)
```

### `aggregateByKey(zeroValue)(seqOp, combOp, [numPartitions])`

### `sortByKey([ascending], [numPartitions])` 根据key排序

在（K，V）键值对的数据集调用，其中 K 实现 Ordered 接口，按照升序或降序顺序返回按键排序的（K，V）键值对的数据集

``` scala
val rddSortAsc:RDD[(String,Int)] = rdd.sortByKey(true, 1)
// (k01,3),(k01,26),(k02,6),(k03,2)
```

## 行动Action

下面是常见的Action操作

``` scala
/* count操作 */
rddInt.count()
/* countByValue操作 */
rddInt.countByValue()
/* reduce操作 */
rddInt.reduce((x ,y) => x + y)
/* fold操作 */
rddInt.fold(0)((x ,y) => x + y))
/* aggregate操作 */
val res:(Int,Int) = rddInt.aggregate((0,0))((x,y) => 
            (x._1 + x._2,y),(x,y) => 
            (x._1 + x._2,y._1 + y._2))
println(res._1 + "," + res._2)
/* foeach操作 */
rddStr.foreach { x => println(x) }
```

### `reduce(func)`

接收一个函数作为参数，这个函数要操作两个相同元素类型的RDD并返回一个同样类型的新元素

``` scala
rddInt.reduce((x ,y) => x + y)
```

### `collect()`

将整个RDD的内容返回

### `take(n)`

返回 RDD 中的n个元素，并且尝试只访问尽量少的分区，因此该操作会得到一个不均衡的集合

### `saveAsTextFile(path)`

将数据集的元素写入到本地文件系统，HDFS 或任何其他 Hadoop 支持的文件系统中的给定目录的文本文件（或文本文件集合）中

### `saveAsSequenceFile(path)`

将数据集的元素写入到本地文件系统，HDFS 或任何其他 Hadoop 支持的文件系统中的给定路径下的 Hadoop SequenceFile中。这在实现 Hadoop 的 Writable 接口的键值对的 RDD 上可用

### `foreach(func)`

在数据集的每个元素上运行函数 func

## Key-Value RDD

Spark里创建键值对RDD只可以从内存里读取。所有从文件中读取的RDD都是一般的RDD对象，需要进行转化。

对于Pair RDD常见的聚合操作如：reduceByKey，foldByKey，groupByKey，combineByKey，这些API的定义在 [PairRDDFunctions](http://spark.apache.org/docs/2.3.0/api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions)类中。

常见的Key-Value RDD转换操作：

```
reduceByKey：合并具有相同键的值；
groupByKey：对具有相同键的值进行分组；
keys：返回一个仅包含键值的RDD；
values：返回一个仅包含值的RDD；
sortByKey：返回一个根据键值排序的RDD；
flatMapValues：针对Pair RDD中的每个值应用一个返回迭代器的函数，然后对返回的每个元素都生成一个对应原键的键值对记录；
mapValues：对Pair RDD里每一个值应用一个函数，但是不会对键值进行操作；
combineByKey：使用不同的返回类型合并具有相同键的值；
subtractByKey：操作的RDD我们命名为RDD1，参数RDD命名为参数RDD，剔除掉RDD1里和参数RDD中键相同的元素；
join：对两个RDD进行内连接；
rightOuterJoin：对两个RDD进行连接操作，第一个RDD的键必须存在，第二个RDD的键不再第一个RDD里面有那么就会被剔除掉，相同键的值会被合并；
leftOuterJoin：对两个RDD进行连接操作，第二个RDD的键必须存在，第一个RDD的键不再第二个RDD里面有那么就会被剔除掉，相同键的值会被合并；
cogroup：将两个RDD里相同键的数据分组在一起
sample：对RDD采样；
```

常见的行动操作：

```
countByKey：对每个键的元素进行分别计数；
collectAsMap：将结果变成一个map；
lookup：在RDD里使用键值查找数据
take(num):返回RDD里num个元素，随机的；
top(num):返回RDD里最前面的num个元素，这个方法实用性还比较高；
takeSample：从RDD里返回任意一些元素；
sample：对RDD里的数据采样；
takeOrdered：从RDD里按照提供的顺序返回最前面的num个元素
```

### combineByKey

combineByKey是Spark中一个比较核心的高级函数，其他一些高阶键值对函数底层都是用它实现的。例如groupByKey,reduceByKey等等

``` scala
def combineByKey[C](
        createCombiner: V => C,
        mergeValue: (C, V) => C,
        mergeCombiners: (C, C) => C,
        partitioner: Partitioner,
        mapSideCombine: Boolean = true,
        serializer: Serializer = null)
```

其中主要的前三个参数如下：

- `createCombiner` 函数: 组合器函数，用于将RDD[K,V]中的V转换成一个新的值C1；在找到给定分区中第一次碰到的key（在RDD元素中）时被调用。此方法为这个key初始化一个累加器。
- `mergeValue` 函数：合并值函数，将一个C1类型值和一个V类型值合并成一个C2类型，输入参数为(C1,V)，输出为新的C2; 当累加器已经存在的时候（也就是上面那个key的累加器）调用。
- `mergeCombiners` 函数：合并组合器函数，用于将两个C2类型值合并成一个C3类型，输入参数为(C2,C2)，输出为新的C3。如果哪个key跨多个分区，该参数就会被调用。

举个例子说明计算的流程：

现在有一个Key-Value的Pair RDD，两个分区

``` scala
val pairStrRDD = sc.parallelize[(String, Int)](Seq(("coffee", 1), ("coffee", 2), ("tea", 3), ("coffee", 9)), 2)
pairStrRDD.glom().collect()
```

![](http://7xkfga.com1.z0.glb.clouddn.com/51c59a3d5df8da2f5b4e6fb0cbb93b83.jpg)

这个RDD有两个分区，第一个分区是(("Coffee", 1), ("Coffee", 2))，第二个分区是(("Tea", 3), ("Coffee",9))

然后定义combineByKey的前三个参数

``` scala
// 找到给定分区中第一次碰到的key（在RDD元素中）时被调用
def createCombiner = (value: Int) => (value, 1)

// 当累加器已经存在的时候（也就是上面那个key的累加器）调用
def mergeValue = (acc: (Int, Int), value: Int) => 
    (acc._1 + value, acc._2 + 1)

// 如果哪个key跨多个分区，该参数就会被调用
def mergeCombiners = (acc1: (Int, Int), acc2: (Int, Int)) =>
    (acc1._1 + acc2._1, acc1._2 + acc2._2)
```

在分区1中，第一次遇到key "Coffee"，`createCombiner`函数被调用，为"Coffee"产生一个累加器，coffee的值为1，出现次数为1。(1, 1)；第二次遇到key "Coffee"，调用函数`mergeValue`函数，上个累加器的第一个元素加上这次遇到的value，上个累加器的第二个元素加上1作为次数。(1+2, 1+1)。在分区2中，同理得到两个累加器tea: (3, 1) coffee: (9, 1)

"Coffee"这个key值跨越两个分区，函数`mergeCombiners`被调用，coffee: (3+9, 2+1)

所以最后得到的RDD为 coffee (12, 3), tea (3, 1)

``` scala
val testCombineByKeyRDD = pairStrRDD.combineByKey(createCombiner, mergeValue, mergeCombiners)
testCombineByKeyRDD.collect()
```

![](http://7xkfga.com1.z0.glb.clouddn.com/9f69e79eb7f7bc60521730e9ccac7e40.jpg)


### `groupByKey`的实现

上面说过，例如reduceByKey、groupByKey等键值对RDD的转换，都是基于combineByKey的

![](http://7xkfga.com1.z0.glb.clouddn.com/511e7a2c71489a0c362b45cbd726799a.jpg)

源码中，可以看出定义了combineByKey的三个参数

- createCombiner 将原RDD中的K类型转换为Iterable[V]类型，实现为CompactBuffer
- mergeValue 将原RDD的元素追加到CompactBuffer中，即将追加操作(+=)视为合并操作
- mergeCombiners 针对每个key值所对应的Iterable[V]，提供合并功能(c1 ++= c2)

groupByKey函数针对PairRddFunctions的RDD[(K, V)]按照key对value进行分组

[RDD Programming Guide](http://spark.apache.org/docs/2.3.0/rdd-programming-guide.html#rdd-operations)

## 关于Shuffle

大多数 Spark 作业的性能主要就是消耗在了 shuffle 环节，因为该环节包含了大量的磁盘IO、序列化、网络数据传输等操作

## RDD缓存

`persist()` or `cache()`