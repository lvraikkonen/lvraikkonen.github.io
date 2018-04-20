---
title: Flume1.8安装和入门实例
date: 2018-04-20 15:38:01
categories: [Big Data, Flume]
tag: [Flume]
---

## Flume简介

Flume是一个分布式的，可靠的，可用的，可以非常有效率的对大数据的日志数据进行收集、聚集、转移。

![Flume](http://7xkfga.com1.z0.glb.clouddn.com/cdae13e75ffa7669304e3be06d16923d.jpg)

每一个flume进程都有一个agent层，agent层包含有source、channel、sink：

- source：采集日志数据，并发送给channel
- channel：管道，用于连接source和sink，它是一个缓冲区，从source传来的数据会以event的形式在channel中排成队列，然后被sink取走。
- sink：获取channel中的数据，并存储到目标源中，目标源可以是HDFS和Hbase

<!-- more -->

## 安装与配置

下载 [Flume 1.8.0](http://archive.apache.org/dist/flume/1.8.0/apache-flume-1.8.0-bin.tar.gz)

解压缩，在`flume-env.sh`配置java环境变量，复制一份默认的`flume-conf.properties`

查看flume的版本

``` shell
$ bin/flume-ng version
Flume 1.8.0
Source code repository: https://git-wip-us.apache.org/repos/asf/flume.git
Revision: 99f591994468633fc6f8701c5fc53e0214b6da4f
Compiled by denes on Fri Sep 15 14:58:00 CEST 2017
From source with checksum fbb44c8c8fb63a49be0a59e27316833d
```

通过设置agent的配置文件，可以进行不同类型的数据收集，配置文件格式：

``` shell
# list the sources, sinks and channels for the agent
<Agent>.sources = <Source>
<Agent>.sinks = <Sink>
<Agent>.channels = <Channel1> <Channel2>

# set channel for source
<Agent>.sources.<Source>.channels = <Channel1> <Channel2> ...

# set channel for sink
<Agent>.sinks.<Sink>.channel = <Channel1>
```

OK，可以运行实例了

## 发送一个文件给Flume

### 新建配置文件 `avro.conf`

``` conf
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type= avro
a1.sources.r1.channels = c1
a1.sources.r1.bind = 0.0.0.0
a1.sources.r1.port = 4141

# Describe the sink
a1.sinks.k1.type= logger

# Use a channel which buffers events in memory
a1.channels.c1.type= memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

### 启动Flume代理

```
bin/flume-ng agent -c /Users/lvshuo/bigdata/flume/conf/ -f /Users/lvshuo/bigdata/flume/conf/avro.conf -n a1 -Dflume.root.logger=INFO,console
```
最后控制台上出现 `Avro source r1 started.` 表示agent a1启动成功。

### 使用avro-client发送文件

首先创建一个log文件

``` shell
echo "Hello World" > test.log
```

发送这个文件到上面配置文件设置的localhost:4141

``` shell
bin/flume-ng avro-client -c /Users/lvshuo/bigdata/flume/conf/ -H localhost -p 4141 -F /Users/lvshuo/test.log
```

### Flume控制台接受数据

发送文件之后，可以在控制台上看到刚才创建文件的内容：

![result](http://7xkfga.com1.z0.glb.clouddn.com/99507fa0307f65d323ace6eb5f510986.jpg)

## Spool监测配置目录中的新文件

Spool监测配置的目录下新增的文件，并将文件中的数据读取出来。

需要注意两点：  
- 1) 拷贝到spool目录下的文件不可以再打开编辑。  
- 2) spool目录下不可包含相应的子目录

### 创建agent的配置文件 `spool.conf`

新建一个文件夹，存放被监测的log文件

``` shell
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = spooldir
a1.sources.r1.channels = c1
a1.sources.r1.spoolDir =/Users/lvshuo/bigdata/logs
a1.sources.r1.fileHeader = true

# Describe the sink
a1.sinks.k1.type = logger

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

### 启动代理 agent a1

``` shell
bin/flume-ng agent -c /Users/lvshuo/bigdata/flume/conf/ -f /Users/lvshuo/bigdata/flume/conf/spool.conf -n a1 -Dflume.root.logger=INFO,console
```

### 模拟产生log文件

追加文件到被监测的文件夹中(/Users/lvshuo/bigdata/logs)

``` shell
cp /Users/lvshuo/test.log /Users/lvshuo/bigdata/logs
```

### Flume控制台接受数据

观察到控制台输出log中的内容：

![log](http://7xkfga.com1.z0.glb.clouddn.com/63390f5d1fb2857a0e2360252d519dca.jpg)

Flume在传完文件之后，将会修改文件的后缀，变为.COMPLETED