---
title: Homebrew 管理后台服务
date: 2018-04-11 09:08:38
categories: [备忘]
tag: [Mac, Homebrew]
---

在MacOS上，Homebrew是一个特别好的软件包管理的工具。以前只是拿来安装软件，偶然一次Mac重启了，好多后台服务就停了，这时候想起来Homebrew还能管理它所安装的软件的后台服务。下面内容转自[Starting and Stopping Background Services with Homebrew](https://robots.thoughtbot.com/starting-and-stopping-background-services-with-homebrew) 作为备忘。

<!-- more -->

如果你使用 `Homebrew` 安装过 `mysql` 那么下面的安装后提示你可能比较熟悉

```
To have launchd start mysql at login:
    ln -sfv /usr/local/opt/mysql/*.plist ~/Library/LaunchAgents
Then to load mysql now:
    launchctl load ~/Library/LaunchAgents/homebrew.mxcl.mysql.plist
Or, if you don't want/need launchctl, you can just run:
    mysql.server start
```

如果按上面的说明操作的话，未免太麻烦了，而且也很难记住 plist 的位置。还好 Homebrew 提供了一个易用的接口来管理 plist，然后你就不用再纠结什么 `ln`，`launchctl`，和 plist 的位置了。

## brew services

首先安装 `brew services` 命令
```
brew tap gapple/services
```

下面举个例子
```
$ brew services start mysql
==> Successfully started `mysql` (label: homebrew.mxcl.mysql)
```

在后台，`brew services start` 其实执行了最上面的安装后消息里面提到的所有命令，比如首先运行 `ln -sfv ...`，然后 `launchctl load ~/Library/LaunchAgents/homebrew.mxcl.mysql.plist` 。

假设突然 MySQL 出毛病了，我们要重启一下，那么执行下面的命令就行了
```
brew services restart mysql
Stopping `mysql`... (might take a while)
==> Successfully stopped `mysql` (label: homebrew.mxcl.mysql)
==> Successfully started `mysql` (label: homebrew.mxcl.mysql)
```

想看所有的已启用的服务的话：
```
$ brew services list
redis      started      442 /Users/gabe/Library/LaunchAgents/homebrew.mxcl.redis.plist
postgresql started      443 /Users/gabe/Library/LaunchAgents/homebrew.mxcl.postgresql.plist
mongodb    started      444 /Users/gabe/Library/LaunchAgents/homebrew.mxcl.mongodb.plist
memcached  started      445 /Users/gabe/Library/LaunchAgents/homebrew.mxcl.memcached.plist
mysql      started    87538 /Users/gabe/Library/LaunchAgents/homebrew.mxcl.mysql.plist
```

要注意的是，这里不止显示通过 `brew services` 加载的服务，也包含 `launchctl load` 加载的。

如果你卸载了 MySQL 但是 `Homebrew` 没把 plist 文件删除的话，你可以

```
$ brew services cleanup
Removing unused plist /Users/gabe/Library/LaunchAgents/homebrew.mxcl.mysql.plist
```