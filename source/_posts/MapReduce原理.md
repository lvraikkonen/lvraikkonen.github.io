---
title: MapReduce原理
date: 2018-04-04 16:50:11
categories: [Big Data, Hadoop]
tag: [Big Data, Hadoop]
---

[MapReduce Paper](https://research.google.com/archive/mapreduce.html)

MapReduce是Google提出的一个软件架构，用于大规模数据集（大于1TB）的并行运算。MapReduce最本质的两个过程就是Map和Reduce，思想来源于函数式编程。

初学MapReduce，写了WordCount入门程序后，觉得编写MapReduce程序只需要实现Map和Reduce函数就可以了，后来觉得，这个框架隐藏的细节还是需要好好了解一下的。下面这个图基本描述了MapReduce的整个过程：

![MapReduce pipeline](http://7xkfga.com1.z0.glb.clouddn.com/868d6475efd595a6ea9b1d6f248a32c7.jpg)

其中，Map阶段、Reduce阶段比较好理解，但是Shuffle阶段的这个细节还是很神奇的。下面简单介绍下MapReduce各个阶段。

<!-- more -->

## InputFormat类

该类的作用是将输入的文件和数据分割成许多小的split文件，并将split的每个行通过LineRecorderReader解析成<Key,Value>,通过job.setInputFromatClass()函数来设置，默认的情况为类TextInputFormat，其中Key默认为字符偏移量，value是该行的值。

## Mapper类

<k1, v1> --map--> <k2, v2>

根据输入的<Key,Value>对生成中间结果，默认的情况下使用Mapper类，该类将输入的<Key,Value>对进行映射并作为中间结果输出，通过job.setMapperClass()实现。提供了Mapper的抽象类，需要实现map方法

![Mapper](http://7xkfga.com1.z0.glb.clouddn.com/9342a1d5774a69c1672855e4337ec9a4.jpg)


## Combiner类

实现combine函数，该类的主要功能是合并相同的key键，通过job.setCombinerClass()方法设置，默认为null，不合并中间结果

## Partition类

Map的结果会通过partition分发到指定的Reducer上，哪个key到哪个Reducer的分配过程是由Partitioner规定的。可以通过job.setPartitionerClass()方法进行设置，partition是分割map每个节点的结果，按照key分别映射给不同的reduce，也是可以自定义的，需要实现getPartition函数。

``` java
getPartition(Text key, Text value, int numPartitions)
```

MapReduce默认使用hashPartitioner，计算方法如下：

``` java
reducer = (key.hashCode() & Integer.MAX_VALUE) % numReduceTasks
```

## Reducer类

<k2, value-list> --reduce--> <k3, v3>

将中间结果合并，得到中间结果。通过job.setReduceCalss()方法进行设置，默认使用Reducer类，实现reduce方法。

![](http://7xkfga.com1.z0.glb.clouddn.com/840a93c06c0c2f91ec6669bc57476040.jpg)


## OutputFormat类

该类负责输出结果的格式。可以通过job.setOutputFormatClass()方法进行设置。默认使用TextOUtputFormat类，得到<Key,value>对。

整个mapreduce任务的基本流程如下程序所示：

``` java
public static void main(String[] args) throws IOException {
        Configuration conf = new Configuration();
        Job job = new Job(conf);
        job.setInputFormatClass(TextInputFormat.class);
        job.setMapperClass(Mapper.class);
        job.setCombinerClass(null);
        job.setPartitionerClass(HashPartitioner.class);
        job.setReducerClass(Reducer.class);
        job.setOutputFormatClass(TextOutFormat.class);
    }
}
```

## Shuffle 过程

Shuffle过程是MapReduce的核心，Shuffle过程的性能与整个MapReduce的性能直接相关。Shuffle过程包含在Map和Reduce两端中。

Map端的Shuffle过程是对Map的结果进行分区(partition)、排序(sort)、和溢写(spill)，然后将属于同一个划分的输出合并在一起并写在磁盘上，同时按照不同的划分结果发送给对应的Reduce任务。

Reduce端会将各个Map发送来的属于同一个划分的输出进行合并、然后对合并结果进行排序，最后交给Reduce处理。

![Shuffle](http://7xkfga.com1.z0.glb.clouddn.com/63d9b9c99d588f5fe9348cefc051617c.jpg)

### Map 端

Map端的Shuffle过程，包含了切分(Partition)过程、溢写(Sort&Combine)过程。和合并(Merge)过程。

![Map_Shuffle](http://7xkfga.com1.z0.glb.clouddn.com/2e2be46b87e56964605386a6690c3c1e.jpg)

1. Partition分区

mapreduce提供Partitioner接口，它的作用就是根据key或value及reduce的数量来决定当前的这对输出数据最终应该交由哪个reducetask处理（分区）。MapRedcuce默认使用hashPartitioner。

注意：虽然Partitioner接口会计算出一个值来决定某个输出会交给哪个reduce去处理，但是在缓冲区中并不会实现物理上的分区，而是将结果加载key-value后面。物理上的分区实在磁盘上进行的。

2. 环形缓冲区

map在内存中有一个环形缓冲区(字节数组实现)，用于存储任务的输出。默认是100M，这其中80%的容量用来缓存，当这部分容量满了的时候会启动一个溢出线程进行溢出操作，写入磁盘形成溢写文件；在溢出的过程中剩余的20%对新生产的数据继续缓存。【简单来说就是别读边写】但如果再次期间缓冲区被填满，map会阻塞直到写磁盘过程完成。

阈值是可以设置的，但一般默认就可以了。

1）环形缓冲区大小:`mapred-site.xml`中设置`mapreduce.task.io.sort.mb`的值

2）环形缓冲区溢写的阈值:`mapred-site.xml`中设置`mapreduce.map.sort.spill.percent`的值

作用：为什么要分区呢？？由于map()处理后的数据量可能会非常大，所以如果由一个reduce()处理效率不高，为了解决这个问题可以用分布式的思想，一个reduce()解决不了，就用多个reduce节点。一般来说有几类分区就对应有几个reduce节点，把相同分区交给一个reduce节点处理。

3. Spill溢写sort排序

缓冲区的数据写到磁盘前，会对它进行一个二次快速排序，首先根据数据所属的partition （分区）排序，然后每个partition中再按Key 排序。输出包括一个索引文件和数据文件。如果设定了Combiner，将在排序输出的基础上运行。【Combiner】就是一个简单Reducer操作，它在执行Map 任务的节点本身运行，先对Map 的输出做一次简单Reduce，使得Map的输出更紧凑，更少的数据会被写入磁盘和传送到Reducer。临时文件会在map任务结束后删除。

4. merge文件合并
每次溢写会在磁盘上生成一个溢写文件，如果map的输出结果很大，有多次这样的溢写发生，磁盘上相应的就会有多个溢写文件存在。因为最终的文件只有一个，所以需要将这些溢写文件归并到一起，这个过程就叫做Merge。【merge就是多个溢写文件合并到一个文件】所以可能也有相同的key存在，在这个过程中如果client设置过Combiner，也会使用Combiner来合并相同的key。

map端就处理完了，接下来就是reduce端了。

### Reduce 端

Map端合并最终生成的这个文件也存放在TaskTracker够得着的某个本地目录内，每个reduce task不断地从JobTracker那里获取map task是否完成的信息，如果reduce task得到通知，获知某台TaskTracker上的map task执行完成，Shuffle的后半段过程开始启动。

1. copy复制

reduce端默认有5个数据复制线程从map端复制数据，其通过Http方式得到Map对应分区的输出文件。reduce端并不是等map端执行完后将结果传来，而是直接去map端去Copy输出文件。

2. Merge合并

reduce端的shuffle也有一个环形缓冲区，它的大小要比map端的灵活（由JVM的heapsize设置），由Copy阶段获得的数据，会存放的这个缓冲区中，同样，当到达阀值时会发生溢写到磁盘操作，这个过程中如果设置了Combiner也是会执行的，这个过程会一直执行直到所有的map输出都被复制过来，如果形成了多个磁盘文件还会进行合并，最后一次合并的结果作为reduce的输入而不是写入到磁盘中。

3. reduce执行

当Reducer的输入文件确定后，整个Shuffle操作才最终结束。之后就是Reducer的执行了，最后Reducer会把结果存到HDFS上。


## 案例：分区函数使用

模拟数据：

|            |     |     |    |
| ---------- | --- | --- | -- |
| aa |  1 | 2 | |
| bb |  2 | 22 | |
|cc | 11  | | |
|dd | 1 | | |
|ee | 99 | 99999 | |
|ff | 12 | 23123 | |
|gg | 12 | 41441 | 442 |
|hh | 11 | 41435 | 412 |

现在需要根据字段的个数分区，字段大于3个的为long，小于3个的为short，等于3个的为right。

下面是map、getPartition和reduce函数的实现：

``` java
public static class MyPartitionerMapper extends Mapper<LongWritable, Text, Text, Text>{

        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
            String [] arr_value = value.toString().split(" ");
            if (arr_value.length > 3){
                context.write(new Text("long"), value);
            } else if (arr_value.length < 3){
                context.write(new Text("short"), value);
            } else {
                context.write(new Text("right"), value);
            }
        }
    }

public static class MyPartitionerPartitioner extends Partitioner<Text, Text>{

        @Override
        public int getPartition(Text key, Text value, int numPartitions) {
            int result = 0;
            if (key.toString().equals("long")){
                result = 0 % numPartitions;
            } else if (key.toString().equals("short")){
                result = 1 % numPartitions;
            } else{
                result = 2 % numPartitions;
            }
            return result;
        }
    }

public static class MyPartitionerReducer extends Reducer<Text, Text, Text, Text>{

        @Override
        protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
            // just output data from map
            for (Text val: values){
                context.write(key, val);
            }
        }
    }
```