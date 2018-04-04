---
title: 'Flask 从入门到放弃1: Hello World'
tags:
  - Python
  - Flask
categories: Flask从入门到放弃
date: 2017-07-25 16:25:46
---

除了Flask，常见的Python Web框架还有：

- Django：全能型Web框架；
- web.py：一个小巧的Web框架；
- Bottle：和Flask类似的Web框架；
- Tornado：Facebook的开源异步Web框架。

## Flask简介

Flask 是一个用于 Python 的微型网络开发框架，依赖两个外部库： Jinja2 模板引擎和 Werkzeug WSGI 套件。Flask也被称为microframework，因为它使用简单的核心，用加载扩展的方式增加其他功能。

Flask 没有默认使用的数据库、窗体验证工具。但是，Flask 保留了扩增的弹性，可以用Flask扩展加入这些功能：ORM、窗体验证工具、文件上传、开放式身份验证技术。

![FlaskLogo](http://docs.jinkan.org/docs/flask/_images/logo-full.png)

<!-- more -->

## 准备环境

对于Python来说，有相当数量的外部包，如果管理不当，会让人崩溃，创建一个Flask的Web项目更是这样，所以推荐一个项目一套环境。这里使用 `virtualenv` 在项目的目录中创建这个项目的虚拟环境。

``` shell
$ sudo pip install virtualenv
$ mkdir myproject
$ cd myproject
$ virtualenv venv
$ . venv/bin/activate
$ pip install Flask
```

## Hello World 应用

官方文档上给出的hello world例子很小，但是也基本说明了一个Flask应用都包含了什么

``` python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello World!'

if __name__ == '__main__':
    app.run()
```

保存代码为 `hello.py`，在命令行运行

``` shell
python hello.py
```

这时候访问 http://127.0.0.1:5000/ 可以看见页面输出 Hello World!

## 程序都干了啥

1. 首先导入了 `Flask` 类，这个类的实例会是WSGI应用程序
2. 接下来创建了这个类的实例 `app`
3. 然后，使用 `route()` 装饰器进行路由绑定，通过路由来绑定URL和Python函数的映射关系。
4. 最后使用 `run()` 函数来让应用运行在本地服务器上。 其中 `if __name__ == '__main__':` 确保服务器只会在该脚本被 Python 解释器直接执行的时候才会运行，而不是作为模块导入的时候。

好了，Ctrl+C关闭服务器。到这里一个麻雀虽小五脏俱全的小Flask应用就创建好了。
