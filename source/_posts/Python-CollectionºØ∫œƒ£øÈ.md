---
title: Python Collection集合模块
date: 2017-05-27 18:40:54
tags: 
- Python
- Data Structure
---


``` python
import collections
```

collections是Python内建的一个集合模块，提供了许多有用的集合类。Python拥有一些内置的数据类型，比如`str`, `int`, `list`, `tuple`, `dict`等， collections模块在这些内置数据类型的基础上，提供了几个额外的数据类型：

- namedtuple(): 生成可以使用名字来访问元素内容的tuple子类
- deque: 双端队列，可以快速的从另外一侧追加和推出对象
- Counter: 计数器，主要用来计数
- OrderedDict: 有序字典
- defaultdict: 带有默认值的字典


<!-- more -->


## namedtuple

`namedtuple`主要用来产生可以使用名称来访问元素的数据对象，通常用来增强代码的可读性。

`namedtuple`是一个函数，它用来创建一个自定义的tuple对象，并且规定了tuple元素的个数，并可以用属性而不是索引来引用tuple的某个元素。

``` python
from collections import namedtuple

# namedtuple('名称', [属性list])
Point = namedtuple('Point', ['x', 'y'])
p = Point(1, 2)
print p.x, p.y
# 1, 2
isinstance(p, Point)
# True
isinstance(p, tuple)
# True
```

用`namedtuple`可以很方便地定义一种数据类型，它具备tuple的不变性，又可以根据属性来引用


## deque

`deque`其实是double-ended queue的缩写，翻译过来就是双端队列，它最大的好处就是实现了从队列头部快速增加和取出对象: `.popleft()`, `.appendleft()`

list对象的这两种用法的时间复杂度是 O(n) ，也就是说随着元素数量的增加耗时呈 线性上升。而使用deque对象则是 O(1) 的复杂度

``` python
>>> from collections import deque
>>> q = deque(['a', 'b', 'c'])
>>> q.append('x')
>>> q.appendleft('y')
>>> q
deque(['y', 'a', 'b', 'c', 'x'])
```

`deque`提供了很多方法，例如 `rotate`

``` python
# -*- coding: utf-8 -*-
"""
下面这个是一个有趣的例子，主要使用了deque的rotate方法来实现了一个无限循环
的加载动画
"""
import sys
import time
from collections import deque

fancy_loading = deque('>--------------------')

while True:
    print '\r%s' % ''.join(fancy_loading),
    fancy_loading.rotate(1)
    sys.stdout.flush()
    time.sleep(0.08)

# Result:

# 一个无尽循环的跑马灯
# ------------->-------
```

## Counter

`Counter`是一个简单的计数器，例如，统计字符出现的个数

``` python
>>> from collections import Counter
>>> c = Counter()
>>> for ch in 'programming':
...     c[ch] = c[ch] + 1
...
>>> c
Counter({'g': 2, 'm': 2, 'r': 2, 'a': 1, 'i': 1, 'o': 1, 'n': 1, 'p': 1})

>>> # 获取出现频率最高的5个字符
>>> print c.most_common(5)
```

`Counter`实际上也是dict的一个子类，上面的结果可以看出，字符'g'、'm'、'r'各出现了两次，其他字符各出现了一次。


## defaultdict

使用dict时，如果引用的Key不存在，就会抛出KeyError。如果希望key不存在时，返回一个默认值，就可以用`defaultdict` : 如果使用defaultdict，只要你传入一个默认的工厂方法，那么请求一个不存在的key时， 便会调用这个工厂方法使用其结果来作为这个key的默认值

``` python
>>> from collections import defaultdict
>>> dd = defaultdict(lambda: 'N/A')
>>> dd['key1'] = 'abc'
>>> dd['key1'] # key1存在
'abc'
>>> dd['key2'] # key2不存在，返回默认值
'N/A'
```


## OrderedDict

使用dict时，Key是无序的。在对dict做迭代时，我们无法确定Key的顺序。

如果要保持Key的顺序，可以用`OrderedDict`

``` python
>>> from collections import OrderedDict
>>> d = dict([('a', 1), ('b', 2), ('c', 3)])
>>> d # dict的Key是无序的
{'a': 1, 'c': 3, 'b': 2}
>>> od = OrderedDict([('a', 1), ('b', 2), ('c', 3)])
>>> od # OrderedDict的Key是有序的
OrderedDict([('a', 1), ('b', 2), ('c', 3)])
```

**Note: OrderedDict的Key会按照插入的顺序排列，不是Key本身排序**

