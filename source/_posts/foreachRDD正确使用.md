---
title: foreachRDD正确使用
date: 2018-09-04 11:16:47
tags: 
- Big Data
- Spark
- 流处理
categories:
- Big Data
- Spark
---

上面在Spark-Streaming中介绍了foreach，`dstream.foreachRDD`是一个功能强大的原语primitive，它允许将数据发送到外部系统。输出操作实际上是允许外部系统消费转换后的数据，它们触发的实际操作是DStream转换。所以要掌握它，对它要有深入了解。下面就是一些常见的错误用法。

![](http://7xkfga.com1.z0.glb.clouddn.com/62d2dab6799b4da19b73eb872444cda2.jpg)

<!-- more -->

## 错误使用

在Spark驱动中创建一个连接对象，在 Spark worker 中尝试调用这个连接对象将记录保存到RDD中，很容易下出下面的代码：

``` scala
dstream.foreachRDD { rdd =>
  val connection = createNewConnection()  // executed at the driver
  rdd.foreach { record =>
    connection.send(record) // executed at the worker
  }
}
```

这是不正确的，因为这需要先序列化连接对象，然后将它从driver端发送到worker中。这样的连接对象在机器之间不能传送。通常会报不能序列化的错误:

![](http://7xkfga.com1.z0.glb.clouddn.com/d243270e098a71586531d026096a07cf.jpg)

正确做法是在worker中创建连接对象

``` scala
dstream.foreachRDD { rdd =>
  rdd.foreach { record =>
    val connection = createNewConnection()
    connection.send(record)
    connection.close()
  }
}
```

但是这么做，很明显也有问题：为每一条记录都创建一个连接对象。下面开始改进

## 改进

创建一个连接对象会有资源和时间的开销，为每条记录创建和销毁连接对象会有非常高的开支，第一种优化的方案是：为RDD的每个分区创建一个连接对象：

``` scala
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    val connection = createNewConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    connection.close()
  }
}
```

## 进一步改进

创建连接对象对资源和时间要求很高，那么可以利用连接池来维护有限的连接对象资源。

创建静态连接对象池：

``` java
public class ConnectionPool {

    private static ComboPooledDataSource dataSource = new ComboPooledDataSource();

    static {
        dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/dbs");
        dataSource.setUser("user");
        dataSource.setPassword("pwd");
        dataSource.setMaxPoolSize(50);
        dataSource.setMinPoolSize(2);
        dataSource.setInitialPoolSize(10);
        dataSource.setMaxStatements(100);
    }

    public static Connection getConnection(){
        try{
            return dataSource.getConnection();
        } catch(SQLException e){
            e.printStackTrace();
        }
        return null;
    }

    public static void returnConnection(Connection conn){
        if (conn != null){
            try {
                conn.close();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }

}
```

这样，每使用一个连接对象将一批数据写入外部系统之后，就将该连接对象放回连接池。

``` scala
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    // ConnectionPool is a static, lazily initialized pool of connections
    val connection = ConnectionPool.getConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    ConnectionPool.returnConnection(connection)  // return to the pool for future reuse
  }
}
```

## 再进一步改进

经过上面连接池的改进，基本上性能已经难满足要求了。如果RDD数据量比较大，也可以考虑分批次写入外部存储。

对于rdd中一个分区的数据，首先从连接池中获取一个连接对象，然后准备好SQL语句，超过500条记录后，向数据库提交一次数据(如果不足500条，就将这批数据放到一个批次)，提交数据后，将连接对象放回连接池以便后面使用。

``` scala
dstream.foreachRDD { (rdd, time) =>
    rdd.foreachPartition { partitionRecords =>
        val conn = ConnectionPool.getConnection
        conn.setAutoCommit(false)
        val statement = conn.prepareStatement(s"insert into wordcount(ts, word, count) values (?, ?, ?)")
        partitionRecords.zipWithIndex.foreach { case ((word, count), index) =>
          statement.setLong(1, time.milliseconds)
          statement.setString(2, word)
          statement.setInt(3, count)
          statement.addBatch()
          if (index != 0 && index % 500 == 0) {
            statement.executeBatch()
            conn.commit()
          }
        }
        statement.executeBatch()
        statement.close()
        conn.commit()
        conn.setAutoCommit(true)
        ConnectionPool.returnConnection(conn)
    }
}
```

参考：[Design Patterns for using foreachRDD](https://spark.apache.org/docs/latest/streaming-programming-guide.html#design-patterns-for-using-foreachrdd)