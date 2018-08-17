---
title: Kafka Connect进行数据同步
date: 2018-07-31 11:21:39
categories: [Big Data, Kafka]
tag: [Kafka]
---

![](http://7xkfga.com1.z0.glb.clouddn.com/0d9dc4075f3e89d1ad093a097c6723e4.jpg)

Kafka Connect 是一个可扩展、可靠的在 Kafka 和其他系统之间流传输的数据工具。它可以通过 Connectors （连接器）简单、快速的将大集合数据导入和导出 Kafka，数据的导入导出可能有以下几种选择：

- flume
- kafka connector
- kafka 生产者/消费者API
- 商业ETL工具

Kafka Connect可以将完整的数据库注入到Kafka的Topic中，或者将服务器的系统监控指标注入到Kafka，然后像正常的Kafka流处理机制一样进行数据流处理。

使用Kafka客户端API，是需要在应用程序里面进行开发的，可以根据不同的需求进行数据的操作；而如果需要Kafka连接数据存储系统，则使用例如Connect这种可插拔式的连接器，就会很方便，因为不需要修改系统代码。

<!-- more -->

## 构建数据管道需要考虑的问题

ETL，现在流行的叫法叫做数据管道，构建一个好的数据管道可能要考虑一下几个方面：

1. 及时性、数据频率要求
2. 可靠性，避免单点故障，并能够自动从各种故障中快速恢复；
3. 高吞吐两盒动态吞吐量
4. 数据格式，需要支持各种不同的数据类型
5. 转换，ETL/ELT
6. 安全性
7. 故障处理能力
8. 耦合性和灵活性

## Kafka Connect的工作模式

- Standalone：在standalone模式中，所有的worker都在一个独立的进程中完成。

- Distributed：distributed模式具有高扩展性，以及提供自动容错机制。你可以使用一个group.ip来启动很多worker进程，在有效的worker进程中它们会自动的去协调执行connector和task，如果你新加了一个worker或者挂了一个worker，其他的worker会检测到然后在重新分配connector和task。

## 运行Kafka Connect

下面以Standalone模式举例，通过下面的命令运行Connect：

``` bash
bin/connect-standalone.sh config/connect-standalone.properties Connector1.properties [Connector2.properties ...]
```

第一个参数是worker的配置，这包括Kafka连接的参数设置，序列化格式，以及频繁地提交offset。

Connect进程有以下几个重要的配置参数：

- bootstrap.server
- group.id
- key.converter/val.converter

其余的参数是 Connector（连接器）配置文件

下面以Elasticsearch-sink作为例子，配置一个自己的connector

``` 
name=elasticsearch-sink
connector.class=io.confluent.connect.elasticsearch.ElasticsearchSinkConnector
tasks.max=1
topics=logs
topic.index.map=logs:logs_index
key.ignore=true
schema.ignore=true
connection.url=http://localhost:9200
type.name=log
```

- name： 连接器唯一的名称，不能重复。
- Connector.calss：连接器的Java类。
- tasks.max：连接器创建任务的最大数。
- Connector.class 配置支持多种格式：全名或连接器类的别名。比如连接器是 `org.apache.kafka.connect.file.FileStreamSinkConnector`，你可以指定全名，也可以使用 FileStreamSink或FileStreamSinkConnector
- topics：作为连接器的输入的 topic 列表。

### 加载Elasticsearch-Sink Connector

``` bash
confluent load elasticsearch-sink
```

## REST API管理Connector

Confluent提供了一个用于管理 Connector 的 REST API。默认情况下，此服务的端口是8083

- GET /Connectors：返回活跃的 Connector 列表
- POST /Connectors：创建一个新的 Connector；请求的主体是一个包含字符串name字段和对象 config 字段（Connector 的配置参数）的 JSON 对象。
- GET /Connectors/{name}：获取指定 Connector 的信息
- GET /Connectors/{name}/config：获取指定 Connector 的配置参数
- PUT /Connectors/{name}/config：更新指定 Connector 的配置参数
- GET /Connectors/{name}/status：获取 Connector 的当前状态，包括它是否正在运行，失败，暂停等。
- GET /Connectors/{name}/tasks：获取当前正在运行的 Connector 的任务列表。
- GET /Connectors/{name}/tasks/{taskid}/status：获取任务的当前状态，包括是否是运行中的，失败的，暂停的等，
- PUT /Connectors/{name}/pause：暂停连接器和它的任务，停止消息处理，直到 Connector 恢复。
- PUT /Connectors/{name}/resume：恢复暂停的 Connector（如果 Connector 没有暂停，则什么都不做）
- POST /Connectors/{name}/restart：重启 Connector（Connector 已故障）
- POST /Connectors/{name}/tasks/{taskId}/restart：重启单个任务 (通常这个任务已失败)
- DELETE /Connectors/{name}：删除 Connector, 停止所有的任务并删除其配置

## 测试使用Connector向Elasticsearch中发送数据

首先创建一个Topic：logs

``` bash
kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic logs
```

启动Kafka Connect

``` bash
bin/connect-standalone etc/kafka/connect-standalone.properties etc/kafka-connect-elasticsearch/elasticsearch-connect.properties &
```

向Topic发送一条json数据

``` bash
kafka-console-producer --broker-list localhost:9092 --topic logs

>{"name":"f1","type":"string"}
```

``` bash
kafka-console-consumer --bootstrap-server localhost:9092 --topic logs --from-beginning
```

![](http://7xkfga.com1.z0.glb.clouddn.com/2ae1bbfda905f00a6ed2348771f6da31.jpg)

可以看到，刚才的一条数据已经写入到Topic中了

同时，可以看到Elasticsearch中已经存在刚才Topic中的那条数据了

![](http://7xkfga.com1.z0.glb.clouddn.com/8f0b8517ced51e9b0379e6d63be5b0ab.jpg)

## MySQL -> Kafka -> Elasticsearch

![MySQL -> Kafka -> Elasticsearch](https://www.confluent.io/wp-content/uploads/elasticsearch_kafka-min-768x327.png)

### 准备

首先下载数据库的JDBC驱动， [MySQL](https://dev.mysql.com/downloads/connector/j/5.1.html)、[SQL Server](https://www.microsoft.com/en-us/download/details.aspx?id=56615)...

启动Confluent

![](http://7xkfga.com1.z0.glb.clouddn.com/98017f5944415d87fe4be68d8f672ac3.jpg)

### 在MySQL中创建测试数据

![](http://7xkfga.com1.z0.glb.clouddn.com/998f900ede9a3f68511c8745ad7ec215.jpg)

插入一条测试数据

``` sql
insert into sample_data (id, name) values(1,'claus');
```

### 配置Kafka JDBC Source

``` json
{
    "name": "jdbc_source_mysql_sample_01",
    "config": {
            "_comment": "The JDBC connector class. Don't change this if you want to use the JDBC Source.",
            "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",

            "_comment": "How to serialise the value of keys - here use the Confluent Avro serialiser. Note that the JDBC Source Connector always returns null for the key ",
            "key.converter": "io.confluent.connect.avro.AvroConverter",

            "_comment": "Since we're using Avro serialisation, we need to specify the Confluent schema registry at which the created schema is to be stored. NB Schema Registry and Avro serialiser are both part of Confluent Open Source.",
            "key.converter.schema.registry.url": "http://localhost:8081",

            "_comment": "As above, but for the value of the message. Note that these key/value serialisation settings can be set globally for Connect and thus omitted for individual connector configs to make them shorter and clearer",
            "value.converter": "io.confluent.connect.avro.AvroConverter",
            "value.converter.schema.registry.url": "http://localhost:8081",


            "_comment": " --- JDBC-specific configuration below here  --- ",
            "_comment": "JDBC connection URL. This will vary by RDBMS. Consult your manufacturer's handbook for more information",
            "connection.url": "jdbc:mysql://localhost:3306/kafka_demo?user=claus&password=LvRaikkonen_0306",

            "_comment": "Which table(s) to include",
            "table.whitelist": "sample_data",

            "_comment": "Pull all rows based on an timestamp column. You can also do bulk or incrementing column-based extracts. For more information, see http://docs.confluent.io/current/connect/connect-jdbc/docs/source_config_options.html#mode",
            "mode": "timestamp",

            "_comment": "Which column has the timestamp value to use?  ",
            "timestamp.column.name": "update_ts",

            "_comment": "If the column is not defined as NOT NULL, tell the connector to ignore this  ",
            "validate.non.null": "false",

            "_comment": "The Kafka topic will be made up of this prefix, plus the table name  ",
            "topic.prefix": "mysql-"
    }
}
```

### 加载Connector并将MySQL数据写入Topic中

加载刚才创建的Connector

``` bash
bin/confluent load jdbc_source_mysql_sample_01 -d etc/kafka-connect-jdbc/kafka-connect-mysql-jdbc-source.json
```

![](http://7xkfga.com1.z0.glb.clouddn.com/7f435dce09ab276c7ef26c767bafab3e.jpg)

可以看见，connector已经被加载并且正在运行

![](http://7xkfga.com1.z0.glb.clouddn.com/fecbc7df87463fb381fb0b0dd55e7a3d.jpg)

运行Avro Consumer查看Topic中的数据

``` bash
kafka-avro-console-consumer --bootstrap-server localhost:9092 --property schema.registry.url=http://localhost:8081 --property print.key=true --from-beginning --topic mysql-sample_data
```

![](http://7xkfga.com1.z0.glb.clouddn.com/dd2deaccea181e98ff80d0852260a7c0.jpg)

Kafka Connect使用Topic `connect-offsets`来跟踪数据的偏移量，这里的offset是通过sample_data表的`update_ts`字段来跟踪并提交的，所以connector即使关闭，下次也可以继续追踪新增数据。

### 配置ES-sink并将Topic数据导入到ES中

``` json
{
  "name": "es-sink-mysql-sampleData-01",
  "config": {
    "_comment": "-- standard converter stuff -- this can actually go in the worker config globally --",
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "key.converter": "io.confluent.connect.avro.AvroConverter",
    "key.converter.schema.registry.url": "http://localhost:8081",
    "value.converter.schema.registry.url": "http://localhost:8081",


    "_comment": "--- Elasticsearch-specific config ---",
    "_comment": "Elasticsearch server address",
    "connection.url": "http://localhost:9200",

    "_comment": "Elasticsearch mapping name. Gets created automatically if doesn't exist  ",
    "type.name": "type.name=kafka-connect",

    "_comment": "Which topic to stream data from into Elasticsearch",
    "topics": "mysql-sample_data",

    "_comment": "If the Kafka message doesn't have a key (as is the case with JDBC source)  you need to specify key.ignore=true. If you don't, you'll get an error from the Connect task: 'ConnectException: Key is used as document id and can not be null.",
    "key.ignore": "true"
  }
}
```

可以看到ES中已经创建名为mysql-sample_data的Index，并且Topic中的4条数据已经导入到ES中了。

![](http://7xkfga.com1.z0.glb.clouddn.com/8f606c2881399514f392e62907f3a2f7.jpg)


## 开发Connector

TBD