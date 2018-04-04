---
title: Hexo command Cheatsheet
date: 2017-06-05 15:30:37
tags:
    - hexo
    - Cheat Sheet
---


## hexo

``` bash
npm install hexo -g #安装  
npm update hexo -g #升级  
hexo init #初始化
```

## 简写

``` bash
hexo n "我的博客"  # == hexo new "我的博客" #新建文章
hexo p           # == hexo publish #发布草稿
hexo g           # == hexo generate#生成
hexo s           # == hexo server #启动服务预览
hexo d           # == hexo deploy#部署
```

## 服务器

``` bash
hexo server #Hexo 会监视文件变动并自动更新，您无须重启服务器。
hexo server -s #静态模式
hexo server -p 5000 #更改端口
hexo server -i 192.168.1.1 #自定义 IP

hexo clean #清除缓存 网页正常情况下可以忽略此条命令
hexo g #生成静态网页
hexo d #开始部署
```

## 监视文件变动

``` bash
hexo generate #使用 Hexo 生成静态文件快速而且简单
hexo generate --watch #监视文件变动
```

## 完成后部署

两个命令的作用是相同的

``` bash
hexo generate --deploy
hexo deploy --generate

hexo deploy -g
hexo server -g
```

## 草稿

``` bash
hexo publish [layout] <title>
```

## 模版

``` bash
hexo new "postName" #新建文章
hexo new page "pageName" #新建页面
hexo generate #生成静态页面至public目录
hexo server #开启预览访问端口（默认端口4000，'ctrl + c'关闭server）
hexo deploy #将.deploy目录部署到GitHub

hexo new [layout] <title>
hexo new photo "My Gallery"
hexo new "Hello World" --lang tw
```

|变量  |	描述 |
|----  |---- |
|layout |	布局|
|title	|标题|
|date	|文件建立日期|
