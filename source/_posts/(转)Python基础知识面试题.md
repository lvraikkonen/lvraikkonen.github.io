---
title: (转)Python基础知识面试题
tags:
  - 面试
date: 2017-06-12 17:27:09
---


下面总结了一下常见的、易错的Python面试题



## Question 1

如下代码的输出是什么

```python
def extendList(val, list=[]):
    list.append(val)
    return list

list1 = extendList(10)
list2 = extendList(123,[])
list3 = extendList('a')

print "list1 = %s" % list1
print "list2 = %s" % list2
print "list3 = %s" % list3
```

<!-- more -->

### 答案：

```
list1 = [10, 'a']
list2 = [123]
list3 = [10, 'a']
```

<!-- more -->

### 解释

The new default list is created only once when the function is defined, and that same list is then used subsequently whenever extendList is invoked without a list argument being specified. This is because expressions in default arguments are calculated when the function is defined, not when it’s called.

list1 and list3 are therefore operating on the same default list, whereas list2 is operating on a separate list that it created (by passing its own empty list as the value for the list parameter).

The definition of the extendList function could be modified as follows, though, to always begin a new list when no list argument is specified, which is more likely to have been the desired behavior:

``` python
def extendList(val, list=None):
    if list is None: 
        list = []
    list.append(val)
    return list

# list1 = [10]
# list2 = [123]
# list3 = ['a']
```

## Python的函数参数传递

``` python
a = 1
def fun(a):
    a = 2
fun(a)
print a  # 1

a = []
def fun(a):
    a.append(1)
fun(a)
print a  # [1]
```

类型是属于对象的，而不是变量。而对象有两种, 可更改 (mutable) 与 不可更改(immutable)对象。在python中，strings, tuples, 和numbers是不可更改的对象，而list,dict等则是可以修改的对象。当一个引用传递给函数的时候,函数自动复制一份引用,这个函数里的引用和外边的引用没有半毛关系了.所以第一个例子里函数把引用指向了一个不可变对象,当函数返回的时候,外面的引用没半毛感觉.而第二个例子就不一样了,函数内的引用指向的是可变对象,对它的操作就和定位了指针地址一样,在内存里进行修改。

## Question 2

下面代码输出

``` python
def multipliers():
    return [lambda x : i * x for i in range(4)]
    
print [m(2) for m in multipliers()]
```

The output of the above code will be `[6, 6, 6, 6]` (not `[0, 2, 4, 6]`).

The reason for this is that Python’s closures are late binding. This means that the values of variables used in closures are looked up at the time the inner function is called. So as a result, when any of the functions returned by multipliers() are called, the value of i is looked up in the surrounding scope at that time. By then, regardless of which of the returned functions is called, the for loop has completed and i is left with its final value of 3. Therefore, every returned function multiplies the value it is passed by 3, so since a value of 2 is passed in the above code, they all return a value of 6 (i.e., 3 x 2).

(Incidentally, as pointed out in The Hitchhiker’s Guide to Python, there is a somewhat widespread misconception that this has something to do with lambdas, which is not the case. Functions created with a lambda expression are in no way special and the same behavior is exhibited by functions created using an ordinary def.)

Below are a few examples of ways to circumvent this issue.

One solution would be use a Python generator as follows:

``` python
def multipliers():
    for i in range(4):
        yield lambda x : i * x
```

Another solution is to create a closure that binds immediately to its arguments by using a default argument. For example:

``` python
def multipliers():
    return [lambda x, i=i : i * x for i in range(4)]
```

Or alternatively, you can use the `functools.partial` function:

``` python
from functools import partial
from operator import mul

def multipliers():
    return [partial(mul, i) for i in range(4)]
```

## 迭代器和生成器



## Question 3

``` python
class Parent(object):
    x = 1

class Child1(Parent):
    pass

class Child2(Parent):
    pass

print Parent.x, Child1.x, Child2.x
Child1.x = 2
print Parent.x, Child1.x, Child2.x
Parent.x = 3
print Parent.x, Child1.x, Child2.x

# 1, 1, 1
# 2, 1, 2
# 3, 2, 3
```

### 解释

in Python, class variables are internally handled as dictionaries. If a variable name is not found in the dictionary of the current class, the class hierarchy (i.e., its parent classes) are searched until the referenced variable name is found (if the referenced variable name is not found in the class itself or anywhere in its hierarchy, an AttributeError occurs).

Therefore, setting x = 1 in the Parent class makes the class variable x (with a value of 1) referenceable in that class and any of its children. That’s why the first print statement outputs 1 1 1.

Subsequently, **if any of its child classes overrides that value (for example, when we execute the statement Child1.x = 2), then the value is changed in that child only**. That’s why the second print statement outputs 1 2 1.

Finally, **if the value is then changed in the Parent (for example, when we execute the statement Parent.x = 3), that change is reflected also by any children that have not yet overridden the value** (which in this case would be Child2). That’s why the third print statement outputs 3 2 3.


## Question 4

在 Python2 中下面代码输出是什么

``` python
def div1(x,y):
    print "%s/%s = %s" % (x, y, x/y)
    
def div2(x,y):
    print "%s//%s = %s" % (x, y, x//y)

div1(5,2)
div1(5.,2)
div2(5,2)
div2(5.,2.)

# 5/2 = 2
# 5.0/2 = 2.5
# 5//2 = 2
# 5.0//2.0 = 2.0
```

