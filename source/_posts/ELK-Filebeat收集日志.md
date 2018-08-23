---
title: ELK+Filebeat收集日志
date: 2018-08-22 16:45:44
tags:
- Elasticsearch
- ELK
---

ELK(Elasticsearch + Logstash + Kibana) 简单来说，可以完成对于海量日志数据的汇总、搜索、查询以及可视化，可以快速定位和分析问题。

- 通过 Logstash 我们可以把各种日志进行转换后存储到 elasticsearch 中
- 通过 Elasticsearch 可以非常灵活的存储和索引日志，并且elasticsearch 提供了丰富的 HTTP REST API 接口来对数据进行增删查等操作
- 通过 Kibana 我们可以对存储在 elasticsearch 中的日志以图形化的形式进行展现，并且提供非常丰富的过滤接口，让用户能够通过过滤快速定位问题

在日常日志处理中，也常用Beats工具，可以作为轻量级的数据收集Agent

![Beats](https://www.elastic.co/guide/en/beats/libbeat/current/images/beats-platform.png)

<!-- more -->

与 Logstash 不同，Beats 只是 data shipper。Beats 家族共享 libbeat 这个库，每个产品分别实现对不同数据来源的收集。目前官方实现有：

- Filebeat —— 文件
- Metricbeat —— 系统及应用指标
- Packetbeat —— 网络抓包分析，如 SQL, DNS
- Winlogbeat —— Windows 系统日志
- Auditbeat —— 审计数据
- Heartbeat —— ICMP, TCP, HTTP 监控

## 日志收集框架

![](http://7xkfga.com1.z0.glb.clouddn.com/33eebbae76933312b00db18909209bbd.png)

日志数据流如下:

应用程序将日志存储在服务器上，部署在每台服务器上的FileBeat负责收集日志，然后将日志发送给Kafka进行消息缓冲，同时也可以支持流失处理(流处理应用可以从消息的offset初始的地方来读取)；然后通过Logstash将日志进行处理之后，比如解析等处理，将处理后的对象传递给ElasticSearch，进行落地并进行索引处理；最后通过Kibana来提供web界面，来查看日志等。

下面以Log4net配置的应用Log为例。

## 样例日志数据

针对两种不同类型的应用日志，分别使用两种pattern，也分别记录到Debug.lg和Error.log文件中。

- 日志类型为Debug,Info,Warn的日志，使用[%date] [%thread] %-5level Log4NetTest %logger %method [%message%exception]%n模式，分别记录下时间，线程，日志等级，应用名称，日志记录类属性，日志记录方法，日志信息和异常
- 日志类型为Error和Fatal的日志，使用[%date] [%thread] %-5level Log4NetTest %l [%message%n%exception]%n，分别是时间，线程，日志等级，应用名称，出错位置（包含具体文件，以及所在行，需要PDB文件才能到行），日志信息和异常

## 配置Filebeat

Filebeat隶属于Beats，Filebeat占用资源极少，简单配置`filebeat.yml`文件就可以使用

```
filebeat.prospectors:

- type: log
  # Change to true to enable this prospector configuration.
  enabled: true
  # Paths that should be crawled and fetched. Glob based paths.
  paths:
    - /Users/lvshuo/bigdata/logs/*.Error.log
  
  multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2},[0-9]{3}'
  multiline.negate: true
  multiline.match: after
  
  fields:
    log_topics: errorlog

- type: log
  enabled: false
  paths:
    - /Users/lvshuo/bigdata/logs/*.Info.log
  
  multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2},[0-9]{3}'
  multiline.negate: true
  multiline.match: after

  fields:
    log_topics: infolog

#--------------- Logstash output ----------------------
output.logstash:
  hosts: ["localhost:5044"]

  index: '{[fields][log_topics]}'
```

__note:__ 在6.0版本以后，document_type类型就不被支持了。为了分类处理INFO和ERROR日志，并写入不同的index中，在filebeat的配置里，加入`fields`的自定义字段，然后通过%{[]}获取对应的值 ，如下：

    if [fields][fieldname] == "string"

为了以后不再采坑，建议参考安装包中自带的样例配置文件 `filebeat.reference.yml`

上面将收集到的Log数据传递给Logstash(为了演示，直接将Beats输出到Logstash)

## 配置Logstash

```
#指定logstash监听filebeat的端口
input {
    beats {
        port => "5044"
    }
}
# The filter part of this file is commented out to indicate that it is
# optional.
filter {
    grok {
            match => {
             "message" => "%{TIMESTAMP_ISO8601:datetime} \[%{SYSLOGPROG:appName}\] %{LOGLEVEL:level} \s*(?<traceback>([\s\S]*))"
             } 
            overwrite => ["message"]
        }
    date {
        match => ["datetime", "yyyy-MM-dd HH:mm:ss,SSS", "UNIX"]
        target => "@timestamp"
        locale => "cn"
    }
    
}
output {
    if [fields][log_topics] == "errorlog" {
        # file {
        #     path => "/Users/lvshuo/Desktop/tmp/log/errorLog_%{+YYYYMMdd}_%{+HH}.log"
        #     codec => line { format => "%{message}"}
        # }
        stdout { codec => rubydebug }
        elasticsearch { 
            hosts => ["localhost:9200"]
            index => "logstash-%{[fields][log_topics]}-%{+YYYY.MM.dd}"
        }
    }
    
}
```

`Grok` 是 Logstash 最重要的插件，可以在 grok 里预定义好命名正则表达式

下面是一个很好用的Grok Debugger，可以调试grok表达式：

http://grokdebug.herokuapp.com/

### Grok过滤器插件

Grok内置了120多种的正则表达式库，地址:https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns

比如,下面这条日志：

```
55.3.244.1 GET /index.html 15824 0.043
```

这条日志可切分为5个部分，IP(55.3.244.1)、方法(GET)、请求文件路径(/index.html)、字节数(15824)、访问时长(0.043),对这条日志的解析模式(正则表达式匹配)如下:

```
%{IP:client} %{WORD:method} %{URIPATHPARAM:request} %{NUMBER:bytes} %{NUMBER:duration}
```

写到filter中:

```
filter {
    grok {
        match => { "message" => "%{IP:client} %{WORD:method} %{URIPATHPARAM:request} %{NUMBER:bytes} %{NUMBER:duration}"}
    }
}
```

解析后:

```
client: 55.3.244.1
method: GET
request: /index.html
bytes: 15824
duration: 0.043
```

如果内置的正则表达式不能满足要求，可以按下面步骤解析任意格式日志：

1. 先确定日志的切分原则，也就是一条日志切分成几个部分。
2. 对每一块进行分析，如果Grok中正则满足需求，直接拿来用。如果Grok中没用现成的，采用自定义模式。
3. 学会在Grok Debugger中调试。

下面两条日志:

```
2017-03-07 00:03:44,373 4191949560 [CASFilter.java:330:DEBUG]  entering doFilter()
2017-03-16 00:00:01,641 133383049 [UploadFileModel.java:234:INFO]  上报内容准备写入文件
```

切分原则:

```
2017-03-16 00:00:01,641:时间
133383049：编号
UploadFileModel.java:java类名
234:代码行号
INFO：日志级别
entering doFilter()：日志内容
```

前五个字段用Grok中已有的，分别是TIMESTAMP_ISO8601、NUMBER、JAVAFILE、NUMBER、LOGLEVEL，最后一个采用自定义正则的形式，日志级别的]之后的内容不论是中英文，都作为日志信息处理，最后一个字段的内容用info表示，正则如下:

(?<info>([\s\S]*))

上面两条日志对应的完整的正则如下，其中\s*用于剔除空格。

```
%{TIMESTAMP_ISO8601:time} %{NUMBER:num} \[\s*%{JAVAFILE:class}\s*\:\s*%{NUMBER:lineNumber}\s*\:%{LOGLEVEL:level}\s*\] (?<info>([\s\S]*))
```

http://grokdebug.herokuapp.com/

### 时间处理插件Date

date 插件可以用来转换你的日志记录中的时间字符串，变成 LogStash::Timestamp 对象，然后转存到 @timestamp 字段里

```
filter {
    grok {
        match => ["message", "%{HTTPDATE:logdate}"]
    }
    date {
        match => ["logdate", "dd/MMM/yyyy:HH:mm:ss Z"]
    }
}
```

### 数据修改插件(Mutate)

mutate 插件是 Logstash 另一个重要插件。它提供了丰富的基础类型数据处理能力。包括类型转换，字符串处理和字段处理等。

```
filter {
    mutate {
        convert => ["request_time", "float"]
    }
}
```

### GeoIP 地址查询归类

GeoIP 库可以根据 IP 地址提供对应的地域信息，包括国别，省市，经纬度等，对于可视化地图和区域统计非常有用。


## 启动

首先启动log所在机器上部署的Filebeat agent

mac:
```
sudo chown root filebeat.yml 
sudo ./filebeat -e -c filebeat.yml -d "publish"
```

windows:
```
PS C:\Program Files\Filebeat> Start-Service filebeat
```

然后启动Logstash

``` shell
bin/logstash -f first-pipeline.conf --config.test_and_exit 测试配置文件 
bin/logstash -f first-pipeline.conf --config.reload.automatic 自动加载配置文件的修改
```

可以在Kibana中看到

![](http://7xkfga.com1.z0.glb.clouddn.com/70053f47dfddf0541465bf90767091ce.jpg)
