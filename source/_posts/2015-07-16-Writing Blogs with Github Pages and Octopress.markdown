---
layout: post
title: "使用Github与Octopress写博客"
date: 2015-07-16 10:06:06 +0800
comments: true
tag: 
- Octopress
categories: 
- 备忘
---

## Octopress和Github Pages是什么？
![Octopress_logo](http://7xkfga.com1.z0.glb.clouddn.com/octopress_logo.jpg)

- Octopress是一个基于Ruby语言的开源静态网站框架
- Github Pages是Github上的一项服务， 注册用户可以申请一个和自己账号关联的二级域名， 在上面可以托管一个静态网站，网站内容本身就是Github的一个repository也就是项目， 维护这个项目的代码就是在维护自己的网站。简单来说就是 yourname.github.io/

使用Octopress搭建博客，然后使用Github托管，有以下几个原因：

1. 免费
2. 版本控制，可以使用git实现写文章、建网站时候修改的版本控制
3. Octopress容易上手，并且这个风格正是我喜欢的，尤其对于一个理工男
4. 使用Markdown，markdown是世界上最流行的轻量级标记语言
5. 很酷，能装逼，当然这不是重点

<!--more-->

## 搭建Octopress博客系统
**Note:** *在这儿写的是关于在Mac上安装Octopress博客系统，和windows有细微的差别，不过个人觉得还是用Mac，无论写代码还是做黑客都更专业。*

### 安装基本工具
**git**

对于Mac来说，安装XCode之后，自带了git，可以使用下面命令检查本机的git版本

``` bash
$ git version
```

**Ruby**

Mac本身自带Ruby，但是也许版本过低，在这儿多说一句：有时候在低版本ruby下搭建好的Octopress，莫名其妙不好用了，原因也许就是Mac升级之后，ruby也升级了，要注意一下

至于Mac下如何使用Homebrew安装，请查看~~其他文章~~。

``` bash
$ brew install ruby
$ ruby --version
```

Ruby版本在1.9.3以上就可以了，就可以使用`gem`来安装Ruby的包了

**PS.** *gem在Ruby中，相当于Python中的`pip`*

由于我们生活在一个伟大的国家，so在下一步安装前，先更改一下`gem`的更新源，改为淘宝的源

``` bash
gem sources -a http://ruby.taobao.org/
gem sources -r http://rubygems.org/
gem sources -l
```
三行命令的作用分别是：添加淘宝源；删除默认源；显示当前源列表。显示淘宝地址就表示成功。

安装bundle和bundler，

``` bash
gem install bundle
gem install bundler
```

**Note:** 安装配置完新版本的Ruby后，一定要重新安装bundle和bundler，否则bundle仍会bundler指向旧版本的Ruby，PS. 由于手贱，把MAC升级到最新系统了，结果各种奇妙的事情就发生了，不过处理方法一般都是：安装最新版本的Ruby，然后再安装`bundler`和`bundler`

**Octopress**

这个就是我们要使用的框架，它是基于Jekyll的一个静态博客生成框架，Jekyll是一个静态网站生成框架，它有很多功能，也可以直接使用，但是就麻烦得多，很多东西要配置和从头写。

``` bash
git clone git://github.com/imathis/octopress.git octopress
cd octopress
bundle install
rake install
```

**创建Github账号和Github Pages**

大多数人都已经有了Github帐号了，访问[Github](http://github.com/ "Github")来注册帐号，然后访问Github Pages来创建博客空间，唯一需要注意的是Repo必须是Github帐号.github.io，否则不会起作用。 然后运行：

``` bash
rake setup_github_pages
```
输入Github Page的Repo的地址，例如：git@github.com:username/username.github.io.git，就可以了

### 测试一下

输入命令生成页面

``` bash
rake generate
```

生成完毕后，使用以下命令启动网站进程，默认占用4000端口， preview一下

``` bash
rake preview
```

可以使用 [http://localhost:4000](http://localhost:4000) 访问你的博客页面了

### 配置博客

配置文件是根目录下的 _config.yml文件，使用vim或者其他文本编辑器编辑它吧

``` bash
# ----------------------- #
#      Main Configs       #
# ----------------------- #

url: http://lvraikkonen.github.io
title: My Data Science Path
subtitle: Shut up, just coding
author: Claus Lv
simple_search: https://www.google.com/search
description:
```

### 语法高亮

例子：

``` javascript
alert("欢迎")
```

``` python
import math
print "Hello World"
lst = range(100)
print lst.map(lambda x: x**2)
```

``` html
	<!-- mathjax config similar to math.stackexchange -->
	<script type="text/x-mathjax-config">
		MathJax.Hub.Config({
  		jax: ["input/TeX", "output/HTML-CSS"],
  		tex2jax: {
    		inlineMath: [ ['$', '$'] ],
    		displayMath: [ ['$$', '$$']],
    		processEscapes: true,
    		skipTags: ['script', 'noscript', 'style', 'textarea', 'pre', 'code']
  		},
  		messageStyle: "none",
  		"HTML-CSS": { preferredFont: "TeX", availableFonts: ["STIX","TeX"] }
		});
	</script>
	<script src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS_HTML" type="text/javascript"></script>
```

### 添加社交分享

Octopress默认是带有社交分享功能的，比如Twitter, Facebook, Google Plus等，但这些全世界都通用的东西在我大天朝就是不好使。

网站页的分享有很多第三方的库，这里用[jiathis](http://www.jiathis.com/)

1. 在`_config.yml`中加入social_share: true

2. 修改/source/_includes/post/sharing.html

3. 访问[jiathis](http://www.jiathis.com/)获取分享的代码，放入新建的/source/_includes/post/social_media.html

### 添加博客评论

Octopress也默认集成有评论系统Disqus，这个是国外最大的第三方评论平台，世界都在用，除了我大天朝。这里使用[多说](http://www.duoshuo.com)

1. 到多说注册，获取用户名，也就是在多说上添的youname.duoshuo.com中的yourname
2. 在`_config.yml`中添加

   ```
   duoshuo_comments: true
   duoshuo_short_name: yourname
   ```
3. 在/source/_layouts/post.html中把评论模版添加到网页中 
 
4. 创建/source/_includes/post/duoshuo.html，将上步获取的HTML代码放进去

      
### 国内访问加速

**jQuery源**

修改jQuery的源 打开source/_includes/head.html，找到如下

``` html
    <script src="//ajax.googleapis.com/ajax/libs/jquery/1.9.1/jquery.min.js"></script>
```
改为：

``` html
    <script src="//ajax.aspnetcdn.com/ajax/jQuery/jquery-1.9.1.min.js"></script>
```

**字体源**

Octopress的英文字体是加载的Google Fonts，我们将其改成国内的CDN源， 打开source/_includes/custom/head.html, 将其中的`https://fonts.googleapis.com`改为`http://fonts.useso.com`


**Twitter Facebook Google+关闭**

在前面提到的`_config.yml`中相关的例如twitter_tweet_butto改为`false`

### 写博客

要发布一篇新文章，在命令行中输入以下命令：

``` bash
rake new_post["postName"]
```

之后在/source/_post/里面就有该博文的markdown文件了，使用Markdown文本编辑器写博客吧

- rake generate 生成静态的博客文件，生成的文件在_deploy中
- rake preview 在本地预览博客，这与发布到Github Pages后的效果是一样的
- rake deploy 这是最后一步，就是把Octopress生成的文件（在_deploy）发布到Github上面去。这里的实际是Octopress根据你的配置用sources中的模板，生成网页（HTML，JavaScript, CSS和资源），再把这些资源推送到yourname.github.io这个Repo中去，然后访问https://*yourname*.github.io 就能看到你的博客了

### 发布

执行命令

``` bash
$ rake generate
$ rake deploy
```

第一行命令用来生成页面，第二行命令用来部署页面，上述内容完成，就可以访问 http://[your_username].github.io/看博客了

**Note**: octopress 根目录为`source`分支， `_deploy`目录下为`master`分支，`rake deploy`时候会把`_deploy`下的内容发布到github上的`master`分支。别忘了把源文件（包括配置等）发布到`source`分支下

push时候可用

``` bash
$ git status
```

查看状态

执行以下命令，将源文件发布到Github的`source`分支

``` bash
git add .
git commit -m "备注内容"
git push origin source
```

如果遇到类似`error: failed to push some refs to`的错误，参考 [stackoverflow](http://stackoverflow.com/questions/24114676/git-error-failed-to-push-some-refs-to)解决