By default, Python 2 automatically performs integer arithmetic if both operands are integers. As a result, 5/2 yields 2, while 5./2 yields 2.5.

Note that you can override this behavior in Python 2 by adding the following import:

``` python 
from __future__ import division
```

Also note that the “double-slash” (//) operator will always perform integer division, regardless of the operand types. That’s why 5.0//2.0 yields 2.0 even in Python 2.

Python 3, however, does not have this behavior; i.e., it does not perform integer arithmetic if both operands are integers. Therefore, in Python 3, the output will be as follows:

```
5/2 = 2.5
5.0/2 = 2.5
5//2 = 2
5.0//2.0 = 2.0
```

## Question 5

``` python
list = ['a', 'b', 'c', 'd', 'e']
print list[10:]

# []
```

输出为空list, 不会报 `IndexError`错误

As one would expect, attempting to access a member of a list using an index that exceeds the number of members (e.g., attempting to access list[10] in the list above) results in an IndexError. However, attempting to access a slice of a list at a starting index that exceeds the number of members in the list will not result in an IndexError and will simply return an empty list.

What makes this a particularly nasty gotcha is that it can lead to bugs that are really hard to track down since no error is raised at runtime.

## Question 6

``` python
list = [ [] ] * 5
list  # output?
list[0].append(10)
list  # output?
list[1].append(20)
list  # output?
list.append(30)
list  # output?
```

### 答案

```
[[], [], [], [], []]
[[10], [10], [10], [10], [10]]
[[10, 20], [10, 20], [10, 20], [10, 20], [10, 20]]
[[10, 20], [10, 20], [10, 20], [10, 20], [10, 20], 30]
```

list = [ [ ] ] * 5 simply creates a list of 5 lists.

However, the key thing to understand here is that the **statement list = [ [ ] ] * 5 does NOT create a list containing 5 distinct lists; rather, it creates a a list of 5 references to the same list**

``` python
print id(list[0]) == id(list[1])
# True
```

`list[0].append(10)` appends 10 to the first list. But since all 5 lists refer to the same list, the output is: [[10], [10], [10], [10], [10]].

Similarly, `list[1].append(20)` appends 20 to the second list. But again, since all 5 lists refer to the same list, the output is now: [[10, 20], [10, 20], [10, 20], [10, 20], [10, 20]].

In contrast, `list.append(30)` is appending an entirely new element to the “outer” list, which therefore yields the output: [[10, 20], [10, 20], [10, 20], [10, 20], [10, 20], 30].


## Question 7

Given a list of N numbers, use a single list comprehension to produce a new list that only contains those values that are:

1. even numbers, and
2. from elements in the original list that had even indices

### 答案

``` python
#        0   1   2   3    4    5    6    7    8
list = [ 1 , 3 , 5 , 8 , 10 , 13 , 18 , 36 , 78 ]

print [x for x in list[::2] if x%2==0]
# [10, 18, 78]
```


## `*args` and `**kwargs`

当你不确定你的函数里将要传递多少参数时你可以用*args.例如,它可以传递任意数量的参数:

``` python
def print_everything(*args):
    for count, thing in enumerate(args):
        print '{0}. {1}'.format(count, thing)

print_everything('apple', 'banana', 'cabbage')
# 0. apple
# 1. banana
# 2. cabbage
```

相似的, `**kwargs`允许你使用没有事先定义的参数名:

``` python
def table_things(**kwargs):
    for name, value in kwargs.items():
        print '{0} = {1}'.format(name, value)

table_things(apple = 'fruit', cabbage = 'vegetable')
# cabbage = vegetable
# apple = fruit
```

你也可以混着用.命名参数首先获得参数值然后所有的其他参数都传递给*args和**kwargs.命名参数在列表的最前端.例如:

def table_things(titlestring, **kwargs)
*args和**kwargs可以同时在函数的定义中,但是*args必须在**kwargs前面.

当调用函数时你也可以用*和**语法.例如:

``` python
def print_three_things(a, b, c):
    print 'a = {0}, b = {1}, c = {2}'.format(a,b,c)

mylist = ['aardvark', 'baboon', 'cat']
print_three_things(*mylist)

a = aardvark, b = baboon, c = cat
```

## Python里的拷贝

引用和copy(),deepcopy()的区别

``` python
import copy
a = [1, 2, 3, 4, ['a', 'b']]  #原始对象

b = a  #赋值，传对象的引用
c = copy.copy(a)  #对象拷贝，浅拷贝
d = copy.deepcopy(a)  #对象拷贝，深拷贝

a.append(5)  #修改对象a
a[4].append('c')  #修改对象a中的['a', 'b']数组对象

print 'a = ', a
print 'b = ', b
print 'c = ', c
print 'd = ', d

# 输出结果：
# a =  [1, 2, 3, 4, ['a', 'b', 'c'], 5]
# b =  [1, 2, 3, 4, ['a', 'b', 'c'], 5]
# c =  [1, 2, 3, 4, ['a', 'b', 'c']]
# d =  [1, 2, 3, 4, ['a', 'b']]
```