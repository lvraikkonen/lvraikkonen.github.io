---
title: 搞懂Python装饰器
date: 2017-07-20 11:49:07
tags:
        - Python
        - 装饰器
---


装饰器是Python中的一个高阶概念，装饰器是可调用的对象，其参数是另外一个函数。装饰器可能会处理被装饰的函数然后把它返回，或者将其替换成另外一个函数或者可调用对象。

这么介绍装饰器确实很难懂，还是以例子逐步理解更容易些。

装饰器的强大在于它能够在不修改原有业务逻辑的情况下对代码进行扩展，常见的应用场景有：权限校验、用户认证、日志记录、性能测试、事务处理、缓存等。

下面记录一下我逐步理解装饰器的过程。

<!-- more -->

## 一等函数

在Python中，函数是一等对象，也就是说函数是满足以下条件的程序实体：

1. 在运行时创建
2. 能赋值给变量或者数据结构中的元素
3. 能作为参数传给函数
4. 能作为函数的返回结果

下面先看一个简单函数的定义

``` python
def hello():
    print("Hello world!")
```

python解释器遇到这段代码的时候，发生了两件事：

1. 编译代码生成一个函数对象
2. 将名为"hello"的名字绑定到这个函数对象上

![createSimpleFunc](http://7xkfga.com1.z0.glb.clouddn.com/createfunc.png)

Python中函数是一等对象，也就是说函数可以像int、string、float对象一样作为参数、或者作为返回值等进行传递。

### 函数作为参数

``` python
def foo(bar):
    return bar + 1

print(foo)
print(foo(2))
print(type(foo))


def call_foo_with_arg(foo, arg):
    return foo(arg)

print(call_foo_with_arg(foo, 3))
```

![](http://7xkfga.com1.z0.glb.clouddn.com/funcAsParam.png)

函数 `call_foo_with_arg` 接收两个参数，其中一个是可被调用的函数对象 `foo`

### 嵌套函数

函数也可以定义在另外一个函数中，作为嵌套函数

``` python
def parent():
    print("Printing from the parent() function.")

    def first_child():
        return "Printing from the first_child() function."

    def second_child():
        return "Printing from the second_child() function."

    print(first_child())
    print(second_child())
```

`first_child` 和 `second_child` 函数是嵌套在 `parent` 函数中的函数。

![nestedFunction](http://7xkfga.com1.z0.glb.clouddn.com/nestFunc.png)

当调用 `parent` 函数时，内嵌的`first_child` 和 `second_child` 函数也被调用，但是如果在 `parent` 函数中并不是调用`first_child` 和 `second_child` 函数， 而是返回这两个函数对象呢？

下面歪个楼，先介绍一下Python的变量作用域的规则。

## 变量作用域

Python是动态语言，Python的变量名解析机制有时称为LEGB法则，当在函数中使用未认证的变量名时，Python搜索4个作用域：

- local 函数内部作用域
- enclosing 函数内部与内嵌函数之间
- global 全局作用域
- build-in 内置作用域

``` python
def f1(a):
    print(a)
    print(b)

f1(3)
```

这段程序会抛出错误："NameError: name 'b' is not defined"，这是因为在函数体内，Python编译器搜索上面 LEGB 的变量，没有找到。

再下面一个例子：

``` python
b = 6
def f2(a):
    print(a)
    print(b)
    b = 9

f2(3)
```

在Python中，Python不要求声明变量，但是**假定在函数体中被赋值的变量是局部变量**，所以在这个函数体中，变量b被判断成局部变量，所以在print(b)调用时会抛出 "UnboundLocalError: local variable 'b' referenced before assignment" 的错误。

要想上面的代码运行，就必须手动在函数体内声明变量b为全局变量

``` python
b = 6
def f2(a):
    global b
    print(a)
    print(b)
    b = 9

f2(3)
```

## 闭包

有了上面的背景知识，下面就可以介绍闭包了。**闭包是指延伸了作用域的函数**。

要想理解这个概念还是挺难的，下面还是用例子来说明。

现在有个avg函数，用于计算不断增长的序列的平均值。

``` python
def make_averager():
    series = []

    def averager(new_value):
        series.append(new_value)
        total = sum(series)
        return total / len(series)

    return averager

avg = make_averager()
avg(10)
avg(11)
```

调用函数 `make_averager` 时候，返回一个 `averager` 函数对象，每次调用 `averager` 函数，会把参数添加到 `series` 中，然后计算当前平均值。

`series` 是 `make_averager` 函数的局部变量，调用 `avg(10)` 时， `make_averager` 函数已经返回，所以本地作用域也就不存在了。但是在 `averager` 函数中，`series` 是自由变量

![closure](http://7xkfga.com1.z0.glb.clouddn.com/closure.png)

这里的 `avg` 就是一个闭包，本质上它还是函数，闭包是引用了自由变量(series)的函数(averager)

![avg_func](http://7xkfga.com1.z0.glb.clouddn.com/avg_func_closure.png)


### nonlocal声明

刚才的例子稍稍改动一下，使用total和count来计算移动平均值

``` python
def make_averager():
    count = 0
    total = 0

    def averager(new_value):
        count += 1
        total += new_value
        return total / count

    return averager

avg = make_averager()
avg(10)
```

这时候会抛出错误 "UnboundLocalError: local variable 'count' referenced before assignment"。这是因为：当count为数字或者任何不可变类型时，在函数体定义中 `count = count + 1` 实际上是为count赋值，所以count就变成了局部变量。为了避免这个问题，python3引入了 `nonlocal` 声明，作用是把变量标记成 **自由变量**

``` python
def make_averager():
    count = 0
    total = 0

    def averager(new_value):
        nonlocal count, total
        count += 1
        total += new_value
        return total / count

    return averager

avg = make_averager()
print(avg(10))
print(avg(11))
print(avg(12))
```

## 装饰器

了解了闭包之后，下面就可以用嵌套函数实现装饰器了。事实上，装饰器就是一种闭包的应用，只不过传递的是函数。


### 无参数装饰器

下面写一个简单的装饰器的例子


```python
def makebold(fn):
    def wrapped():
        return '<b>' + fn() + '</b>'

    return wrapped

def makeitalic(fn):
    def wrapped():
        return '<i>' + fn() + '</i>'

    return wrapped

@makebold
@makeitalic
def hello():
    return "Hello World"

print(hello())
```

`makeitalic` 装饰器将函数 `hello` 传递给函数 `makeitalic`，函数 `makeitalic` 执行完毕后返回被包装后的 hello 函数，而这个过程其实就是通过闭包实现的

装饰器有一个语法糖@,直接@my_new_decorator就把上面一坨代码轻松化解了，这就是Pythonic的代码，简洁高效，使用语法糖其实等价于下面显式使用闭包

``` python
hello_bold = makebold(hello)
hello_italic = makeitalic(hello)
```

装饰器是可以叠加使用的，对于Python中的"@"语法糖，装饰器的调用顺序与使用 @ 语法糖声明的顺序相反，上面案例中叠加装饰器相当于如下包装顺序：

``` python
hello = makebold(makeitalic(hello))
```

### 被装饰的函数带参数

再来一个例子

``` python
import time
import functools

def clock(func):

    @functools.wraps(func)
    def clocked(*args, **kwargs):
        """ in wrapper """
        t0 = time.time()

        # execute
        result = func(*args, **kwargs)

        elapsed = time.time() - t0

        name = func.__name__
        arg_lst = []
        if args:
            arg_lst.append(', '.join(repr(arg) for arg in args))
        if kwargs:
            pairs = ['%s=%r' % (k, w) for k, w in sorted(kwargs.items())]
            arg_lst.append(', '.join(pairs))
        arg_str = ', '.join(arg_lst)
        print('[%0.8fs] %s(%s) -> %r ' % (elapsed, name, arg_str, result))

        return result
    return clocked

@clock
def snooze(seconds):
    """ sleep for seconds """
    time.sleep(seconds)

@clock
def factorial(n):
    """ calculate n! """
    return 1 if n<2 else n*factorial(n-1)
```

`snooze`和`factorial`函数会作为func参数传给clock函数，然后clock函数会返回`clocked`函数。所以现在`factorial`保留的是`clocked`函数的引用。但是这也是装饰器的一个副作用：会把被装饰函数的一些元数据，例如函数名、文档字符串、函数签名等信息覆盖掉。下面会使用functools库中的 `@wraps` 装饰器来避免这个。

![func_ref](http://7xkfga.com1.z0.glb.clouddn.com/func_ref.png)

内嵌包装函数 `clocked` 的参数跟被装饰函数的参数对应，这里使用了 `(*args, **kwargs)`，是为了适应可变参数。

clocked函数做了以下几件事：
1. 记录初始时间
2. 调用原来的factorial函数，保存结果
3. 计算经过的时间
4. 格式化收集的数据，然后打印出来
5. 返回第2步保存的结果

``` python
print('*'*40, 'Calling factorial(6)')
print('6! = ', factorial(6))
```

![result](http://7xkfga.com1.z0.glb.clouddn.com/result.png)

装饰器的典型行为就是：**把被装饰的函数体换成新函数，二者接受相同的参数，返回被装饰的函数本该返回的值，同时有额外操作**

另外，内嵌包装函数 `clocked` 添加了functools库中的 `@wraps` 装饰器，这个装饰器可以把被包装函数的元数据，例如函数名、文档字符串、函数签名等信息保存下来。

``` python
print(snooze(5))
print(snooze.__doc__)
print('origin func name is:', snooze.__name__)
```

```
[5.01506114s] snooze(5) -> None
None
 sleep for seconds
origin func name is: snooze
```

### 参数化装饰器

如果装饰器本身需要传入参数，那就需要编写一个返回decorator的高阶函数，也就是针对装饰器进行装饰。

下面代码来自 Python Cookbook：

``` python
from functools import wraps
import logging

def logged(level, name=None, message=None):
    """
    Add logging to a function. level is the logging
    level, name is the logger name, and message is the
    log message. If name and message aren't specified,
    they default to the function's module and name.
    """
    def decorate(func):
        logname = name if name else func.__module__
        log = logging.getLogger(logname)
        logmsg = message if message else func.__name__

        @wraps(func)
        def wrapper(*args, **kwargs):
            log.log(level, logmsg)
            return func(*args, **kwargs)
        return wrapper
    return decorate

# Example use
@logged(logging.DEBUG)
def add(x, y):
    return x + y

@logged(logging.CRITICAL, 'example')
def spam():
    print('Spam!')
```

最外层的函数 `logged()` 接受参数并将它们作用在内部的装饰器函数上面。 内层的函数 `decorate()` 接受一个函数作为参数，然后在函数上面放置一个包装器。这个装饰器的处理过程相当于：


``` python
spam = logged(x, y)(spam)
```

首先执行`logged('x', 'y')`，返回的是 `decorate` 函数，再调用返回的函数，参数是 `spam` 函数。

## 装饰器在真实世界的应用

更多的装饰器的案例： [PythonDecoratorLibrary](https://wiki.python.org/moin/PythonDecoratorLibrary)

### 1. 给函数调用做缓存

像求第n个斐波那契数来说，是个递归算法，对于这种慢速递归，可以把耗时函数的结果先缓存起来，在调用函数之前先查询一下缓存，如果没有才调用函数

``` python
from functools import wraps

def memo(func):
    cache = {}
    miss = object()

    @wraps(func)
    def wrapper(*args):
        result = cache.get(args, miss)
        if result is miss:
            result = func(*args)
            cache[args] = result
        return result

    return wrapper

@memo
@clock
def fib(n):
    if n < 2:
        return n
    return fib(n-2) + fib(n-1)
```

也可以使用下面的functools库里面的 `lru_cache` 装饰器来实现缓存。

### 2. LRUCache

LRU就是Least Recently Used，即最近最少使用，是一种内存管理算法。

``` python
import functools

@functools.lru_cache()
@clock
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-2) + fibonacci(n-1)

print(fibonacci(6))
```

### 3. 给函数输出记日志

``` python
import time
from functools import wraps

def log(func):

    @wraps(func)
    def wrapper(*args, **kwargs):
        print("Function running")
        ts = time.time()
        result = func(*args, **kwargs)
        te = time.time()
        print("Function  = {0}".format(func.__name__))
        print("Arguments = {0} {1}".format(args, kwargs))
        print("Return    = {0}".format(result))
        print("time      = %.6f seconds" % (te - ts))

    return wrapper

@log
def sum(x, y):
    return x + y

print(sum(1, 2))
```

### 4. 数据库连接

``` python
def open_and_close_db(func):
    def wrapper(*a, **k):
        conn = connect_db()
        result = func(conn=conn, *a, **k)
        conn.commit()
        conn.close()
        return result
    return wrapper

@open_and_close_db
def query_for_dict(sql, conn):
    cur = conn.cursor()
    try:
        cur.execute(sql)
        conn.commit()
        entries = [dict(zip([i[0] for i in cur.description], row)) for row in cur.fetchall()]
            print entries
    except Exception,e:
        print e
    return entries
```

### 5. Flask路由

拿Flask的 hello world来说：

``` python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World!"

if __name__ == '__main__':
    app.run()
```


到这儿，装饰器的一些基本概念就都清楚了。

## 参考

[Python 的闭包和装饰器](https://segmentfault.com/a/1190000004461404)

[Fluent Python](https://www.amazon.com/Fluent-Python-Concise-Effective-Programming/dp/1491946008/ref=sr_1_1?ie=UTF8&qid=1500368395&sr=8-1&keywords=fluent+python)
