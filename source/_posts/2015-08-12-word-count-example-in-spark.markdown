---
layout: post
title: "利用Spark进行单词计数"
date: 2015-08-12 13:38:08 +0800
comments: true
categories: [Big Data, Spark]
tag: [Spark, Big Data]
---

这里就不再介绍Spark了，这篇文章主要记录一下关于Spark的核心`RDD`的相关操作以及以单词计数这个简单的例子，描述一下Spark的处理流程。

## Spark RDD
Spark是以RDD概念为中心运行的。RDD是一个容错的、可以被并行操作的元素集合。创建一个RDD有两个方法：在你的驱动程序中并行化一个已经存在的集合；从外部存储系统中引用一个数据集，这个存储系统可以是一个共享文件系统，比如HDFS、HBase或任意提供了Hadoop输入格式的数据来源。

RDD支持两类操作：

- 转换(Transform)
- 动作(Action)

还是不翻译的好，下面都用英文描述。`Transform`：用于从已有的数据集转换产生新的数据集，Transform的操作是`Lazy Evaluation`的，也就是说这条语句过后，转换并没有发生，而是在下一个`Action`调用的时候才会返回结果。`Action`：用于计算结果并向驱动程序返回结果。

<!--more-->

演示一下上面两种基本操作：

```python
lines = sc.textFile("data.txt")
lineLength = line.map(lambda x: len(x))
totalLength = lineLength.reduce(lambda x, y: x + y)
```

第一行是有外部存储系统中创建一个RDD对象，第二行定义map操作，是一个`Transform`操作，由于`Lazy Evaluation`，对象`lineLength`并没有立即计算得到。第三行，`reduce`是一个`Action`操作，这时，Spark将整个计算过程划分成许多任务在多台机器上并行执行，每台机器运行自己部分的map操作和reduce操作，最终将自己部分的运算结果返回给驱动程序。

```python
	lineLength.persist()
	# lineLength.cache()
```

这一行，Spark将`lineLength`对象保存在内存中，以便后面计算中使用。Spark的一个重要功能就是在将数据集持久化（或缓存）到内存中以便在多个操作中重复使用。

以上就是RDD的一些基本操作，API文档中写的都很清楚，我就不多说了。

## 统计一篇文档中单词的个数
首先，写一个函数，用来计算单词个数

```python
def wordCount(wordListRDD):
    wordCountsCollected = wordListRDD
                                .map(lambda x: (x, 1))
                                .reduceByKey(lambda x, y: x + y)
    return wordCountsCollected
```

使用正则表达式清理原始文本

```python
	import re
	import string
	def removePunctuation(text):
	    regex = re.compile('[%s]' % re.escape(string.punctuation))
	    return regex.sub('', text).lower().strip()
	print removePunctuation(' No under_score!')
```

去读文件内容到RDD中

```python
	import os.path
	baseDir = os.path.join('data')
	inputPath = os.path.join('cs100', 'lab1', 'shakespeare.txt')
	fileName = os.path.join(baseDir, inputPath)
	# 
	shakespeareRDD = (sc.textFile(fileName, 8).map(removePunctuation))
	print '\n'.join(shakespeareRDD.zipWithIndex().map(lambda (l, num): '{0}: {1}'.format(num,l)).take(15))
```

这时候，需要把单词通过空格隔开，然后过滤掉为空的内容

```python
shakespeareWordsRDD = shakespeareRDD.flatMap(lambda x: x.split())
shakespeareWordCount = shakespeareWordsRDD.count()
print shakespeareWordsRDD.top(5)
shakeWordsRDD = shakespeareWordsRDD
```

统计出出现次数前15多的单词以及个数：

```python
top15WordAndCounts = wordCount(shakeWordsRDD).takeOrdered(15, key=lambda (k, v): -v)
print '\n'.join(map(lambda (w, c): '{0}: {1}'.format(w, c), top15WordsAndCounts))
```

输出结果为：

word | count
---- | ------
the: | 27361
and: | 26028
i:   | 20681
to:  | 19150
of:  | 17463
a:   | 14593
you: | 13615
my:  | 12481
in:  | 10956
that:| 10890
is:  | 9134
not: | 8497
with:| 7771
me:  | 7769
it:  | 7678
     |


Spark是用Scala写出来的，所以可想而知如果用Scala写的效率会比Python高一些，在这儿顺便贴一个Scala版写的WordCount：

```scala
val wordCounts = textFile.flatMap(line => line.split(" "))
                         .map(word => (word, 1))
                         .reduceByKey((a, b) => a + b)
```

真是简洁，Spark真好，嘿嘿~
