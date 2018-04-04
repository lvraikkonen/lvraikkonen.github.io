---
title: 'Flask 从入门到放弃3: 渲染模版'
date: 2017-08-14 11:22:12
tags:
  - Python
  - Flask
categories: Flask从入门到放弃
---


## MVC

MVC：Model-View-Controller，中文名“模型-视图-控制器”

- Model（模型）是应用程序中用于处理应用程序数据逻辑的部分。通常模型对象负责在数据库中存取数据。
- View（视图）是应用程序中处理数据显示的部分。通常视图是依据模型数据创建的。
- Controller（控制器）是应用程序中处理用户交互的部分。通常控制器负责从视图读取数据，控制用户输入，并向模型发送数据。

Flask支持MVC模型，Flask默认使用Jinjia2模板引擎，对模版进行渲染，最终生成HTML文件。视图方法有两个作用：处理业务逻辑（比如操作数据库）和 返回响应内容。模板起到了将两者分开管理的作用。


下面介绍自动生成HTML的方法：模版渲染

<!-- more -->

## 模版

默认情况下,Flask 在程序文件夹中的 templates 子文件夹中寻找模板。

一个模版文件 user.html：

``` HTML
<html>
  <head>
    <title>{{ title }} - microblog</title>
  </head>
  <body>
      <h1>Hello, {{ user_name }}!</h1>
  </body>
</html>
```

调用模版

``` python
from flask import Flask, render_template

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/user/<name>')
def user(name):
    return render_template('user.html', user_name=name)
```

`render_template()`第一个参数是模板的名称，然后是 键/值 对，user_name=name左边表示模板中的占位符，右边是当前视图中的变量。

### 变量类型

模板中不仅能使用字符串数字等简单的数据类型，还能接收复杂的数据结构，比如dict、list、obj等。

``` HTML
<p>A value from a dictionary: {{ mydict['key'] }}.</p>
<p>A value from a list: {{ mylist[3] }}.</p>
<p>A value from a list, with a variable index: {{ mylist[myintvar] }}.</p>
<p>A value from an object's method: {{ myobj.somemethod() }}.</p>
```

### 控制结构

Jinjia2能够使用常见的控制流，如下是常用的几种控制流：

- if else
- for

``` html
{% if user %}
    Hello, {{user}}
{% else %}
    Hello, stranger
{% endif %}
```

``` html
<ul>
   {% for comment in comments%}
        <li>{{ comment }}</li>
    {% endfor %}
</ul>
```

## 模版继承

和类继承的方式类似，如果多个页面的大部分内容相同，可以定义一个父模板，包含相同的内容，然后子模板继承内容，并根据需要进行部分修改。`block`标签定义的元素可以在衍生模板中修改。


``` html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    {% block head%}
        <title>
            {% block title%}{% endblock%}- My Application
        </title>
    {% endblock %}
</head>
<body>
    {% block body%}
    {% endblock%}
</body>
</html>
```

父模版定义了`head`, `title`和`body`三个块，可以在子模板中进行修改。


``` html
{% extends 'base.html'%}
{% block title%}
    Index
{% endblock %}
{% block head%}
    {{ super() }}
    <style>
    </style>
{% endblock%}
{% block body%}
    <h1>Helll, World!</h1>
{% endblock%}
```

继承父模版，extends命令声明这个模版继承自base.html。其中，`super()`这个命令调用父模版的内容。

## 例子：Jinjia2集成Bootstrap

[Bootstrap](http://getbootstrap.com/) 是腿特开源的一个开源框架，可以使用这个框架创建一个比较漂亮的网页。这里使用叫做`Flask-Bootstrap`的Flask扩展。

``` shell
(venv) $ pip install flask-bootstrap
```

安装好之后，就可以在命名空间中导入了。

``` python
from flask_bootstrap import Bootstrap
# ...
bootstrap = Bootstrap(app)
```

### 创建UI的父模版

页面整体可以分为两部分：导航条和页面主体。

![Bootstrap_baseTemplate](http://7xkfga.com1.z0.glb.clouddn.com/bootstrap_fartherTemplate.png)

`extends "bootstrap/base.html"` 表明这个模版继承自Bootstrap中的bootstrap/base.html。在这个模板中定义了`title`, `navbar`, `content`和`page_content`这几个块。

### 自定义404页面

UI的框架已经定义好了，下面就可以自定义一个模版，用来显示404页面。

404.html

``` html
{% extends "base.html" %}

{% block title %}Flasky - Page not found{% endblock %}

{% block page_content %}
<div class="page-header">
    <h1>Not Found</h1>
</div>
{% endblock %}
```

这个页面继承自上面定义好的父模版，在404.html模版页里面，对`title`和`page_content`这两个块的内容进行修改。

现在页面的UI就从丑陋变得稍微美观一点了。

![render_template_html](http://7xkfga.com1.z0.glb.clouddn.com/render_template_result.png)
