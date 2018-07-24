---
title: (转)Kafka生产者发送消息流程
date: 2018-07-24 13:07:48
categories: [Big Data, Kafka]
tag: [Kafka]
---

参考[Kafka 源码解析之 Producer 发送模型（一）](http://matt33.com/2017/06/25/kafka-producer-send-module/)

## 一个简单的生产者

下面的代码是一个简单的生产者向Kafka中发送消息的例子：

``` java
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.util.Properties;

public class SimpleProducer {

    public static void main(String[] args){
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

        Producer<String, String> producer = new KafkaProducer<>(props);
        for (int i = 0; i < 10; i++){
            String msg = "This is message " + i;
            ProducerRecord<String, String> pr = new ProducerRecord<>("testTopic", msg);
            producer.send(pr);
        }
        producer.close();
    }
}
```

Kafka提供了生产者API，使用时候需要实例化KafkaProducer，然后调用send方法发送数据。

## 生产者数据发送流程

下面的流程图展示了消息的发送流程

![producer send message procedure](http://7xkfga.com1.z0.glb.clouddn.com/ffb9e67b8d65d57b7cb829db352c0af7.jpg)

首先创建一个`ProducerRecord`对象，包含了目标主题和要发送的内容。然后数据被发送给`序列化器`，将键和值对象序列化成字节数据。然后数据发送给`分区器`，如果ProducerRecord里面指定了分区，分区器不做任何操作，否则根据ProducerRecord对象的键来选择一个分区。然后上面的记录被追加到一个`记录批次`里面，有一个独立的线程将记录批次发送到相应的broker上。

<!-- more -->

实例化了生产者对象之后，调用send方法发送数据

``` java
// 异步向Topic发送数据
@Override
public Future<RecordMetadata> send(ProducerRecord<K, V> record) {
    return send(record, null);
}

// 异步向Topic发送数据，发送确认后调用回调函数
@Override
public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
    // intercept the record, which can be potentially modified; this method does not throw exceptions
    ProducerRecord<K, V> interceptedRecord = this.interceptors.onSend(record);
    return doSend(interceptedRecord, callback);
}
```

### doSend的具体实现

``` java
/**
 * Implementation of asynchronously send a record to a topic.
 */
private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
    TopicPartition tp = null;
    try {
        // 1.确认数据要发送到的topic的 metadata 是可用的
        ClusterAndWaitTime clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), maxBlockTimeMs);
        long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
        Cluster cluster = clusterAndWaitTime.cluster;
        // 2.序列化key和value
        byte[] serializedKey;
        try {
            serializedKey = keySerializer.serialize(record.topic(), record.headers(), record.key());
        } catch (ClassCastException cce) {
            throw new SerializationException("Can't convert key of class " + record.key().getClass().getName() +
                " to class " + producerConfig.getClass(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG).getName() +
                " specified in key.serializer", cce);
        }
        byte[] serializedValue;
        try {
            serializedValue = valueSerializer.serialize(record.topic(), record.headers(), record.value());
        } catch (ClassCastException cce) {
            throw new SerializationException("Can't convert value of class " + record.value().getClass().getName() +
                " to class " + producerConfig.getClass(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG).getName() +
                " specified in value.serializer", cce);
        }
        // 3.获取消息的partition值
        int partition = partition(record, serializedKey, serializedValue, cluster);
        tp = new TopicPartition(record.topic(), partition);

        setReadOnly(record.headers());
        Header[] headers = record.headers().toArray();

        int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(),
                    compressionType, serializedKey, serializedValue, headers);
        ensureValidRecordSize(serializedSize);
        long timestamp = record.timestamp() == null ? time.milliseconds() : record.timestamp();
        log.trace("Sending record {} with callback {} to topic {} partition {}", record, callback, record.topic(), partition);
        // producer callback will make sure to call both 'callback' and interceptor callback
        Callback interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);

        if (transactionManager != null && transactionManager.isTransactional())
                transactionManager.maybeAddPartitionToTransaction(tp);
        
        // 4.向累加器中追加数据
        RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey, serializedValue, headers, interceptCallback, remainingWaitMs);
        // 5.如果batch满了，唤醒sender线程发送数据 
        if (result.batchIsFull || result.newBatchCreated) {
            log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
            this.sender.wakeup();
        }
        return result.future;
        // handling exceptions and record the errors;
        // for API exceptions return them in the future,
        // for other exceptions throw directly
    } catch (ApiException e) {
        log.debug("Exception occurred during message send:", e);
        if (callback != null)
            callback.onCompletion(null, e);
        this.errors.record();
        this.interceptors.onSendError(record, tp, e);
        return new FutureFailure(e);
    } catch (InterruptedException e) {
        this.errors.record();
        this.interceptors.onSendError(record, tp, e);
        throw new InterruptException(e);
    } catch (BufferExhaustedException e) {
        this.errors.record();
        this.metrics.sensor("buffer-exhausted-records").record();
        this.interceptors.onSendError(record, tp, e);
        throw e;
    } catch (KafkaException e) {
        this.errors.record();
        this.interceptors.onSendError(record, tp, e);
        throw e;
    } catch (Exception e) {
        // we notify interceptor about all exceptions, since onSend is called before anything else in this method
        this.interceptors.onSendError(record, tp, e);
        throw e;
    }
}
```

上面`doSend`方法中，发送数据总共分5步：

1. 确认数据要发送到的 topic 的 metadata 是可用的
2. 序列化key和value
3. 获取消息的partition值
4. 向累加器中追加数据，数据先进行缓存
5. 如果batch满了，唤醒sender线程发送数据

### 序列化

生产者端对数据的key和value进行序列化操作，消费者端再进行相应的反序列化操作，下面是Kafka提供的序列化器和反序列化器

![Kafka.serializers](http://7xkfga.com1.z0.glb.clouddn.com/08d251f219dd8a86c817d387540b36c6.jpg)

对于自带的序列化器不能满足需求的情况，可以使用例如Avro、Thrift或者Protobuf等序列化框架或者使用自定义的序列化器。

下面使用[Avro](https://avro.apache.org/)序列化记录

首先使用Avro命令生成自定义对象，.avsc文件通过JSON来描述数据的schema

``` avsc
{
  "type": "record",
  "namespace": "com.example",
  "name": "Customer",
  "version": "1",
  "fields": [
    { "name": "first_name", "type": "string", "doc": "First Name of Customer" },
    { "name": "last_name", "type": "string", "doc": "Last Name of Customer" },
    { "name": "age", "type": "int", "doc": "Age at the time of registration" },
    { "name": "height", "type": "float", "doc": "Height at the time of registration in cm" },
    { "name": "weight", "type": "float", "doc": "Weight at the time of registration in kg" },
    { "name": "automated_email", "type": "boolean", "default": true, "doc": "Field indicating if the user is enrolled in marketing emails" }
  ]
}
```

生成自定义的数据对象后，将schema保存在注册表中，这样消费者在读取数据的时候就知道用什么schema来反序列化记录。

![schemaRegistry](http://7xkfga.com1.z0.glb.clouddn.com/b0ae8380071924c101579698e00ac4d7.jpg)

下面是使用avro的KafkaAvroSerializer来序列化对象，需要制定schema注册表的位置`http://127.0.0.1:8081`

``` java
import io.confluent.kafka.serializers.KafkaAvroSerializer;
import org.apache.kafka.clients.producer.Callback;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;

public class KafkaAvroProducer {


    public static void main(String[] args){

        Properties properties = new Properties();
        properties.setProperty("bootstrap.servers", "localhost:9092");
        properties.setProperty("acks", "1");
        properties.setProperty("retries", "10");
        properties.setProperty("key.serializer", StringSerializer.class.getName());
        properties.setProperty("value.serializer", KafkaAvroSerializer.class.getName());
        properties.setProperty("schema.registry.url", "http://127.0.0.1:8081");

        String topic = "customer-avro";

        KafkaProducer<String, Customer> kafkaProducer = new KafkaProducer<String, Customer>(properties);

        Customer customer = Customer.newBuilder()
                .setFirstName("Mike")
                .setLastName("Tom")
                .setAge(30)
                .setHeight(171.0f)
                .setWeight(120.5f)
                .setAutomatedEmail(false)
                .build();

        ProducerRecord<String, Customer> producerRecord = new ProducerRecord<String, Customer>(
                topic, customer
        );

        kafkaProducer.send(producerRecord, new Callback() {
            @Override
            public void onCompletion(RecordMetadata recordMetadata, Exception e) {
                if (e == null){
                    System.out.println("Success");
                    System.out.println(recordMetadata.toString());
                }else {
                    e.printStackTrace();
                }
            }
        });

        kafkaProducer.flush();
        kafkaProducer.close();
    }
}
```

更具体的Avro序列化，查看Avro的相关文档。

### 获取分区

看一下partition方法的实现：

``` java
private int partition(ProducerRecord<K, V> record, byte[] serializedKey, byte[] serializedValue, Cluster cluster) {
    Integer partition = record.partition();
    return partition != null ? partition.intValue() : this.partitioner.partition(record.topic(), record.key(), serializedKey, record.value(), serializedValue, cluster);
}
```

如果record指定了分区则指定的分区会被使用，如果没有则使用partitioner分区器来选择分区

``` java
public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
    List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
    int numPartitions = partitions.size();
    // key为空
    if (keyBytes == null) {
        // 根据topic名获取上次计算分区时使用的一个整数并加一
        int nextValue = this.nextValue(topic);
        List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
        if (availablePartitions.size() > 0) {
            // 如果大于0则使用获取的nextValue的值和可用分区数进行取模操作
            int part = Utils.toPositive(nextValue) % availablePartitions.size();
            return ((PartitionInfo)availablePartitions.get(part)).partition();
        } else {
            return Utils.toPositive(nextValue) % numPartitions;
        }
    } else {
        // 如果消息的key不为null，就根据hash算法murmur2就算出key的hash值，然后和分区数进行取模运算。
        return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
    }
}

private int nextValue(String topic) {
    AtomicInteger counter = (AtomicInteger)this.topicCounterMap.get(topic);
    if (null == counter) {// 第一次调用时，随机产生
        counter = new AtomicInteger(ThreadLocalRandom.current().nextInt());
        AtomicInteger currentCounter = (AtomicInteger)this.topicCounterMap.putIfAbsent(topic, counter);
        if (currentCounter != null) {
                counter = currentCounter;
        }
    }

    return counter.getAndIncrement();
}
```

### 向累加器追加数据

生产者向累加器中追加数据，当batch满了，唤醒sender线程发送数据

Producer 通过`RecordAccumulator`实例追加数据，每个 TopicPartition 都会对应一个 Deque<RecordBatch>，当添加数据时，会向其 topic-partition 对应的这个 queue 最新创建的一个 RecordBatch 中添加 record，而发送数据时，则会先从 queue 中最老的那个 RecordBatch 开始发送。

``` java
public RecordAppendResult append(TopicPartition tp,
                                    long timestamp,
                                    byte[] key,
                                    byte[] value,
                                    Header[] headers,
                                    Callback callback,
                                    long maxTimeToBlock) throws InterruptedException {
    // We keep track of the number of appending thread to make sure we do not miss batches in
    // abortIncompleteBatches().
    appendsInProgress.incrementAndGet();
    ByteBuffer buffer = null;
    if (headers == null) headers = Record.EMPTY_HEADERS;
    try {
        // 获取该 topic-partition 对应的 queue
        Deque<ProducerBatch> dq = getOrCreateDeque(tp);// 每个 topicPartition 对应一个 queue
        synchronized (dq) {
            if (closed)
                throw new IllegalStateException("Cannot send after the producer is closed.");
            // 向 queue 中追加数据, 先获取 queue 中最新加入的那个 RecordBatch,
            // 如果不存在或者存在但剩余空余不足以添加本条 record 则返回 null
            RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq);
            if (appendResult != null)
                return appendResult;
        }

        // we don't have an in-progress record batch try to allocate a new batch
        byte maxUsableMagic = apiVersions.maxUsableProduceMagic();
        // 为 topic-partition 创建一个新的 RecordBatch 缓冲区, 需要初始化相应的 RecordBatch，要为其分配的大小是: max（batch.size, 加上头文件的本条消息的大小）
        int size = Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(maxUsableMagic, compression, key, value, headers));
        log.trace("Allocating a new {} byte message buffer for topic {} partition {}", size, tp.topic(), tp.partition());
        buffer = free.allocate(size, maxTimeToBlock);
        synchronized (dq) {
            // Need to check if producer is closed again after grabbing the dequeue lock.
            if (closed)
                throw new IllegalStateException("Cannot send after the producer is closed.");

            RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq);
            if (appendResult != null) {
                // Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...
            return appendResult;
            }

            // 给 topic-partition 创建一个 RecordBatch
            MemoryRecordsBuilder recordsBuilder = recordsBuilder(buffer, maxUsableMagic);
            ProducerBatch batch = new ProducerBatch(tp, recordsBuilder, time.milliseconds());
            // 向新的 RecordBatch 中追加数据
            FutureRecordMetadata future = Utils.notNull(batch.tryAppend(timestamp, key, value, headers, callback, time.milliseconds()));

            dq.addLast(batch);
            incomplete.add(batch);

            // Don't deallocate this buffer in the finally block as it's being used in the record batch
            buffer = null;

            // 如果 dp.size()>1 就证明这个 queue 有一个 batch 是可以发送了
            return new RecordAppendResult(future, dq.size() > 1 || batch.isFull(), true);
        }
    } finally {
        if (buffer != null)
            free.deallocate(buffer);
        appendsInProgress.decrementAndGet();
    }
}
```

1. 获取该 topic-partition 对应的 queue，没有的话会创建一个空的 queue；
2. 向 queue 中追加数据，先获取 queue 中最新加入的那个 RecordBatch，如果不存在或者存在但剩余空余不足以添加本条 record 则返回 null，成功写入的话直接返回结果，写入成功；
3. 如果不存在现有的RecordBatch，创建一个新的 RecordBatch，初始化内存大小根据 `max(batch.size, Records.LOG_OVERHEAD + Record.recordSize(key, value))` 来确定（防止单条 record 过大的情况）；
4. 向新建的 RecordBatch 写入 record，并将 RecordBatch 添加到 queue 中，返回结果，写入成功。

### 发送RecordBatch到Broker

当 record 写入成功后，如果发现 RecordBatch 已满足发送的条件（通常是 queue 中有多个 batch，那么最先添加的那些 batch 肯定是可以发送了），那么就会唤醒 sender 线程，发送 RecordBatch

``` java
private void sendProduceRequests(Map<Integer, List<ProducerBatch>> collated, long now) {
    for (Map.Entry<Integer, List<ProducerBatch>> entry : collated.entrySet())
        sendProduceRequest(now, entry.getKey(), acks, requestTimeout, entry.getValue());
}

private void sendProduceRequest(long now, int destination, short acks, int timeout, List<ProducerBatch> batches) {
    if (batches.isEmpty())
        return;

    Map<TopicPartition, MemoryRecords> produceRecordsByPartition = new HashMap<>(batches.size());
    final Map<TopicPartition, ProducerBatch> recordsByPartition = new HashMap<>(batches.size());

    // find the minimum magic version used when creating the record sets
    byte minUsedMagic = apiVersions.maxUsableProduceMagic();
    for (ProducerBatch batch : batches) {
        if (batch.magic() < minUsedMagic)
            minUsedMagic = batch.magic();
    }

    for (ProducerBatch batch : batches) {
        TopicPartition tp = batch.topicPartition;
        MemoryRecords records = batch.records();

        if (!records.hasMatchingMagic(minUsedMagic))
            records = batch.records().downConvert(minUsedMagic, 0, time).records();
        produceRecordsByPartition.put(tp, records);
        recordsByPartition.put(tp, batch);
    }

    ProduceRequest.Builder requestBuilder = ProduceRequest.Builder.forMagic(minUsedMagic, acks, timeout,
            produceRecordsByPartition, transactionalId);
    RequestCompletionHandler callback = new RequestCompletionHandler() {
        public void onComplete(ClientResponse response) {
            handleProduceResponse(response, recordsByPartition, time.milliseconds());
        }
    };

    String nodeId = Integer.toString(destination);
    ClientRequest clientRequest = client.newClientRequest(nodeId, requestBuilder, now, acks != 0, callback);
    client.send(clientRequest, now);
    log.trace("Sent produce request to {}: {}", nodeId, requestBuilder);
}
```