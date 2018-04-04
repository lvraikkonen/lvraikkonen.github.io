---
title: 利用Github分支备份Hexo源文件
date: 2016-05-31 10:25:41
tags:
    - 备忘
    - hexo
---


Hexo是一款基于Node.js的静态博客框架，前一阵俺的笔记本泡水直接退役，但是博客的原文件还在那台死去的机器上，所以备份啊。。。
本质上，Hexo是将本地的md文件编译成静态文件上传到github上（或者其他），所以建议是将本地的整个Hexo项目（blog）原件同步提交到github或者其他代码托管的站点。

> `hexo deploy` 生成的静态网址默认是保存在your_blog_name/your_blog_name.github.io的`master`分支，可以在原有的基础上增加一个`hexo_source`分支保存网址原始数据，并将这个分支设置为默认分支，这样每次恢复和迁移文件时候只需要git clone即可获取迁移的文件了。

下面记录一下备份、以及在另外的电脑上恢复博客的过程，为了以后备查。

## 备份过程

1. 本地克隆github.io的远程仓库

``` shell
git clone https://github.com/your_blog_name/your_blog_name.github.io.git
```

2. 创建新的远程分支，用来备份hexo源文件

``` shell
git checkout -b hexo_source
git push origin hexo_source:hexo_source
```

> `git checkout -b hexo_source` hexo_source,并切换到该分支,等同于`git branch hexo_source`; `git checkout hexo_source`;
> 
> `git push origin hexo_source:hexo_source` 提交本地新建分支hexo_source到远程服务器hexo_source分支(origin是默认远程主机名);
> 
> `git push origin :hexo_source`或者 `git push origin --delete hexo_source` 删除远程分支hexo_source;
> 
> `git branch` 可以查看当前分支；`git branch -a` 可以查看所有分支(包括远程分支)；

3. 创建忽略规则文件 `.gitignore`

``` shell
vi .gitignore
```

按需添加如下内容：

```
.DS_Store
Thumbs.db
db.json  
*.log
.deploy*/
node_modules/
.npmignore
public/
```

上面最后一行 public 目录，因其已被 hexo 插件同步到 master 分支里，因此不需要再同步，deploy 是 hexo 的 git 配置存放目录，也不需要同步。其他内容可选择忽略也可以选择同步。

> 建议把Hexo博客目录下_config.yml文件复制粘贴一份，并重命名为hexo_config.yml；把themes目录下你用到主题目录下的_config.yml文件也复制一份，并粘贴到博客根目录，并命名为theme_config.yml

4. 添加内容到仓库并提交到远程仓库

``` shell
git add .
git commit -m "first commit"
git remote add origin git@github.com/your_blog_name/your_blog_name.github.io.git	# 后面仓库目录改成自己新建的。
git push -u origin hexo_source
```

按照以上的步骤就进行了 hexo 源文件的初次备份。
以后每次修改了内容之后，都可通过以下几条命令实现同步。

``` shell
git add .
git commit -m "..."	 # 双引号内填写更新内容
git push origin hexo_source	# 或者 git push
```

## 通过 git submodule 来同步第三方主题

我们一般会选择第三方主题的仓库直接git clone下来。这是一个非常不好的习惯，正确做法是：Fork该第三方主题仓库，这样就会在自己账号下生成一个同名的仓库，并对应一个url，我们应该git clone自己账号下的url。

这样做的原因是：我们很有可能在原来主题基础上做一些自定义的小改动，为了保持多终端的同步，我们需要将这些改动提交到远程仓库。而第三方仓库我们是无法直接push的。

这样就会出现git仓库的嵌套问题，我们通过git submodule来解决这个问题。

``` shell
git submodule add git@github.com:lvraikkonen/hexo-theme-next.git themes/next
```

我们修改主题后:

``` shell
git commit -am "refine themes"
git push origin hexo_source
```

然后就完成了第三方主题的备份

在其他电脑同步源文件时，需要执行如下命令来同步主题

``` shell
git submodule init // 这句很重要
git submodule update
```

## 新机器同步

现在需要在一台新的电脑上写博客，首先先安装好node、git、ssh、hexo，创建好hexo文件夹，安装好插件，(选做：将hexo生成的文件及文件夹删除)

``` shell
npm install hexo-cli -g
hexo init blog_folder
cd blog_folder
npm install
hexo server
```

在该文件夹(`blog_folder`)下

``` shell
git init # 初始化git仓库
git remote add origin <server> # 为本地仓库添加远程仓库
git fetch --all
git reset --hard origin/hexo_source # 获取hexo_source分支源文件
```

然后就是写博客，在新电脑发布完博客之后(`hexo deploy`)，记得将博客备份同步到远程仓库 

``` shell
git add .
git commit -m "写了一篇博客"
git push origin hexo_source
```

至此，已经完成了博客的撰写并修改了远端仓库的博客源文件，然后使用`hexo g`和`hexo d`更新博客就OK啦！

## 新机器安装npm失败解决方案

由于众所周知的原因，好多东西无法安装，可以添加第三方源来解决

``` shell
# 添加淘宝源
npm install -g cnpm --registry=https://registry.npm.taobao.org
# nrm类似包管理器
cnpm install nrm -g
nrm ls
# 使用淘宝
nrm use taobao
npm install -g hexo-cli
```


## 参考

- [关于博客同步的解决办法](http://devtian.me/2015/03/17/blog-sync-solution/)
- [使用Git Submodule管理子模块](https://segmentfault.com/a/1190000003076028)
