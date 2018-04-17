---
title: 记录配置伪分布式Hadoop环境遇到的坑
date: 2018-04-17 14:51:54
categories: [Hadoop]
tag: [Hadoop]
---

## 问题

本地mac搭了个Hadoop3.0的伪分布式环境，HDFS运行正常、MapReduce单机模式也正常，JPS后台守护进程都正常。但是写了个MapReduce任务，想去跑一下，结果卡在了Map 100% Reduce 0%

![stuckAtReduce](http://7xkfga.com1.z0.glb.clouddn.com/8decc87de4ed7b15bd9df527e063dbd5.jpg)

去stackoverflow查了一圈，基本都在说运行环境没有分配足够的内存导致的，从container的syslog里面只有下面的ERROR：

```
Error: org.apache.hadoop.mapreduce.task.reduce.Shuffle$ShuffleError: error in shuffle in fetcher#1
	at org.apache.hadoop.mapreduce.task.reduce.Shuffle.run(Shuffle.java:134)
	at org.apache.hadoop.mapred.ReduceTask.run(ReduceTask.java:377)
	at org.apache.hadoop.mapred.YarnChild$2.run(YarnChild.java:174)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:422)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1965)
	at org.apache.hadoop.mapred.YarnChild.main(YarnChild.java:168)
Caused by: java.io.IOException: Exceeded MAX_FAILED_UNIQUE_FETCHES; bailing-out.
	at org.apache.hadoop.mapreduce.task.reduce.ShuffleSchedulerImpl.checkReducerHealth(ShuffleSchedulerImpl.java:396)
	at org.apache.hadoop.mapreduce.task.reduce.ShuffleSchedulerImpl.copyFailed(ShuffleSchedulerImpl.java:311)
	at org.apache.hadoop.mapreduce.task.reduce.Fetcher.openShuffleUrl(Fetcher.java:291)
	at org.apache.hadoop.mapreduce.task.reduce.Fetcher.copyFromHost(Fetcher.java:330)
	at org.apache.hadoop.mapreduce.task.reduce.Fetcher.run(Fetcher.java:198)
```

<!-- more -->

## 内存问题解决方法

歪个楼，看了不少的关于内存不足的分析，也学了不少，记在下面以备以后需要。

引用自[MapReduce任务Shuffle Error错误](https://blog.csdn.net/dslztx/article/details/46445725)

> shuffle过程的输入是：map任务的输出文件，它的输出接收者是：运行reduce任务的机子上的内存
> buffer，并且shuffle过程以并行方式运行
> 参数mapreduce.reduce.shuffle.input.buffer.percent控制运行reduce任务的机子上多少比例的
> 内存用作上述buffer(默认值为0.70)，参数mapreduce.reduce.shuffle.parallelcopies控制
> shuffle过程的并行度(默认值为5)
> 那么”mapreduce.reduce.shuffle.input.buffer.percent” * 
> “mapreduce.reduce.shuffle.parallelcopies” 必须小于等于1，否则就会出现如上错误
> 因此，我将mapreduce.reduce.shuffle.input.buffer.percent设置成值为0.1，
> 就可以正常运行了（设置成0.2，还是会抛同样的错）
> 
> 另外，可以发现如果使用两个参数的默认值，那么两者乘积为3.5，大大大于1了，
> 为什么没有经常抛出以上的错误呢？
> 
> 首先，把默认值设为比较大，主要是基于性能考虑，将它们设为比较大，可以大大加快从map复制数据的速度
> 其次，要抛出如上异常，还需满足另外一个条件，就是map任务的数据一下子准备好了等待shuffle去复制，
> 在这种情况下，就会导致shuffle过程的“线程数量”和“内存buffer使用量”都是满负荷的值，自然就造成
> 了内存不足的错误；而如果map任务的数据是断断续续完成的，那么没有一个时刻shuffle过程的“线程数
> 量”和“内存buffer使用量”是满负荷值的，自然也就不会抛出如上错误
> 另外，如果在设置以上参数后，还是出现错误，那么有可能是运行Reduce任务的进程的内存总量不足，可以
> 通过mapred.child.java.opts参数来调节，比如设置mapred.child.java.opts=-Xmx2024m
> reduce会在map执行到一定比例启动多个fetch线程去拉取map的输出结果，放到reduce的内存、磁盘中，
> 然后进行merge。当数据量大时，拉取到内存的数据就会引起OOM，所以此时要减少fetch占内存的百分比，
> 将fetch的数据直接放在磁盘上。
> 
> mapreduce.reduce.shuffle.memory.limit.percent：每个fetch取到的map输出的大小能够占的内
> 存比的大小。默认是0.25。因此实际每个fetcher的输出能放在内存的大小是reducer的java heap 
> size*0.9*0.25

## 我的问题解决

言归正传，按照上面的解决办法，问题依然存在，然后我删掉了Hadoop重新配置了一个全新的，跑一下伪分布式的hadoop-example，依然是这样，大脑都快炸了，啊啊啊

后来发现有人说是本机host的问题，把计算机名加入到host里面

```
127.0.0.1   localhost
-- added
127.0.0.1   my_computername
...
```

好了~~~内牛满面啊

## Hadoop伪分布式环境配置

### core-site.xml

``` xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>file:/Users/lvshuo/bigdata/hadoop/tmp</value>
    </property>
    <property>
        <name>io.file.buffer.size</name>
        <value>131702</value>
    </property>
</configuration>
```

### hdfs-site.xml

``` xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>

    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/Users/lvshuo/bigdata/hadoop/tmp/hdfs/namenode</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/Users/lvshuo/bigdata/hadoop/tmp/hdfs/datanode</value>
    </property>
</configuration>
```

### yarn-site.xml

``` xml
<property>
     <name>yarn.nodemanager.aux-services</name>
     <value>mapreduce_shuffle</value>
</property>
```

### mapred-site.xml

``` xml
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <property>
    <name>mapred.child.java.opts</name>
    <value>-Xmx4096m</value>
  </property>

  <property>  
    <name>mapreduce.reduce.shuffle.memory.limit.percent</name>  
    <value>0.1</value>  
    <description>Expert: Maximum percentage of the in-memory limit that a  
    single shuffle can consume</description>  
  </property> 

  <property>
    <name>mapreduce.application.classpath</name>
    <value>
    /Users/lvshuo/bigdata/hadoop/etc/hadoop:/Users/lvshuo/bigdata/hadoop/share/hadoop/common/lib/*:/Users/lvshuo/bigdata/hadoop/share/hadoop/common/*:/Users/lvshuo/bigdata/hadoop/share/hadoop/hdfs:/Users/lvshuo/bigdata/hadoop/share/hadoop/hdfs/lib/*:/Users/lvshuo/bigdata/hadoop/share/hadoop/hdfs/*:/Users/lvshuo/bigdata/hadoop/share/hadoop/mapreduce/*:/Users/lvshuo/bigdata/hadoop/share/hadoop/yarn:/Users/lvshuo/bigdata/hadoop/share/hadoop/yarn/lib/*:/Users/lvshuo/bigdata/hadoop/share/hadoop/yarn/*
    </value>
  </property>
</configuration>
```