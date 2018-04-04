---
title: 'Flask 从入门到放弃2: 深入理解@app.route()'
tags:
  - Python
  - Flask
categories: Flask从入门到放弃
date: 2017-07-26 11:11:20
---


下面这段代码是Flask的主页上给出的，这是一段Hello World级别的代码段，但是里面包含的概念可一点都不简单。

``` python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World!"
```

这里面的 `@app.route('/')` 到底是什么意思呢，具体又是如何实现的呢？很多初学者都是很迷茫。我在集中精力理解了装饰器之后，慢慢的就对app.route 这个装饰器的原理以及目的有了了解了。

以前写过一篇文章，详细说明了装饰器的概念：[搞懂Python装饰器](https://lvraikkonen.github.io/2017/07/20/%E6%90%9E%E6%87%82Python%E8%A3%85%E9%A5%B0%E5%99%A8/) 要是忘了可以随时复习一下。

<!-- more -->

## 给装饰器传参数

需要外嵌一个工厂函数，调用这个函数，然后返回函数的装饰器

``` python
def decorator_factory(enter_message, exit_message):
    # return this decorator
    print "In decorator_factory"

    def simple_deco(func):
        def wrapper():
            print enter_message
            func()
            print exit_message
        return wrapper
    return simple_deco

@decorator_factory("Start", "End")
def hello():
    print "Hello World"

hello()
```

注意，这里使用 `@decorator_factory("Start", "End")` 的时候，实际调用的是 `decorator_factory` 函数。相当于如下调用：

```
decorator_factory("Start", "End")(hello)
```

输出结果为：

![flask_hello_world](http://7xkfga.com1.z0.glb.clouddn.com/flaskHelloWorld.png)

## 创建自己的Flask类

现在我们已经有了足够的装饰器的背景知识，可以模拟一下Flask对象里面的内容。

### 创建route装饰器

我们知道，Flask是一个类，而类方法也可以被用作装饰器。

``` python
class MyFlask():

    # decorator_factory
    def route(self, route_str):
        def decorator(func):
            return func

        return decorator

app = MyFlask()

@app.route("/")
def hello():
    return "Hello World"
```

这里不想修改被装饰函数的行为，只是想获取被装饰函数的引用，以便后面注册这个函数用。

### 添加一个存储路由的字典

现在，需要一个变量去存储路由和其关联的函数

``` python
class MyFlask():
    def __init__(self):
        self.routes = {}

    # decorator_factory
    def route(self, route_str):
        def decorator(func):
            self.routes[route_str] = func
            return func
    return decorator

    # access register variable
    def serve(self, path):
        view_function = self.routes.get(path)
        if view_function:
            return view_function()
        else:
            raise ValueError('Route "{}" has not been registered'.format(path))

app = MyFlask()

@app.route("/")
def hello():
    return "Hello World"
```

当给定的路径被注册过则返回函数运行结果，当路径尚未注册时则抛出一个异常。

### 解释动态URL

形如 `@app.route("/hello/<username>")` 这样的路径又是如何解析出参数的呢？Flask使用正则的形式表达路径。这样就可以将路径作为一种模式进行匹配。

#### 使用正则表达式

``` python
import re

route_regex = re.compile(r'^/hello/(?P<username>.+)$')
match = route_regex.match("/hello/ains")

print match.groupdict()
```

输出结果为：`{'username': 'ains'}`


现在需要使用 `(pattern, view_function)` 这个元组来保存路径编译变成一个正则表达式和注册函数的关系。然后在装饰器中，把编译好的正则表达式和注册函数的元组保存在列表中。

``` python
routes = []

def build_route_pattern(route):
    route_regex = re.sub(r'(<\w+>)', r'(?P\1.+)', route)
    return re.compile("^{}$".format(route_regex))

def route(self, route_str):
    def decorator(func):
        route_pattern = build_route_pattern(route_str)
        routes.append((route_pattern, func))

        return func

    return decorator
```

接下来，再创建一个访问routes变量的函数，如果匹配上，则返回正则表达式匹配组和注册函数组成的元组。

``` python
def get_route_match(path):
    for route_pattern, view_function in routes:
        m = route_pattern.match(path)
        if m:
            return m.groupdict(), view_function

    return None
```

再接下来要找出调用view_function的方法，使用来自正则表达式匹配组字典的正确参数

``` python
def serve(path):
    route_match = get_route_match(path)
    if route_match:
        kwargs, view_function = route_match
        return view_function(**kwargs)
    else:
        raise ValueError('Route "{}"" has not been registered'.format(path))
```

改好的MyFlask类如下：

``` python
class MyFlask():
    def __init__(self):
        self.routes = []

    @staticmethod
    def build_route_pattern(route):
        route_regex = re.sub(r'(<\w+>)', r'(?P\1.+)', route)
        return re.compile("^{}$".format(route_regex))

    def route(self, route_str):
        def decorator(func):
            route_pattern = self.build_route_pattern(route_str)
            self.routes.append((route_pattern, func))

            return func

        return decorator

    def get_route_match(self, path):
        for route_pattern, view_function in self.routes:
            m = route_pattern.match(path)
            if m:
                return m.groupdict(), view_function

        return None

    def serve(self, path):
        route_match = self.get_route_match(path)
        if route_match:
            kwargs, view_function = route_match
            return view_function(**kwargs)
        else:
            raise ValueError('Route "{}"" has not been registered'.format(path))
```

运行一段带参数的试试

``` python
app = MyFlask()

@app.route("/hello/<username>")
def hello_user(username):
    return "Hello {}!".format(username)

print app.serve("/hello/ains")
```

下面是程序运行的引用关系图

![RefDiagram](http://7xkfga.com1.z0.glb.clouddn.com/refPic.png)
