---
title: Kafka消费者基础使用
date: 2018-07-25 15:08:00
categories: [Big Data, Kafka]
tag: [Kafka]
---

## Kafka消费者相关概念

### 消费者与消费者组

消费者读取过程

创建消费者对象 -> 订阅主题 -> 读取消息 -> 验证消息 -> 保存消息

Kafka消费者从属于消费者群组。一个群组里的消费者订阅的是同一个主题，每个消费者接收主题一部分分区的消息。消费者加入群组或者由于关闭或崩溃导致消费者离开群组时候，分区的所有权需要从一个消费者转移到另一个消费者，这样的行为被称为**再均衡**

群组协调器

对于每一个消费者组，都会从所有的broker中选取一个作为消费者组协调器(group coordinator)，负责维护和管理这个消费者组的状态，它的主要工作是当一个consumer加入、一个consumer离开（挂掉或者手动停止等）或者topic的partition改变时重新进行partition分配(再均衡)

### Offset偏移量管理

提交：更新分区当前读取位置的操作叫做提交

偏移量：消息在分区中的位置，决定了消费者下次开始读取消息的位置

如果提交偏移量小于当前处理的消息位置，则两个之间的消息会被再次处理；

![LastCommitedOffsetSmallerThanCurrent](http://7xkfga.com1.z0.glb.clouddn.com/b405c7e558d48b48a2a5150fd9cf00e5.jpg)

如果提交偏移量大于当前处理的消息位置，则两个之间的消息会丢失。

![LastCommitedOffsetLargetThanCurrent](http://7xkfga.com1.z0.glb.clouddn.com/7578af6b37abadb9f79c48424cce9322.jpg)


<!-- more -->

- Last Committed Offset：这是 group 最新一次 commit 的 offset，表示这个 group 已经把 Last Committed Offset 之前的数据都消费成功了；
- Current Position：group 当前消费数据的 offset，也就是说，Last Committed Offset 到 Current Position 之间的数据已经拉取成功，可能正在处理，但是还未 commit；
- Log End Offset：Producer 写入到 Kafka 中的最新一条数据的 offset；
- High Watermark：已经成功备份到其他 replicas 中的最新一条数据的 offset，也就是说 Log End Offset 与 High Watermark 之间的数据已经写入到该 partition 的 leader 中，但是还未成功备份到其他的 replicas 中，这部分数据被认为是不安全的，是不允许 Consumer 消费的

`__consumer_offsets` 是 Kafka 内部使用的一个 topic，专门用来存储 group 消费的情况，默认情况下有50个 partition，每个 partition 三副本，而具体 group 的消费情况要存储到哪一个 partition 上，是根据 abs(GroupId.hashCode()) % NumPartitions 来计算（其中，NumPartitions 是__consumer_offsets 的 partition 数，默认是50个）的

## 消费者的初始化和配置

下面是一个消费者的最基本配置

``` java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092"); // 通过其中的一台broker来找到group的coordinator，并不需要列出所有的broker
props.put("group.id", "consumer-group-simple");
props.put("key.deserializer", StringDeserializer.class.getName());
props.put("value.deserializer", StringDeserializer.class.getName());
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props); // consumer实例
```

## 订阅Topic

创建好消费者之后，接下来调用subscribe() 方法来订阅Topic

``` java
consumer.subscribe(Collections.singleton("topicList"));
```

## 轮询请求数据

轮询是消费者API的核心，在循环中调用`poll`方法，轮询就会处理所有的细节，包括群组协调、分区再均衡、发送心跳和获取数据。

``` java
try {
    while (running) {
        ConsumerRecords<String, String> records = consumer.poll(1000);
        for (ConsumerRecord<String, String> record : records)
            System.out.println(record.offset() + ": " + record.value());
    }
} finally {
    consumer.close();
}
```

`poll()`方法返回一个记录列表。每条记录都包含了记录所属主题的信息、记录所在分区的信息、记录在分区里的偏移量，以及记录的键值对。

## 偏移量管理

当一个消费者群组刚开始被创建的时候，最初的offset是通过`auto.offset.reset`配置项来进行设置的。一旦消费者开始处理数据，它根据应用的需要来定期地对offset进行commit。在每一次的再均衡之后，群组会将这个offset设置为`Last Committed Offset`

消费者API提供了很多种方式来提交偏移量

### 自动提交

如果`enable.auto.commit`被设为true，那么每过5s(这个时长可以通过`auto.commit.interval.ms`来进行配置)，消费者会自动把从poll() 方法接收到的最大偏移量提交上去。

``` java
props.put("enable.auto.commit", "true"); // 自动commit
props.put("auto.commit.interval.ms", "1000"); // 自动commit的间隔
```

在使用自动提交时，每次调用轮询方法都会把上一次调用返回的偏移量提交上去，它并不知道具体哪些消息已经被处理了。如果使用默认的自动commit机制，系统是保证at least once消息处理，因为offset是在这些messages被应用处理后才进行commit的

### 手动提交

把`auto.commit.offset`设为false，让应用程序决定何时提交偏移量。使用`commitSync()`同步提交偏移量，API也提供了`commitAsync()`方法异步提交偏移量。这个API 会提交由poll() 方法返回的最新偏移量，提交成
功后马上返回，如果提交失败就抛出异常

``` java
try {
    while (running) {
        ConsumerRecords<String, String> records = consumer.poll(1000);
        for (ConsumerRecord<String, String> record : records)
            System.out.println(record.offset() + ": " + record.value());
        try {
            consumer.commitSync();
        } catch (CommitFailedException e) {
            // application specific failure handling
        }
    }
} finally {
    consumer.close();
}
```

## 从特定偏移量处开始读取

Kafka提供了用于查找特定偏移量的API，通过`seek()`来指定分区位移开始消费

``` java
public class SaveOffsetsOnRebalance implements ConsumerRebalanceListener {
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        //在消费者负责的分区被回收前提交数据库事务，保存消费的记录和位移
        commitDBTransaction();
    }
    
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        //在开始消费前，从数据库中获取分区的位移，并使用seek()来指定开始消费的位移
        for(TopicPartition partition: partitions)
            consumer.seek(partition, getOffsetFromDB(partition));
    } 
}

    consumer.subscribe(topics, new SaveOffsetOnRebalance(consumer));
    //在subscribe()之后poll一次，并从数据库中获取分区的位移，使用seek()来指定开始消费的位移
    consumer.poll(0);
    for (TopicPartition partition: consumer.assignment())
        consumer.seek(partition, getOffsetFromDB(partition));

    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(100);
        for (ConsumerRecord<String, String> record : records)
        {
            processRecord(record);
            //保存记录结果
            storeRecordInDB(record);
            //保存位移
            storeOffsetInDB(record.topic(), record.partition(), record.offset());
        }
        //提交数据库事务，保存消费的记录以及位移
        commitDBTransaction();
    }
```