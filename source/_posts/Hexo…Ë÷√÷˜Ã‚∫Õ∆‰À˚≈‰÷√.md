---
title: Hexo设置主题和其他配置
date: 2017-06-05 14:59:24
tags:
  - hexo
---

博客的主题很多，我选择一个非常流行的主题 [NexT](http://theme-next.iissnan.com/)，作为我博客的主题，这个主题简约，并且文档和维护都很好。

下面记录一下我的Hexo站点的配置，以及NexT主题的配置，以备日后查找

## 站点配置

博客根目录下的`_config.yml` 文件是站点的配置文件。


为了能够使Hexo部署到GitHub上，需要安装一个插件：

``` shell
npm install hexo-deployer-git --save
```

设置站点配置文件指定部署的位置

```
deploy:
  type: git
  repo: https://github.com/lvraikkonen/lvraikkonen.github.io.git
  branch: master
```

在 `站点配置文件` 中找到theme字段，把值改为 `next`

<!-- more -->

## 主题NexT配置

### 选择主题样式

Scheme 是 NexT 提供的一种特性，借助于 Scheme，NexT 为你提供多种不同的外观。同时，几乎所有的配置都可以 在 Scheme 之间共用。目前 NexT 支持三种 Scheme，他们是：

- Muse - 默认 Scheme，这是 NexT 最初的版本，黑白主调，大量留白
- Mist - Muse 的紧凑版本，整洁有序的单栏外观
- Pisces - 双栏 Scheme，小家碧玉似的清新 (我的)

``` yml
#scheme: Muse
#scheme: Mist
scheme: Pisces
```

### 头像设置

编辑 `主题配置文件`， 修改字段 avatar， 值设置成头像的链接地址。其中，头像的链接地址可以是：

|地址	 |值 |
|----| --- |
|完整的互联网URI	 |http://example.com/avatar.png |
|站内地址 | - 将头像放置主题目录下的 source/uploads/ 或者 放置在 source/images/ 目录下 |

配置为：avatar: /images/avatar.png

### 添加标签页和分类页

``` shell
hexo new page "tags"
hexo new page "categories"
```

同时，在/source目录下会生成一个tags文件夹和categories文件夹，里面各包含一个index.md文件。

修改/source/tags目录下的 `index.md`文件

``` markdown
title: tags
date: 2017-05-29 18:16:02
type: "tags"

---
```

修改/source/categories目录下的 `index.md`文件

``` markdown
title: categories
date: 2015-09-29 18:17:14
type: "categories"

---
```

修改 `主题配置文件`， 去掉相应的注释

``` yml
menu:
  home: /                       #主页
  categories: /categories	#分类页（需手动创建）
  #about: /about		  #关于页面（需手动创建）
  archives: /archives		#归档页
  tags: /tags			#标签页（需手动创建）
  #commonweal: /404.html        #公益 404 （需手动创建）

```

### 设置网站的图标Favicon

从网上找一张 icon 图标文件，放在 source 目录下就可以了

### 添加友情链接

在 `站点配置文件` 中添加参数：

``` yml
links_title: 友情链接
links:
    #百度: http://www.baidu.com/
    #新浪: http://example.com/
```

### 设置代码高亮

NexT 使用 Tomorrow Theme 作为代码高亮，共有5款主题供你选择。 NexT 默认使用的是 白色的 normal 主题，可选的值有 `normal`，`night`， `night blue`， `night bright`， `night eighties`

更改 highlight_theme 字段，将其值设定成你所喜爱的高亮主题

### 配置Algolia 搜索

[官网](https://www.algolia.com/) 注册一个账号，可以用Github账户注册

![AlgoliaSinup](http://7xkfga.com1.z0.glb.clouddn.com/Algolia_Signup.png)

登录进入Dashboard控制台页面，创建一个新Index

![AlgoliaCreateIndex](http://7xkfga.com1.z0.glb.clouddn.com/AlgoliaCreateIndex.png)

进入 API Keys 界面，拷贝 `Application ID` 、`Search-Only API Key` 和 `Admin API Key`

编辑 `站点配置文件` ，新增以下配置：

``` yml
algolia:
  applicationID: 'your applicationID'
  apiKey: 'your Search-Only API Key'
  adminApiKey: 'your Admin API Key'
  indexName: 'your newcreated indexName'
  chunkSize: 5000
```

安装Hexo Algolia
在Hexo根目录执行如下指令，进行Hexo Algolia的安装：

``` shell
npm install --save hexo-algolia
```

到Hexo的根目录，在其中找到package.json文件，修改其中的hexo-algolia属性值为^0.2.0，如下图所示：

![package.json](http://7xkfga.com1.z0.glb.clouddn.com/hexo_algolia_version.png)

当配置完成，在站点根目录下执行hexo algolia 来更新Index

``` shell
hexo algolia
```

*注意：*

> 如果发现没有上传数据，这时候可以先 `hexo clean` 然后再 `hexo algolia` _

在 `主题配置文件`中，找到Algolia Search 配置部分：

``` yml
# Algolia Search
algolia_search:
  enable: true
  hits:
    per_page: 10
  labels:
    input_placeholder: Search for Posts
    hits_empty: "We didn't find any results for the search: ${query}"
    hits_stats: "${hits} results found in ${time} ms"
```

将 `enable` 改为true 即可，根据需要你可以调整labels 中的文本。

### 配置来必力评论

评论插件，最出名的是 Disqus，但是对于国内用户来说，自带梯子好些。改用多说，路边社消息，多说好像要完蛋了。发现了个叫 LiveRe（来必力）的评论插件，韩国出的，用着感觉还不错。

[来必力官网](https://livere.com/)注册账号

LiveRe 有两个版本：

- City 版：是一款适合所有人使用的免费版本；
- Premium 版：是一款能够帮助企业实现自动化管理的多功能收费版本。

选择City版就可以了

![installLiveRe](http://7xkfga.com1.z0.glb.clouddn.com/installLiveRe.png)

获取 LiveRe UID。 编辑 `主题配置文件`， 编辑 livere_uid 字段，设置如下：

``` yml
livere_uid: #your livere_uid
```

NexT 已经支持来必力，这样就能在页面显示评论了

### 添加阅读次数统计

这里使用 [LeanCloud](https://leancloud.cn/) 为文章添加统计功能

注册账号登陆以后获得 `AppID`以及 `AppKey`这两个参数即可正常使用文章阅读量统计的功能了。

创建应用

![CreateApp](http://7xkfga.com1.z0.glb.clouddn.com/leancloudCreateApp.png)

配置应用

![configApp](http://7xkfga.com1.z0.glb.clouddn.com/configApp.png)

在应用的数据配置界面，左侧下划线开头的都是系统预定义好的表。在弹出的选项中选择创建Class来新建Class用来专门保存我们博客的文章访问量等数据:
为了保证我们前面对NexT主题的修改兼容，此处的新建Class名字为 `Counter`

![GetAppKey](http://7xkfga.com1.z0.glb.clouddn.com/leancloudGetAPIKey.png)

复制 `AppID`以及 `AppKey` 修改 `主题配置文件`

``` yml
# You can visit https://leancloud.cn get AppID and AppKey.
leancloud_visitors:
  enable: true
  app_id: #<app_id>
  app_key: #<app_key>
```

重新生成并部署博客就可以显示阅读量了

_note: 记录文章访问量的唯一标识符是 `文章的发布日期`以及`文章的标题`，因此请确保这两个数值组合的唯一性，如果你更改了这两个数值，会造成文章阅读数值的清零重计_

后台可以看到，刚才创建的Counter类

![backendReadCount](http://7xkfga.com1.z0.glb.clouddn.com/leanCloudbackend_data.png)


### 添加字数统计功能

首先在博客目录下使用 npm 安装插件

``` shell
npm install hexo-wordcount --save
```

在 `主题配置文件`中打开wordcount 统计功能

``` yml
# Post wordcount display settings
# Dependencies: https://github.com/willin/hexo-wordcount
post_wordcount:
  item_text: true
  wordcount: true
  min2read: false
```

找到..\themes\next\layout\_macro\post.swig 文件，将"字"、"分钟" 字样添加到如下位置

```
<span title="{{ __('post.wordcount') }}">
  {{ wordcount(post.content) }} 字
</span>
 ...
<span title="{{ __('post.min2read') }}">
  {{ min2read(post.content) }} 分钟
</span>
```