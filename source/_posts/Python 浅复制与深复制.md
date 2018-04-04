---
title: Python 浅复制与深复制
date: 2017-07-31 11:48:31
tags:
- Python
- Reference
---

> Python中，万物皆对象。

在介绍Python的浅复制和深复制之前，先来歪个楼，说明一下Python的可变对象和不可变对象。提到这里，有两个坑不得不拿出来说一下。

## 坑1：可变对象作为函数默认值

先介绍一个Python里面常见的坑：

``` python
def append_to_list(value, def_list=[]):
    def_list.append(value)
    return def_list

my_list = append_to_list(1)
my_other_list = append_to_list(2)
print my_other_list
```

这时候的输出是什么呢？

<!-- more -->

```
[1, 2]
```

意不意外？惊不惊喜？ :) 为什么呢？这是因为这个默认值是在函数建立的时候就生成了, 每次调用都是用了这个对象的”缓存”。下面图表示第二次调用函数 `append_to_list`时候的引用状况：

![ref_realtion](http://7xkfga.com1.z0.glb.clouddn.com/mutable_object_ref.png)

这就是一条避坑指南：**不要使用可变对象作为函数默认值**

``` python
def append_to_list(value, def_list=None):
    if def_list is None:
        def_list = []
    def_list.append(value)
    return def_list
```

## 坑2：list `+=` 的不同行为

``` python
a1 = range(3)
a2 = a1
a2 += [3]
print a1, a2

a1 = range(3)
a3 = a1
a3 = a3 + [3]
print a1, a3
```

你会发现第一段代码的结果a1和a2都为[0,1,2,3]，而第二段的代码a1为[0,1,2] a3为[0,1,2,3]。为什么两次会不同呢？上学时候老师不是说 `a+=b` 等价于 `a=a+b` 的吗？

## 可变和不可变数据类型

Python中，对象分为可变(`mutable`)和不可变(`immutable`)两种类型

1. 字典型(dictionary)和列表型(list)的对象是可变对象
2. 元组（tuple)、数值型（number)、字符串(string)均为不可变对象

下面验证一下这两种类型，可变对象一旦创建之后还可改变但是地址不会发生改变，即该变量指向的还是原来的对象。而不可变对象则相反，创建之后不能更改，如果更改则变量会指向一个新的对象。

``` python
s = 'abc' # 不可变对象
print(id(s))
# 45068120
s += 'd'
print(id(s))
# 75648536

l = ['a','b','c']
print(id(l)) # 可变对象
# 74842504
l += 'd'
print(id(l))
# 74842504
```

这个案列也就解释了上面坑2的原因，原因：对于可变对象，例子中的list， += 操作调用 `__iadd__` 方法，相当于 `a1 = a1.__iadd__([3])`，是直接在 a2(a1的引用)上面直接更新。而对于 +操作来说，会调用 `__add__` 方法，返回一个新的对象。所以对于可变对象来说 `a+=b` 是不等价于 `a=a+b` 的。

好了，下面进入正题，如何复制一个对象。

## 浅复制

Python中对象之间的赋值是按引用传递的 (**敲黑板！**)

标识一个对象唯一身份的是：对象的id(内存地址)，对象类型，对象值，而浅拷贝就是创建一个具有相同类型，相同值但不同id的新对象

如果需要复制对象，可以使用标准库中的copy模块，copy.copy是浅复制，只会复制父对象，而不会复制对象内部的子对象；

对于list对象，使用 list() 构造方法或者切片的方式，做的是浅复制，复制了外层容器，副本中的元素其实是源容器中元素的 **引用**

``` python
l1 = [3, [55, 44], (7, 8, 9)]
l2 = list(l1)

print(id(l1), id(l2))
print(id(l1[1]), id(l2[1]))
# 2148206816520 2148206718344
# 2148206717768 2148206717768
```

![copyObject](http://7xkfga.com1.z0.glb.clouddn.com/copyObject.png)

## 深复制

copy.deepcopy是深复制，会复制对象及其子对象。

``` python
import copy

origin_list = [0, 1, 2, [3, 4]]
copy_list = copy.copy(origin_list)
deepcopy_list = copy.deepcopy(origin_list)

origin_list.append('hhh')
origin_list[3].append('aaa')

print(origin_list, copy_list, deepcopy_list)
```

看了下面的引用关系，结果就猜不错了

![deepcopyred](http://7xkfga.com1.z0.glb.clouddn.com/deepcopy_ref.png)

在原列表后面添加一个元素，不会对复制的两个列表有影响；浅复制列表中最后一个元素是原列表最后一个元素的引用，所以添加一个元素也影响浅复制的列表。结果为

```
origin_list:   [0, 1, 2, [3, 4, 'aaa'], 'hhh']
copy_list:     [0, 1, 2, [3, 4, 'aaa']]
deepcopy_list: [0, 1, 2, [3, 4]]
```
