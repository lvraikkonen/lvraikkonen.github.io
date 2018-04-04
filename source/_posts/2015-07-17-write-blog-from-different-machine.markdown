---
layout: post
title: "在不同的机器上写博客"
date: 2015-07-17 16:48:40 +0800
comments: true
tag: Octopress
categories: 备忘
---


在家的Mac上配置好了Octopress，上班到公司还是要面对大Windows，这时候，想写一篇博客记录一下遇到的问题，怎么办？

## Octopress原理

Octopress的git仓库(repository)有两个分支，分别是`master`和`source`。`master`存储的是博客网站本身（html静态页面），而`source`存储的是生成博客的源文件（包括配置等）。`master`的内容放在根目录的`_deploy`文件夹内，当你`push`源文件时会忽略，它使用的是`rake deploy`命令来更新的。

下面开始在一台新机器上搞

<!--more-->

## 创建一个本地Octopress仓库

将博客的源文件，也就是`source`分支clone到本地的Octopress文件夹内

``` bash
$ git clone -b source git@github.com:username/username.github.com.git octopress
```

然后将博客文件也就是`master`分支clone到Octopress文件夹的`_deploy`文件夹内

``` bash
$ cd octopress
$ git clone git@github.com:username/username.github.com.git _deploy
```

然后安装博客

``` bash
$ gem install bundler
$ bundle install
$ rake setup_github_pages
```

OK了

## 继续写博客

当你要在一台电脑写博客或做更改时，首先更新`source`仓库

``` bash
$ cd octopress
$ git pull origin source  # update the local source branch
$ cd ./_deploy
$ git pull origin master  # update the local master branch
```

**写完博客之后不要忘了push，下面的步骤在每次更改之后都必须做一遍。**

``` bash
$ rake generate
$ git add .
$ git commit -am "Some comment here." 
$ git push origin source  # update the remote source branch 
$ rake deploy             # update the remote master branch
```