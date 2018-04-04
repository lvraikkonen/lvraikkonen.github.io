title: 使用Scrapy编写爬虫
date: 2016-09-18 10:39:32
tags:
- Python 
- 爬虫
categories: [Python, 框架]

---

Scrapy是Python开发的一个快速,高层次的屏幕抓取和web抓取框架，用于抓取web站点并从页面中提取结构化的数据。Scrapy用途广泛，可以用于数据挖掘、监测和自动化测试。
![Scrapy Logo](http://7xkfga.com1.z0.glb.clouddn.com/Scrapy_logo.png)

[Scrapy Home Site](https://scrapy.org/)

``` shell
pip install scrapy
```

<!-- more -->

## Scrapy 处理流程图

借个图简单介绍下Scrapy处理的流程(这就是框架，帮我们完成了大部分工作)
![Scrapy workflow](http://7xkfga.com1.z0.glb.clouddn.com/scrapy_architecture.png)

- 引擎(Scrapy Engine)，用来处理整个系统的数据流处理，触发事务。
- 调度器(Scheduler)，用来接受引擎发过来的请求，压入队列中，并在引擎再次请求的时候返回。
- 下载器(Downloader)，用于下载网页内容，并将网页内容返回给蜘蛛。
- 蜘蛛(Spiders)，蜘蛛是主要干活的，用它来制订特定域名或网页的解析规则。编写用于分析response并提取item(即获取到的item)或额外跟进的URL的类。 每个spider负责处理一个特定(或一些)网站。
- 项目管道(Item Pipeline)，负责处理有蜘蛛从网页中抽取的项目，他的主要任务是清晰、验证和存储数据。当页面被蜘蛛解析后，将被发送到项目管道，并经过几个特定的次序处理数据。
- 下载器中间件(Downloader Middlewares)，位于Scrapy引擎和下载器之间的钩子框架，主要是处理Scrapy引擎与下载器之间的请求及响应。
- 蜘蛛中间件(Spider Middlewares)，介于Scrapy引擎和蜘蛛之间的钩子框架，主要工作是处理蜘蛛的响应输入和请求输出。
- 调度中间件(Scheduler Middlewares)，介于Scrapy引擎和调度之间的中间件，从Scrapy引擎发送到调度的请求和响应。


接下来简单介绍一下爬虫的写法，从开发爬虫到部署，把爬取的数据写入mongodb:

1. 创建新的Scrapy工程
2. 定义要抽取的Item对象
3. 编写一个Spider来爬取某个网站并提取出Item对象
4. 编写Item Pipeline来存储提取出来的Item对象
5. 使用Scrapyd-client部署Spider项目

把Aqicn上的北京空气质量的数据爬取下来，为日后分析做准备
![Beijing Air Quality](http://7xkfga.com1.z0.glb.clouddn.com/Beijing_Aqi.png)

## 创建Scrapy工程

在目录下执行命令

``` shell
scrapy startproject projectName
```

创建如下的项目文件夹，目录结构如下
![Folder](http://7xkfga.com1.z0.glb.clouddn.com/ScrapyProjectFolder.png)


## 定义Item对象

创建一个`scrapy.Item`类，将所要爬取的字段定义好

``` python
import scrapy

class AirqualityItem(scrapy.Item):
    date = scrapy.Field()
    hour = scrapy.Field()
    city = scrapy.Field()
    # area = scrapy.Field()
    aqivalue = scrapy.Field()
    aqilevel = scrapy.Field()
    pm2_5 = scrapy.Field()
    pm10 = scrapy.Field()
    co = scrapy.Field()
    no2 = scrapy.Field()
    o3 = scrapy.Field()
    so2 = scrapy.Field()
    temp = scrapy.Field()
    dew = scrapy.Field()
    pressure = scrapy.Field()
    humidity = scrapy.Field()
    wind = scrapy.Field()
    # add field to log spider crawl time
    crawl_time = scrapy.Field()
```

## 编写Spider

Spider类里面定义如何从一个domain组中爬取数据，包括：初始化url列表、如何跟踪url和如何解析页面提取Item，定义一个Spider，需要继承scrapy.Spider类

- name: 定义Spider的名称，以后调用爬虫应用时候使用;
- start_url: 初始化url;
- parse(): 解析下载后的Response对象，解析并返回页面数据并提取出相应的Item对象


### 抽取Item对象内容

Scrapy Selector是Scrapy提供的一套选择器，通过特定的XPath或者CSS表达式来选择HTML文件中某个部分 (note: Chrome浏览器自带的copy XPath或者CSS功能非常好用)，在开发过程中，可以使用Scrapy内置的Scrapy-Shell来debug选择器。

爬虫的代码如下

``` python
class AirQualitySpider(CrawlSpider):
    name = "AqiSpider"
    download_delay = 2
    allowed_domains = ['aqicn.org']
    start_urls = ['http://aqicn.org/city/beijing/en/']

    def parse(self, response):
        sel = Selector(response)
        pm25 = int(sel.xpath('//*[@id="cur_pm25"]/text()').extract()[0])
        pm10 = int(sel.xpath('//*[@id="cur_pm10"]/text()').extract()[0])
        o3 = int(sel.xpath('//*[@id="cur_o3"]/text()').extract()[0])
        no2 = int(sel.xpath('//*[@id="cur_no2"]/text()').extract()[0])
        so2 = int(sel.xpath('//*[@id="cur_so2"]/text()').extract()[0])
        co = int(sel.xpath('//*[@id="cur_co"]/text()').extract()[0])
        item = AirqualityItem()
        item['date'] = updatetime.strftime("%Y%m%d")
        item['hour'] = updatetime.hour # strftime("%H%M%S")
        item['city'] = city
        item['aqivalue'] = aqivalue
        item['aqilevel'] = aqilevel
        item['pm2_5'] = pm25
        item['pm10'] = pm10
        item['co'] = co
        item['no2'] = no2
        item['o3'] = o3
        item['so2'] = so2
        item['temp'] = temp
        item['dew'] = dew
        item['pressure'] = pressure
        item['humidity'] = humidity
        item['wind'] = wind
        item['crawl_time'] = cur_time
        yield item
```

## 数据存储到MongoDB

在Item已经被爬虫抓取之后，Item被发送到Item Pipeline去做更复杂的处理，比如存储到文件中或者数据库中。Item Pipeline常见的用途如下

- 清洗抓取来的HTML数据
- 验证抓取来的数据
- 查询与去除重复数据
- 将Item存储到数据库中


``` python
class AirqualityPipeline(object):
    def __init__(self):
        connection = pymongo.MongoClient(settings['MONGODB_SERVER'], settings['MONGODB_PORT'])
        db = connection[settings['MONGODB_DB']]
        self.collection = db[settings['MONGODB_COLLECTION']]

    def process_item(self, item, spider):
        # save data into mongodb
        valid = True
        if not item:
            valid = False
            raise DropItem("Missing {0}".format(item))
        if valid:
            self.collection.insert(dict(item))
            log.msg("an aqi data added to MongoDB database!", level=log.DEBUG, spider=spider)
        return item
```

接下来需要在setting.py文件中配置Item Pipeline与数据库信息

``` shell
ITEM_PIPELINES = {
   'AirQuality.pipelines.AirqualityPipeline': 300,
}

MONGODB_SERVER = "localhost"
MONGODB_PORT = 27017
MONGODB_DB = "aqihistoricaldata"
MONGODB_COLLECTION = "aqidata"
```

到此，简单的爬虫就已经写好了，可以使用以下命令来抓取相关页面来测试一下这个爬虫

``` shell
scrapy crawl AqiSpider
```

其中，AqiSpider就是在Spider程序中设置的Spider的name属性

## 防止爬虫被禁的几种方法

很多网站都有反爬虫的机制，对于这些网站，可以采用以下的一些办法来绕开反爬虫机制：

1. 使用User Agent池，每次发送请求的时候从池中选取不一样的浏览器头信息
2. 禁止Cookie，有些网站会根据Cookie识别用户身份
3. 设置dowload_delay，频繁请求数据肯定会被禁


## 使用Scrapyd和Scrapyd-client部署爬虫

scrapyd是一个用于部署和运行scrapy爬虫的程序，它允许你通过JSON API来部署爬虫项目和控制爬虫运行。crapyd是一个守护进程，监听爬虫的运行和请求，然后启动进程来执行它们。

### 安装

``` shell
pip install scrapyd
pip install scrapyd-client
```

### 启动服务

``` shell
scrapyd
```

### 配置服务器信息

编辑scrapy.cfg文件，添加如下内容

``` shell
[deploy:MySpider]
url = http://localhost:6800/
project = AirQuality
```

其中，MySpider是服务器名称， url是服务器地址

检查配置，列出当前可用的服务器

``` shell
scrapyd-deploy -l
```

部署Spider项目

``` shell
scrapyd-deploy MySpider -p AirQuality
```

部署完成后，在http://localhost:6800 可以看到如下的爬虫部署的监控信息
![schedule](http://7xkfga.com1.z0.glb.clouddn.com/scrapyd.png)

可以使用curl命令去调用爬虫，也可以使用contab命令来定时的去调用爬虫，来实现定时爬取的任务。

版权声明：<br>
<hr>
除非注明，本博文章均为原创，转载请以链接形式标明本文地址。<br>
