---
layout: post
title: "Install Spark on Mac OSX Yosemite"
date: 2015-07-17 18:30:50 +0800
comments: true
tag: [Big Data, Spark]
categories: [Big Data, Spark]
---


Spark是个好东西。

![Spark Logo](http://7xkfga.com1.z0.glb.clouddn.com/spark_logo.jpg)

Spark有以下四种运行模式：

- local: 本地单进程模式，用于本地开发测试Spark代码
- standalone：分布式集群模式，Master-Worker架构，Master负责调度，Worker负责具体Task的执行
- on yarn/mesos: ‌运行在yarn/mesos等资源管理框架之上，yarn/mesos提供资源管理，spark提供计算调度，并可与其他计算框架(如MapReduce/MPI/Storm)共同运行在同一个集群之上
- on cloud(EC2): 运行在AWS的EC2之上


在Spark上又有多个应用，尤其是`MLlib`，`Spark SQL`和`DataFrame`，提供给数据科学家们无缝接口去搞所谓Data Science

![spark stack](http://7xkfga.com1.z0.glb.clouddn.com/spark_stack.jpg)


本文记录一下我在Mac上安装Spark单机为分布式的过程


<!--more-->

## 1.安装环境

Spark依赖JDK 6.0以及Scala 2.9.3以上版本，安装好Java和Scala，然后配置好Java、Scala环境，最后再用`java -version`和`scala -version`验证一下

在`~/.bash_profile`中加入：

``` bash
# Setting PATH for scala
export SCALA_HOME=/usr/local/Cellar/scala/2.11.6
export PATH=$SCALA_HOME/bin:$PATH
```

别忘了

``` bash
source ~/.bash_profile
```

生效

由于在后面学习中主要会用到Spark的Python接口`pyspark`，所以在这儿也需要配置Python的环境变量：

``` bash
# Setting PATH for Python 2.7
# The orginal version is saved in .bash_profile.pysave
PATH="/usr/local/Cellar/python/2.7.9/bin:${PATH}"
export PATH
```

## 2.伪分布式安装

Spark的安装和简单，只需要将Spark的安装包download下来，加入PATH即可。这里我用的是[Spark 1.4.0](http://www.apache.org/dyn/closer.cgi/spark/spark-1.4.1/spark-1.4.1-bin-hadoop2.6.tgz)

当然，这里也可以使用Homebrew安装，那就更轻松了，直接

``` bash
$ brew install apache-spark
```
就搞定了，不过Homebrew安装没办法自己控制箱要安装的版本

这里我使用下载对Hadoop2.6的预编译版本安装

``` bash
cd /usr/local/Cellar/
wget http://www.apache.org/dyn/closer.cgi/spark/spark-1.4.0/spark-1.4.0-bin-hadoop2.6.tgz
tar zxvf spark-1.4.0-bin-hadoop2.6.tgz
```

设置Spark环境变量，`~/.bash_profile`：

``` bash
	export SPARK_MASTER=localhost
	export SPARK_LOCAL_IP=localhost
	export SPARK_HOME=/usr/local/Cellar/spark-1.4.0-bin-hadoop2.6
	export PATH=$PATH:$SCALA_HOME/bin:$SPARK_HOME/bin
	export PYTHONPATH=$SPARK_HOME/python:$SPARK_HOME/python/lib/py4j-0.8.2.1-src.zip:$PYTHONPATH
	export PATH="/usr/local/sbin:$PATH"
```

安装完成，貌似也没什么安装哈~

## 跑起来
执行Spark根目录下的pyspark就可以以交互的模式使用Spark了，这也是他的一个优点

![Spark Run success](http://7xkfga.com1.z0.glb.clouddn.com/SparkRun.jpg)

出现Spark的标志，那就说明安装成功了。下面再小配置下，让画面的log简单一点。在`$SPARK_HOME/conf/`下配置一下`log4j`的设置。

把`log4j.properties.template`文件复制一份，并删掉`.template`的扩展名

把这个文件中的`INFO`内容全部替换成`WARN`

## 在IPython中运行Spark
说Spark好，那么IPython更是一大杀器，这个以后再介绍。先说设置

首先，创建IPython的Spark配置

``` bash
$ ipython profile create pyspark
```

然后创建文件`$HOME/.ipython/profile_spark/startup/00-pyspark-setup.py`并添加：

``` python
import os
import sys
spark_home = os.environ.get('SPARK_HOME', None)
if not spark_home:
    raise ValueError('SPARK_HOME environment variable is not set')
sys.path.insert(0, os.path.join(spark_home, 'python'))
sys.path.insert(0, os.path.join(spark_home, 'python/lib/py4j-0.8.2.1-src.zip'))
execfile(os.path.join(spark_home, 'python/pyspark/shell.py'))
```

在IPython notebook中跑Spark

``` bash
$ ipython notebook --profile=pyspark
```

开始学习Spark吧！

参考： [Getting Started with Spark (in Python)](https://districtdatalabs.silvrback.com/getting-started-with-spark-in-python)
