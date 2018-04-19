---
title: Kafka入门
date: 2018-04-18 19:25:38
categories: [Kafka]
tag: [Kafka]
---

Kafka是一个分布式消息发布订阅系统，Kafka可以用于以下场景：
- 消息系统， 例如ActiveMQ 和 RabbitMQ.
- 站点的用户活动追踪。 用来记录用户的页面浏览，搜索，点击等。
- 操作审计。 用户/管理员的网站操作的监控。
- 日志聚合。收集数据，集中处理。
- 流处理等

![流处理平台](http://7xkfga.com1.z0.glb.clouddn.com/f82cbd2ddbb1b9d8967aef1f9400d95e.jpg)

<!-- more -->

## 基本概念

- Producer：消息生产者，就是向kafka broker发消息的客户端。
- Consumer：消息消费者，是消息的使用方，负责消费Kafka服务器上的消息。
- Topic：主题，由用户定义并配置在Kafka服务器，用于建立Producer和Consumer之间的订阅关系。生产者发送消息到指定的Topic下，消息者从这个Topic下消费消息。
- Partition：消息分区，一个topic可以分为多个 partition，每个partition是一个有序的队列。partition中的每条消息都会被分配一个有序的id（offset）。

    _note: 一个topic的一个partition只能被一个consumer group中的一个consumer消费，多个consumer消费同一个partition中的数据是不允许的，但是一个consumer可以消费多个partition中的数据_


![partition](http://7xkfga.com1.z0.glb.clouddn.com/d01cc984be0f688c7b965359c18bb69b.jpg)

- Broker：一台kafka服务器就是一个broker。一个集群由多个broker组成。一个broker可以容纳多个topic。
- Consumer Group：消费者分组，用于归组同类消费者。每个consumer属于一个特定的consumer group，多个消费者可以共同消费一个Topic下的消息，每个消费者消费其中的部分消息，这些消费者就组成了一个分组，拥有同一个分组名称，通常也被称为消费者集群。

## 快速实践Kafka

下载kafka和zookeeper的安装包，解压，配置环境变量，启动zookeeper服务，启动kafka守护进程

### 新建一个topic

使用kafka自带的命令行工具kafka-topic.sh创建一个新的topic

``` shell
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
```

### 发送一个消息到topic

``` shell
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test 
This is a message
This is another message
```

### 读取刚创建的消息

``` shell
bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning
```

可以看到刚才创建的两条消息被输出来了。

## 使用python的Kafka客户端实现生产者消费者

实现上面命令行的生产者消费者的例子：

### 生产者

``` python
from kafka import KafkaProducer
import time

producer = KafkaProducer(bootstrap_servers='localhost:9092')  
  
i = 0
while True:
    ts = int(time.time() * 1000)
    producer.send(topic="myfirsttopic", value=bytes(str(i).encode('utf-8')), timestamp_ms=ts)
    producer.flush()
    print(i)
    i += 1
    time.sleep(1)
```

### 消费者

``` python
from kafka import KafkaConsumer

KAFKA_HOST = 'localhost'
KAFKA_PORT = 9092
KAFKA_TOPIC = 'myfirsttopic'

consumer = KafkaConsumer(KAFKA_TOPIC, bootstrap_servers='localhost:9092'
)

try:
    for message in consumer:
        print(message)
except KeyboardInterrupt:
    print("Catch keyboard interrupt")
```

可以看到，生产者产生的消息被消费者消费了

![producer_consumer_python1](http://7xkfga.com1.z0.glb.clouddn.com/6846419c10b1905e609a30845996320d.jpg)

### 追踪偏移量offset

kafka允许consumer将当前消费的消息的offset提交到kafka中，这样如果consumer因异常退出后，下次启动仍然可以从上次记录的offset开始向后继续消费消息。

修改一下刚才的consumer代码，把enable_auto_commit设为false，让应用程序决定何时提交偏移

``` python
from kafka import KafkaConsumer, TopicPartition, OffsetAndMetadata

KAFKA_HOST = 'localhost'
KAFKA_PORT = 9092
kafka_topic = TopicPartition("myfirsttopic", 0)

consumer = KafkaConsumer(
    bootstrap_servers=['localhost:9092'],
    group_id="testgroup",
    auto_offset_reset="earliest",
    enable_auto_commit=False
)

consumer.assign([tp])
print("Start offset is: " + str(consumer.position(tp)))

try:
    for message in consumer:
        print(message)
        consumer.seek(tp, message.offset+1)
        consumer.commit()
except KeyboardInterrupt:
    print("Catch keyboard interrupt")
```

在offset为249时候终止消费者，然后再次启动消费者，可以看出是从上次停止的地方(250)继续消费：

![offset](http://7xkfga.com1.z0.glb.clouddn.com/5e88d29ceebffad0e1f3a4447e3e5d20.jpg)

默认情况下auto.commit.enable等于true，这也就意味着consumer会定期的commit offset

## Kafka消息发送分区选择

Kafka发送消息的流程图如下：

![sendMsg](http://7xkfga.com1.z0.glb.clouddn.com/e5100a41ab185ec8ac110f7d12d67648.jpg)

那么，发送消息的时候，是如何选择分区的呢？KafkaProducer对象通过send方法，将记录发给kafka

``` java
producer.send(new ProducerRecord<String, String>(topicName, Integer.toString(i), Integer.toString(i)));
```

在KafkaProducer中，是通过内部的私有方法doSend来发送消息

``` java
int partition = this.partition(record, serializedKey, serializedValue, cluster);
```

看一下partition方法的实现：

``` java
private int partition(ProducerRecord<K, V> record, byte[] serializedKey, byte[] serializedValue, Cluster cluster) {
        Integer partition = record.partition();
        return partition != null ? partition.intValue() : this.partitioner.partition(record.topic(), record.key(), serializedKey, record.value(), serializedValue, cluster);
}
```

如果record指定了分区则指定的分区会被使用，如果没有则使用partitioner分区器来选择分区

partitioner的实现：

``` java
public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
    List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
    int numPartitions = partitions.size();
    if (keyBytes == null) {
        int nextValue = this.nextValue(topic);
        List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
        if (availablePartitions.size() > 0) {
            int part = Utils.toPositive(nextValue) % availablePartitions.size();
            return ((PartitionInfo)availablePartitions.get(part)).partition();
        } else {
            return Utils.toPositive(nextValue) % numPartitions;
        }
    } else {
        return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
    }
}

private int nextValue(String topic) {
    AtomicInteger counter = (AtomicInteger)this.topicCounterMap.get(topic);
    if (null == counter) {
        counter = new AtomicInteger(ThreadLocalRandom.current().nextInt());
        AtomicInteger currentCounter = (AtomicInteger)this.topicCounterMap.putIfAbsent(topic, counter);
        if (currentCounter != null) {
                counter = currentCounter;
        }
    }

    return counter.getAndIncrement();
}
```

如果key为null，则先根据topic名获取上次计算分区时使用的一个整数并加一。然后判断topic的可用分区数是否大于0，如果大于0则使用获取的nextValue的值和可用分区数进行取模操作。 如果topic的可用分区数小于等于0，则用获取的nextValue的值和总分区数进行取模操作。

如果消息的key不为null，就根据hash算法murmur2就算出key的hash值，然后和分区数进行取模运算。

## Kafka Connect

Kafka Connect 是Kafka 的一部分，它为在Kafka 和外部数据存储系统之间移动数据提供了一种可靠且可伸缩的方式。

![KafkaConnect](http://7xkfga.com1.z0.glb.clouddn.com/732b33d9089033702a584c8b2754f886.jpg)

Kafka Connnect有两个核心概念：Source和Sink。 Source负责导入数据到Kafka，Sink负责从Kafka导出数据，它们都被称为Connector。

### Kafka Connect实例

ToDo

## 参考

- [Apache Kafka Doc](http://kafka.apache.org/)
- [Kafka Connect](https://www.confluent.io/)
- [Building a Real-Time Streaming ETL Pipeline in 20 Minutes](https://www.confluent.io/blog/building-real-time-streaming-etl-pipeline-20-minutes/)