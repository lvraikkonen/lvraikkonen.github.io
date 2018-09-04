---
title: Spark-Streaming入门和实践
date: 2018-08-31 14:01:19
tags: 
- Big Data
- Spark
- 流处理
categories:
- Big Data
- Spark
---


## 流处理类型

Spark Streaming是Spark解决方案中实时处理的组件，本质是将数据源分割为很小的批次，以类似离线批处理的方式处理这部分数据。这种方式提升了数据吞吐能力，但是也增加了数据处理的延迟，延迟通常是秒级或者分钟级。

![Mini-Batch data process](http://spark.apache.org/docs/latest/img/streaming-flow.png)

Spark Streaming底层依赖 Spark Core的 RDD，内部的调度方式也依赖于DAG调度器。Spark Streaming的离散数据流DStream本质上是RDD在流式数据上的抽象。

<!-- more -->

## 编写Spark Streaming应用

添加下述依赖到你的Maven项目中：

``` xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming_2.11</artifactId>
    <version>2.3.0</version>
</dependency>
```

下面是WordCount程序的Streaming版本：

``` scala
import org.apache.spark._
import org.apache.spark.streaming._
import org.apache.spark.streaming.StreamingContext._ // not necessary since Spark 1.3

// Create a local StreamingContext with two working thread and batch interval of 1 second.

val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
val ssc = new StreamingContext(conf, Seconds(1))

val lines = ssc.socketTextStream("localhost", 9999)
val words = lines.flatMap(_.split(" "))
val pairs = words.map(word => (word, 1))
val wordCounts = pairs.reduceByKey(_ + _)

wordCounts.print()

ssc.start()             // Start the computation
ssc.awaitTermination()  // Wait for the computation to terminate
```

这个HelloWorld程序主要有下面几个步骤组成：

1. 初始化StreamingContext
2. 创建DStream接收器 (会单独起一个线程运行接收器)
3. 定义DStream的转换操作
4. 定义DStream的输出
5. 启动流处理程序

## 接收器

接收器有两类：基础的接收器(可以自己实现) 和高级数据源(例如 Kafka, Flume等)

下面模拟实现一个自定义的接收器，需要继承Receiver抽象类，然后实现 `onStart()`方法和`onStop()`方法

``` scala
class CustomReceiver(host: String, port: Int)
  extends Receiver[String](StorageLevel.MEMORY_AND_DISK_2) with Logging {

  override def onStart(): Unit = {
    // 启动进程接收数据
    new Thread("Socket Receiver"){
      override def run(){ receive() }
    }.start()
  }

  override def onStop(): Unit = {

  }

  // create a customize socket connection and receive data until receiver is stopped
  private def receive(): Unit ={

    var socket: Socket = null
    var userInput: String = null
    try {
      // connect to host:port
      socket = new Socket(host, port)

      val reader = new BufferedReader(new InputStreamReader(socket.getInputStream(), "UTF-8"))
      userInput = reader.readLine()
      while(!isStopped() && userInput != null){
        store(userInput)
        userInput = reader.readLine()
      }
      reader.close()
      socket.close()

      restart("Trying to connect again")

    }catch {
      case e: java.net.ConnectException =>
        restart("Error connecting to " + host + ":" + port, e)
      case t: Throwable =>
        restart("Error receiving data", t)
    }
  }
}
```

[Spark Streaming Custom Receivers](http://spark.apache.org/docs/latest/streaming-custom-receivers.html)

关于例如Kafka和Flume等高级数据源，下面会有一章介绍

## DStream的转换操作

上面说过DStream本质上是RDD在流式数据上的抽象，所以RDD上很多的转换操作在这里都能通用，例如下面几个：
- map(func)
- flatMap(func)
- filter(func)
- repartition(numPartitions)
- union(otherStream)
- count()
- reduce(func)
- countByValue()
- reduceByKey(func, [numTasks])
- join(otherStream, [numTasks])
- cogroup(otherStream, [numTasks])
- transform(func)	
- updateStateByKey(func)

下面是流处理需要处理的窗口函数(Window Operations)

### Window Operation 窗口操作

![WindowOperation](http://spark.apache.org/docs/latest/img/streaming-dstream-window.png)

在窗口操作中需要指定两个参数:

- window length 窗口长度 —— 窗口的时间长度（上图中为3）
- sliding interval 滑动间隔 —— 窗口操作的执行间隔（上图中为2）

这两个参数必须为流处理定义的批处理间隔的整数倍。

``` scala
// Reduce last 30 seconds of data, every 10 seconds
val windowedWordCounts = pairs.reduceByKeyAndWindow((a:Int,b:Int) => (a + b), Seconds(30), Seconds(10))
```

## DStreams上的输出操作

### foreachRDD

`dstream.foreachRDD`是一个功能强大的原语primitive，它允许将数据发送到外部系统。下面将结果保存到数据库中

``` scala
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    // ConnectionPool is a static, lazily initialized pool of connections
    val connection = ConnectionPool.getConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    ConnectionPool.returnConnection(connection)  // return to the pool for future reuse
  }
}
```

## Caching/Persistence 缓存/持久化

类似于RDD，DStream也允许开发者在内存中持久化stream流数据

## Checkpoint 设置

一个Streaming程序是需要7X24运行的，所以故障恢复能力是很重要的，为此，Spark Streaming需要检查点以便从故障中恢复。

有两种类型的检查点：

1. Metadata检查点
2. 数据检查点，将生成RDD保存在可靠的存储中

note: 如果在应用程序中使用`updateStateByKey`或者`reduceByKeyAndWindow` (带有反转函数)，那么必须提供checkpointing目录

## 集成Kafka、Flume等

下面是集成Kafka的方法，首先添加依赖包

``` xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming-kafka-0-10_2.11</artifactId>
    <version>2.3.0</version>
</dependency>
```

**这里有个坑: 版本不匹配的依赖可能会产生很多兼容性问题**，由于Kafka 0.10.0之后引入了新的Kafka Consumer API，所以现在的这个版本的集成还是实验性的，以后可能还会有变化。Spark 2.3.0开始，kafka-0-8的依赖不再被支持。

![](http://7xkfga.com1.z0.glb.clouddn.com/a6df27183cf689b28129f31bc93527a1.jpg)

### 连接方式的比较

- Kafka consumer传统消费者（老方式）需要连接zookeeper，简称Receiver方式，是高级的消费API，自动更新偏移量，支持WAL，但是效率比较低。

- 新的方式（高效的方式）不需要连接Zookeeper，但是需要自己维护偏移量，简称直连方式，直接连在broker上，但是需要手动维护偏移量，以迭代器的方式边接收数据边处理，效率较高。    

Kafka 0.10以后的只支持直连方式

### 创建DirectStream

``` scala
import org.apache.kafka.clients.consumer.ConsumerRecord
import org.apache.kafka.common.serialization.StringDeserializer
import org.apache.spark.streaming.kafka010._
import org.apache.spark.streaming.kafka010.LocationStrategies.PreferConsistent
import org.apache.spark.streaming.kafka010.ConsumerStrategies.Subscribe

val kafkaParams = Map[String, Object](
  "bootstrap.servers" -> "localhost:9092,anotherhost:9092",
  "key.deserializer" -> classOf[StringDeserializer],
  "value.deserializer" -> classOf[StringDeserializer],
  "group.id" -> "use_a_separate_group_id_for_each_stream",
  "auto.offset.reset" -> "latest",
  "enable.auto.commit" -> (false: java.lang.Boolean)
)

val topics = Array("topicA", "topicB")
val stream = KafkaUtils.createDirectStream[String, String](
  streamingContext,
  PreferConsistent,
  Subscribe[String, String](topics, kafkaParams)
)

stream.map(record => (record.key, record.value))
```

`KafkaUtils.creatDirectStream` 需要传入3个参数：

1. StreamingContext
2. LocationStrategy，默认采用`LocationStrategies.PreferConsistent` 
3. ConsumerStrategy

### 偏移量操作

上面说过，直连的方式需要手动维护偏移量。Kafka提供了一个提交offset的API，这个API把特定的kafka topic的offset进行存储。默认情况下，新的消费者会周期性的自动提交offset，这里把`enable.auto.commit`设置为false。但是，使用commitAsync api，用户可以在确保计算结果被成功保存后自己来提交offset。和使用checkpoint方式相比，Kafka是一个不用关心应用代码的变化可靠存储系统。

获取偏移量

``` scala
stream.foreachRDD { rdd =>
  val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
  rdd.foreachPartition { iter =>
    val o: OffsetRange = offsetRanges(TaskContext.get.partitionId)
    println(s"${o.topic} ${o.partition} ${o.fromOffset} ${o.untilOffset}")
  }
}
```

可以使用`commitAsync`来提交偏移量

``` scala
stream.foreachRDD { rdd =>
  val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges

  // some time later, after outputs have completed
  stream.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges)
}
```