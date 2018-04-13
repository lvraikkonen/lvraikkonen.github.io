---
title: MapReduce 实现倒排索引
date: 2018-04-08 09:23:29
categories: [Big Data, Hadoop]
tag: [Big Data, Hadoop]
---


## 概念

倒排索引是一种索引方法，被用来存储在全文搜索下某个单词在一个文档或者一组文档中的存储位置的映射，常被应用于搜索引擎和关键字查询的问题中。

以英文为例，下面是要被索引的文本：

```
T0 = "it is what it is"  
T1 = "what is it"  
T2 = "it is a banana"  
```

有两种不同的反向索引形式：

- 一条记录的水平反向索引（或者反向档案索引）包含每个引用单词的文档的列表。
- 一个单词的水平反向索引（或者完全反向索引）又包含每个单词在一个文档中的位置

我们就能得到下面的反向文件索引：
```
"a":      {2}
"banana": {2}
"is":     {0, 1, 2}
"it":     {0, 1, 2}
"what":   {0, 1}
```

检索的条件"what","is"和"it"将对应集合的交集。检索的条件"what", "is" 和 "it" 将对应这个集合： ${\displaystyle \{0,1\}\cap \{0,1,2\}\cap \{0,1,2\}=\{0,1\}}$。

下面得到的是第二种倒排索引，包含有`文档数量和单词结果在文档中的位置`组成的的成对数据。

```
"a":      {(2, 2)}
"banana": {(2, 3)}
"is":     {(0, 1), (0, 4), (1, 1), (2, 1)}
"it":     {(0, 0), (0, 3), (1, 2), (2, 0)} 
"what":   {(0, 2), (1, 0)}
```

<!--more-->

## 实现第一种倒排索引

在Map端，把单词和文件名作为key值，把单词词频作为value值。在Combine阶段，需要把单词设成key，文件名和词频作为value值，**将相同单词的所有记录发送给同一个Reducer处理**

在Reduce端，生成文档列表。

Mapper

``` java
public static class InvertedIndexMapper extends Mapper<Object, Text, Text, Text>{

    private Text word_filename_key = new Text();
    private Text word_frequency = new Text();
    private FileSplit split; // split对象

    @Override
    protected void map(Object key, Text value, Context context)
                throws IOException, InterruptedException {
        //获得<key,value>对所属的FileSplit对象
        split = (FileSplit) context.getInputSplit();

        StringTokenizer itr = new StringTokenizer(value.toString());
        while (itr.hasMoreTokens()){
            word_filename_key.set(itr.nextToken() + ":" + split.getPath().toString());
            word_frequency.set("1");
            context.write(word_filename_key, word_frequency);
        }
    }
    //map output:   I:1.txt    list{1,1,1}
}
```

Combiner

``` java
// Combiner merge word frequency
    public static class InvertedIndexCombiner extends Reducer<Text, Text, Text, Text>{

    private Text new_value = new Text();

    @Override
    public void reduce(Text key, Iterable<Text> values, Context context)
                throws IOException, InterruptedException {
        int sum = 0;
        for (Text value: values){
            sum += Integer.parseInt(value.toString());
        }
        int splitIndex = key.toString().indexOf(":");

        // 文件名和词频作为value值
        new_value.set(key.toString().substring(splitIndex + 1) + ":" + sum);

        // 单词设成key
        key.set(key.toString().substring(0, splitIndex));

        context.write(key, new_value);

        System.out.println("key: " + key);
        System.out.println("value: " + new_value);
    }
    // combiner output: I         1.txt:3
}
```

Reducer

``` java
// 生成单词的文档列表
public static class InvertedIndexReducer extends Reducer<Text, Text, Text, Text>{

    private Text result = new Text();

    @Override
    public void reduce(Text key, Iterable<Text> values, Context context)
                throws IOException, InterruptedException {
        String fileList = "";
        for (Text value: values){
            fileList += " " + value.toString() + ";";
        }
        result.set(fileList);
        context.write(key, result);
    }
}
```

## 实现第二种倒排索引

使用自定义的数据类型