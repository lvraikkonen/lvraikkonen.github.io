---
layout: post
title: "Install Hadoop on Yosemite"
date: 2015-07-17 17:20:58 +0800
comments: true
categories: [Big Data, Hadoop]
tag: [Big Data, Hadoop]
---



终于进入正题，开始写一写我在大数据方面走过的路，自认为被其他人甩下了，所以一定要紧追而上。 首先现在我的Mac上装上单节点的Hadoop玩玩，个人感觉Apache系列的项目，只要download下来，再配置以下参数就能玩了。

![Hadoop Logo](http://7xkfga.com1.z0.glb.clouddn.com/hadooplogo-e1434684861708.png)

在这里感谢如下教程：

[INSTALLING HADOOP ON MAC](http://amodernstory.com/2014/09/23/installing-hadoop-on-mac-osx-yosemite/)

[Writing an Hadoop MapReduce Program in Python](http://www.michael-noll.com/tutorials/writing-an-hadoop-mapreduce-program-in-python/)

下面开始吧

<!--more-->

## 准备
这个阶段主要就是准备一下JAVA的环境，Mac默认是安装了Java的，不过版本就不知道了，这个还是自己安装一下并且写到环境变量里来得踏实

[Java Download](http://www.oracle.com/technetwork/java/javase/downloads/java-archive-downloads-javase7-521261.html#jdk-7u80-oth-JPR)

安装完之后，Java被装到了这个位置
```
/Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home
```
，把这个地址写到系统的环境变量文件`.bash_profile`里

``` bash
# Setting PATH for java
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home
export PATH=$JAVA_HOME/bin:$PATH
```


**配置SSH**

Nothing needs to be done here if you have already generated ssh keys. To verify just check for the existance of ~/.ssh/id_rsa and the ~/.ssh/id_rsa.pub files. If not the keys can be generated using

``` bash
$ ssh-keygen -t rsa
```

Enable Remote Login

“System Preferences” -> “Sharing”. Check “Remote Login”
Authorize SSH Keys
To allow your system to accept login, we have to make it aware of the keys that will be used

``` bash
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```
Let’s try to login.

``` bash
$ ssh localhost
Last login: Fri Mar  6 20:30:53 2015
$ exit
```

## 安装Homebrew
在Mac上，最好的包安装工具就是Homebrew，执行下面代码安装：

``` bash
$ ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

## 安装Hadoop
我去，这么简单，直接

``` bash
$ brew install hadoop
```

就搞定了。。。
这样，Hadoop会被安装在`/usr/local/Cellar/hadoop`目录下

下面才是重点，配置Hadoop

## 配置Hadoop
### hadoop-env.sh

该文件在`/usr/local/Cellar/hadoop/2.6.0/libexec/etc/hadoop/hadoop-env.sh`

找到如下这行：

``` bash
export HADOOP_OPTS="$HADOOP_OPTS -Djava.net.preferIPv4Stack=true"
```

改为：

``` bash
export HADOOP_OPTS="$HADOOP_OPTS -Djava.net.preferIPv4Stack=true -Djava.security.krb5.realm= -Djava.security.krb5.kdc="
```

### Core-site.xml

该文件在`/usr/local/Cellar/hadoop/2.6.0/libexec/etc/hadoop/core-site.xml`

``` xml
  <property>
     <name>hadoop.tmp.dir</name>
     <value>/usr/local/Cellar/hadoop/hdfs/tmp</value>
     <description>A base for other temporary directories.</description>
  </property>
  <property>
     <name>fs.default.name</name>                                     
     <value>hdfs://localhost:9000</value>                             
  </property> 
```

### mapred-site.xml

文件在`/usr/local/Cellar/hadoop/2.6.0/libexec/etc/hadoop/mapred-site.xml`

``` xml
<configuration>
    <property>
        <name>mapred.job.tracker</name>
        <value>localhost:9010</value>
    </property>
</configuration>
```

### hdfs-site.xml

文件在`/usr/local/Cellar/hadoop/2.6.0/libexec/etc/hadoop/hdfs-site.xml`

``` xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

这就配置好了

### 添加启动关闭Hadoop快捷命令

为了以后方便使用Hadoop，在`.bash_profile`中添加

``` bash
alias hstart="/usr/local/Cellar/hadoop/2.6.0/sbin/start-dfs.sh;/usr/local/Cellar/hadoop/2.6.0/sbin/start-yarn.sh"
alias hstop="/usr/local/Cellar/hadoop/2.6.0/sbin/stop-yarn.sh;/usr/local/Cellar/hadoop/2.6.0/sbin/stop-dfs.sh"
```

执行下面命令将配置生效：

``` bash
source ~/.bash_profile
```

以后就能使用命令`hstart`启动Hadoop服务，`hstop`关闭Hadoop

### 格式化HDFS

在使用Hadoop之前，还需要将HDFS格式化

``` bash
$ hdfs namenode -format
```

## Running Hadoop
奔跑吧Hadoop

```
$ hstart
```

使用`jps`命令查看Hadoop运行状态

```
$ jps
18065 SecondaryNameNode
18283 Jps
17965 DataNode
18258 NodeManager
18171 ResourceManager
17885 NameNode
```

下面是几个很有用的监控Hadoop地址：

- Resource Manager: [http://localhost:50070](http://localhost:50070)
- JobTracker: [http://localhost:8088](http://localhost:8088)
- Specific Node Information: [http://localhost:8042](http://localhost:8042)

停止Hadoop：

```
$ hstop
```

## 添加Hadoop环境变量
为了以后安装Spark等方便，在`~/.bash_profile`配置中添加Hadoop环境变量

``` bash
	# Setting PATH for hadoop
	export HADOOP_HOME=/usr/local/Cellar/hadoop/2.6.0/libexec
	export PATH=$HADOOP_HOME/bin:$PATH

	export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
```

## 可能遇到的问题
跑起来之后，或者在跑起来的过程中，可能会遇到各种问题，由于控制台命令太多，很难知道到底是哪儿出的问题，所以我总结出几个我遇到的问题和解决方法，分享给大家。
TBD

1. NameNode启动失败
2. 

大功告成！可以在Hadoop上跑几个MapReduce任务了。