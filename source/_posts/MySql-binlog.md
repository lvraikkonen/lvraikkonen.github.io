---
title: MySQL Binlog以及Canal简单使用
date: 2018-05-16 19:30:02
tags: 
    - MySQL
    - ETL
---

在ETL过程中，如何获取增量数据是完成ETL过程的主要解决问题，前面文章[ETL增量数据的捕获](https://lvraikkonen.github.io/2018/05/07/ETL%E5%A2%9E%E9%87%8F%E6%95%B0%E6%8D%AE%E7%9A%84%E6%8D%95%E8%8E%B7/)中，使用日志对比的方法是一种比较好的获取增量数据流的方式，那篇文章里主要介绍了SQL Server的`Change Data Capture`这么个功能来完成增量数据的获取任务，这篇文章介绍一下对于MySQL来说，如何通过读取binlog来获取增量数据。


## MySQL的日志

MySQL 的日志包括错误日志（ErrorLog），更新日志（Update Log），二进制日志（Binlog），查询日志（Query Log），慢查询日志（Slow Query Log）等。在默认情况下，系统仅仅打开错误日志，关闭了其他所有日志。

<!-- more -->

## 开启Binlog

MySQL 默认没有开启 Binlog，下面要开启Binlog。mac上使用brew安装的mysql，默认安装后的目录是/usr/local/Cellar，在这个目录中没有MySQL的配置文件`my.cnf`，在系统中查看`my.cnf`的路径

``` shell
mysql --help --verbose | grep my.cnf
```

![](http://7xkfga.com1.z0.glb.clouddn.com/d7e725a28b8f5e108a40073f7902b9a8.jpg)


可以看到MySQL是按照这个顺序来读取配置文件的，下面在目录`/etc/`下新建文件``my.cnf`，并添加如下来开启Binlog

```
[mysqld]
#log_bin
log-bin = mysql-bin #开启binlog
binlog-format = ROW #选择row模式
server_id = 1 #配置mysql replication需要定义，不能和canal的slaveId重复 
```

重启MySQL服务

``` shell
mysql.server restart
```

查看Binlog是否开启

``` shell
show variables like "%log_bin%";
```

![](http://7xkfga.com1.z0.glb.clouddn.com/dcd6b992ddb3beec4f3541632e3a4304.jpg)

## 简单查看Binlog内容

首先创建一个表test，然后向表中插入两条数据

``` sql
create table test(id int, name varchar(20)) engine=innodb charset=utf8;
insert into test(id, name) values(1, "aaa");
insert into test(id, name) values(2, "bbb");
```

显示当前使用的Binlog及所处位置，然后查看这个二进制文件内容

```
show master status;
show binlog events in "mysql-bin.000002"; 
```

![](http://7xkfga.com1.z0.glb.clouddn.com/198930e0e777a5f2b30c88cefdeef0b7.jpg)

### 使用MySQL内置的mysqlbinlog工具查看和操作Binlog

``` shell
$ mysqlbinlog -v mysql-bin.000002
```

![](http://7xkfga.com1.z0.glb.clouddn.com/b5797847f10b3018073ce5d9f56f3142.jpg)

可以看到刚才创建表和插入数据的操作都被记录下来了。关于Binlog更详细内容，参考MySQL官方文档[The Binary Log](https://dev.mysql.com/doc/internals/en/binary-log.html)

## 使用Canal监听MySQL增量数据

[Canal](https://github.com/alibaba/canal)是阿里开源的MySQL数据库Binlog的增量订阅&消费组件

MySQL的Binlog有两个重要用途：主从复制和数据恢复，关于主从复制的内容，参考[MySQL主备复制原理、实现及异常处理](https://blog.csdn.net/u013256816/article/details/52536283)

这里其实利用了MySQL支持主从复制的原理，Master产生Binlog记录增删改语句，通过将Binlog发送到Slave节点从而完成Binlog的解析。下图是原理图

![canal工作原理](https://camo.githubusercontent.com/46c626b4cde399db43b2634a7911a04aecf273a0/687474703a2f2f646c2e69746579652e636f6d2f75706c6f61642f6174746163686d656e742f303038302f333130372f63383762363762612d333934632d333038362d393537372d3964623035626530346339352e6a7067)

Canal是CS结构

### 安装配置Canal服务端

下载canal releases包，解压之后，编辑conf/example文件夹下的`instance.properties`：

```
#################################################
## mysql serverId
canal.instance.mysql.slaveId=1234
# position info
canal.instance.master.address=127.0.0.1:3306
canal.instance.master.journal.name=
canal.instance.master.position=
canal.instance.master.timestamp=

# table meta tsdb info
canal.instance.tsdb.enable=true
canal.instance.tsdb.dir=${canal.file.data.dir:../conf}/${canal.instance.destination:}
canal.instance.tsdb.url=jdbc:h2:${canal.instance.tsdb.dir}/h2;CACHE_SIZE=1000;MODE=MYSQL;
#canal.instance.tsdb.url=jdbc:mysql://127.0.0.1:3306/canal_tsdb
canal.instance.tsdb.dbUsername=canal
canal.instance.tsdb.dbPassword=canal


#canal.instance.standby.address =
#canal.instance.standby.journal.name =
#canal.instance.standby.position = 
#canal.instance.standby.timestamp = 
# username/password
canal.instance.dbUsername=canal
canal.instance.dbPassword=canal
canal.instance.defaultDatabaseName=tutorials
canal.instance.connectionCharset=UTF-8
# table regex
canal.instance.filter.regex=.*\\..*
# table black regex
canal.instance.filter.black.regex=
#################################################
```

address设置为mysql的连接地址，defaultDatabaseName设置为自己要监听的库名，这里是tutorial

在MySQL命令行，创建一个新用户，作为slave，这个用户对应配置文件里的dbUsername

``` sql
CREATE USER canal IDENTIFIED BY 'canal!QAZ2wsx';  
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
-- GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ;
FLUSH PRIVILEGES;
```

启动

``` shell
sh bin/startup.sh
```

启动后可以在logs目录下查看日志

### 编写Canal客户端程序

新建一个java maven项目，pom.xml里添加依赖

``` xml
<dependency>
    <groupId>com.alibaba.otter</groupId>
    <artifactId>canal.client</artifactId>
    <version>1.0.25</version>
</dependency>
```

``` java
import com.alibaba.otter.canal.client.CanalConnector;
import com.alibaba.otter.canal.client.CanalConnectors;
import com.alibaba.otter.canal.protocol.CanalEntry;
import com.alibaba.otter.canal.protocol.Message;

import java.net.InetSocketAddress;
import java.util.List;

/**
 * @author lvshuo
 * @create 2018/5/17
 * @since 1.0.0
 */
public class MainApp {

    public static void main(String[] args) throws Exception {
        CanalConnector connector = CanalConnectors.newSingleConnector(
                new InetSocketAddress("127.0.0.1", 11111), "example", "", "");

        int batchSize = 1000;
        int emptyCount = 0;
        try {
            connector.connect();
            connector.subscribe(".*\\..*");
            connector.rollback();
            int totalEmptyCount = 120;
            while (emptyCount < totalEmptyCount) {
                Message message = connector.getWithoutAck(batchSize); // 获取指定数量的数据
                long batchId = message.getId();
                int size = message.getEntries().size();
                if (batchId == -1 || size == 0) {
                    emptyCount++;
                    System.out.println("empty count : " + emptyCount);
                    try {
                        Thread.sleep(10000);
                    } catch (InterruptedException e) {
                    }
                } else {
                    emptyCount = 0;
                    // System.out.printf("message[batchId=%s,size=%s] \n", batchId, size);
                    printEntry(message.getEntries ());
                }

                connector.ack(batchId); // 提交确认
                // connector.rollback(batchId); // 处理失败, 回滚数据
            }

            System.out.println("empty too many times, exit");
        } finally {
            connector.disconnect();
        }
    }

    private static void printEntry(List<CanalEntry.Entry> entrys) {
        for (CanalEntry.Entry entry : entrys) {
            if (entry.getEntryType() == CanalEntry.EntryType.TRANSACTIONBEGIN || entry.getEntryType() == CanalEntry
                    .EntryType
                    .TRANSACTIONEND) {
                continue;
            }

            CanalEntry.RowChange rowChage = null;
            try {
                rowChage = CanalEntry.RowChange.parseFrom(entry.getStoreValue());
            } catch (Exception e) {
                throw new RuntimeException("ERROR ## parser of eromanga-event has an error , data:" + entry.toString(),
                        e);
            }

            CanalEntry.EventType eventType = rowChage.getEventType();
            System.out.println(String.format("================> binlog[%s:%s] , name[%s,%s] , eventType : %s",
                    entry.getHeader().getLogfileName(), entry.getHeader().getLogfileOffset(),
                    entry.getHeader().getSchemaName(), entry.getHeader().getTableName(),
                    eventType));

            for (CanalEntry.RowData rowData : rowChage.getRowDatasList()) {
                if (eventType == CanalEntry.EventType.DELETE) {
                    printColumn(rowData.getBeforeColumnsList());
                } else if (eventType == CanalEntry.EventType.INSERT) {
                    printColumn(rowData.getAfterColumnsList());
                } else {
                    System.out.println("-------> before");
                    printColumn(rowData.getBeforeColumnsList());
                    System.out.println("-------> after");
                    printColumn(rowData.getAfterColumnsList());
                }
            }
        }
    }

    private static void printColumn(List<CanalEntry.Column> columns) {
        for (CanalEntry.Column column : columns) {
            System.out.println(column.getName() + " : " + column.getValue() + "    update=" + column.getUpdated());
        }
    }

}
```

### 模拟MySQL的变化

创建一张新表，插入一条数据，然后更新这条数据，最后删除这条数据，查看一下canal客户端是不是抓取到了增量数据：

``` sql
create table test_canal(id int, name varchar(20)) engine=innodb charset=utf8;
insert into test_canal(id, name) values(1, "aaa");
update test_canal set name="hahaha" where id=1;
```

![](http://7xkfga.com1.z0.glb.clouddn.com/1eafd3b96f831f8203826fe1669eb12b.jpg)

可以看到，创建了一个表，然后插入一条新数据，然后更新了这条数据，更新前后的值都可以看到。