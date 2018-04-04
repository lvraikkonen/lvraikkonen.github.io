---
title: Python dict类型的实现
date: 2017-07-18 16:38:09
tags:
        - Python
        - dict
        - hash
---

程序员们的经验里面，通常都会认为字典和集合的速度是非常快的，字典的搜索的时间复杂读为O(1)，

为什么能有这么快呢？在于字典和集合的后台实现。

## 散列表 Hash table

散列表是一个稀疏数组，散列表里面的单元叫做表元 `bucket`， 在dict的散列表中，每个键值对都占用一个表元，每个表元有两个部分：一个对键值的引用，一个对值的引用。因为所有表元大小一致，可以通过偏移量来读取某个表元。由于是稀疏数组，python会设法保证还有大约三分之一的表元是空的，快要到达这个阈值的时候，会把原有的散列表复制到一个更大的空间里面。

<!-- more -->

如果要把一个对象放入散列表，那么需要先计算这个元素的散列值

``` python
map(hash, (0, 1, 2, 3))
# [0, 1, 2, 3]

map(hash, ("namea", "nameb", "namec", "named"))
# [6674681622036885098, -1135453951840843879, 3071659021342785694, 5386947181042036450]
```

## 常用构造hash函数的方法

构造散列函数有多种方式，比如直接寻址法、数字分析法、平方取中法、折叠法、随机数法、除留余数法等。著名的hash算法: MD5 和 SHA-1 是应用最广泛的Hash算法。

### 直接寻址法

取keyword或keyword的某个线性函数值为散列地址。即H(key)=key或H(key) = a*key + b，当中a和b为常数（这样的散列函数叫做自身函数）

### 数字分析法

分析一组数据，比方一组员工的出生年月日，这时我们发现出生年月日的前几位数字大体同样，这种话，出现冲突的几率就会非常大，可是我们发现年月日的后几位表示月份和详细日期的数字区别非常大，假设用后面的数字来构成散列地址，则冲突的几率会明显减少。因此数字分析法就是找出数字的规律，尽可能利用这些数据来构造冲突几率较低的散列地址。

### 平方取中法

取keyword平方后的中间几位作为散列地址。

### 折叠法

将keyword切割成位数同样的几部分，最后一部分位数能够不同，然后取这几部分的叠加和（去除进位）作为散列地址。

### 随机数法

选择一随机函数，取keyword的随机值作为散列地址，通经常使用于keyword长度不同的场合。

### 除留余数法

取keyword被某个不大于散列表表长m的数p除后所得的余数为散列地址。即 H(key) = key MOD p, p<=m。不仅能够对keyword直接取模，也可在折叠、平方取中等运算之后取模。对p的选择非常重要，一般取素数或m，若p选的不好，easy产生同义词。


## 散列表算法

为了获取my_dict[search_key] 的值， Python会调用hash(search_key) 来计算散列值，把这个值的最低几位数字当做偏移量，在散列表里面查找表元，如果摘到的表元是空的，则抛出KeyError异常，若不是空的， 表元里会有一对found_key: found_value，然后Python会检查search_key是否等于found_key，如果相等，就返回found_value。

![从字典取值流程图](http://7xkfga.com1.z0.glb.clouddn.com/hash_search.png)

如果search_key和found_key不匹配的话，就叫做散列冲突。

## 散列冲突解决方法

### 开放寻址法 Open addressing

Python是使用开放寻址法中的二次探查来解决冲突的。如果使用的容量超过数组大小的2/3，就申请更大的容量。数组大小较小的时候resize为*4，较大的时候resize*2。实际上是用左移的形式。

## 字典的C数据结构

下面的C结构体来存储一个字典项，包括散列值、键和值。

``` C
typedef struct {
    Py_ssize_t me_hash;
    PyObject *me_key;
    PyObject *me_value;
} PyDictEntry;
```

下面的结构代表了一个字典

``` C
typedef struct _dictobject PyDictObject;
struct _dictobject {
    PyObject_HEAD
    Py_ssize_t ma_fill;
    Py_ssize_t ma_used;
    Py_ssize_t ma_mask;
    PyDictEntry *ma_table;
    PyDictEntry *(*ma_lookup)(PyDictObject *mp, PyObject *key, long hash);
    PyDictEntry ma_smalltable[PyDict_MINSIZE];
};
```

- `ma_fill` 是使用了的slots加 dummy slots的数量和。当一个键值对被移除了时，它占据的那个slot会被标记为dummy。如果添加一个新的 key 并且新 key 不属于dummy，则 `ma_fill` 增加 1
- `ma_used` 是被占用了（即活跃的）的slots数量
- `ma_mask` 等于数组长度减一，它被用来计算slot的索引。在查找元素的一个 key 时，使用 `slot = key_hash & mask` 就能直接获得哈希槽序号
- `ma_table` 一个 PyDictEntry 结构体的数组， PyDictEntry 包含 key 对象、value 对象，以及 key 的散列值
- `ma_lookup` 一个用于查找 key 的函数指针
- `ma_smalltable` 是一个初始大小为8的数组。

Cpython中，使用如下算法来进行二次探查序列查找空闲slot

```
i = (5 * i + perturb + 1)
slot_index = i & ma_mask
perturb >>= 5
```

## 字典的使用

### 字典初始化

第一次创建一个字典，PyDict_New()函数会被调用

```
returns new dictionary object
function PyDict_New:
    allocate new dictionary object
    clear dictionary's table
    set dictionary's number of used slots + dummy slots (ma_fill) to 0
    set dictionary's number of active slots (ma_used) to 0
    set dictionary's mask (ma_value) to dictionary size - 1 = 7
    set dictionary's lookup function to lookdict_string
    return allocated dictionary object
```

### 添加项

当添加一个新键值对时PyDict_SetItem()被调用，该函数带一个指向字典对象的指针和一个键值对作为参数。它检查该键是否为字符串并计算它的hash值（如果这个键的哈希值已经被缓存了则用缓存值）。然后insertdict()函数被调用来添加新的键/值对，如果使用了的slots和dummy slots的总量超过了数组大小的2/3则重新调整字典的大小。

```
arguments: dictionary, key, value
return: 0 if OK or -1
function PyDict_SetItem:
    if key's hash cached:
        use hash
    else:
        calculate hash
    call insertdict with dictionary object, key, hash and value
    if key/value pair added successfully and capacity orver 2/3:
        call dictresize to resize dictionary's table
```

`insertdict()` `使用查找函数 `lookdict_string`来寻找空闲的slot，这和寻找key的函数是一样的。`lookdict_string()``函数利用hash和mask值计算slot的索引，如果它不能在slot索引（=hash & mask）中找到这个key，它便开始如上述伪码描述循环来探测直到找到一个可用的空闲slot。第一次探测时，如果key为空(null)，那么如果找到了dummy slot则返回之

下面一个列子，如何将{'a': 1, 'b': 2, 'z': 26, 'y': 25, 'c': 5, 'x': 24} 键值对添加到字典里面。(字典结构的表大小为8)

```
PyDict_SetItem: key='a', value = 1
    hash = hash('a') = 12416037344
    insertdict
        lookdict_string
            slot index = hash & mask = 12416037344 & 7 = 0
            slot 0 is not used so return it
        init entry at index 0 with key, value and hash
        ma_used = 1, ma_fill = 1
PyDict_SetItem: key='b', value = 2
    hash = hash('b') = 12544037731
    insertdict
        lookdict_string
            slot index = hash & mask = 12544037731 & 7 = 3
            slot 3 is not used so return it
        init entry at index 3 with key, value and hash
        ma_used = 2, ma_fill = 2
PyDict_SetItem: key='z', value = 26
    hash = hash('z') = 15616046971
    insertdict
        lookdict_string
            slot index = hash & mask = 15616046971 & 7 = 3
            slot 3 is used so probe for a different slot: 5 is free
        init entry at index 5 with key, value and hash
        ma_used = 3, ma_fill = 3
PyDict_SetItem: key='y', value = 25
    hash = hash('y') = 15488046584
    insertdict
        lookdict_string
            slot index = hash & mask = 15488046584 & 7 = 0
            slot 0 is used so probe for a different slot: 1 is free
        init entry at index 1 with key, value and hash
        ma_used = 4, ma_fill = 4
PyDict_SetItem: key='c', value = 3
    hash = hash('c') = 12672038114
    insertdict
        lookdict_string
            slot index = hash & mask = 12672038114 & 7 = 2
            slot 2 is not used so return it
        init entry at index 2 with key, value and hash
        ma_used = 5, ma_fill = 5
PyDict_SetItem: key='x', value = 24
    hash = hash('x') = 15360046201
    insertdict
        lookdict_string
            slot index = hash & mask = 15360046201 & 7 = 1
            slot 1 is used so probe for a different slot: 7 is free
        init entry at index 7 with key, value and hash
        ma_used = 6, ma_fill = 6
```

![hashtable_items](http://www.laurentluce.com/images/blog/dict/insert.png)

### 移除项

`PyDict_DelItem()`被用来删除一个字典项。key的散列值被计算出来作为查找函数的参数，删除后这个slot就成为了dummy slot。

## 参考

[Python dictionary implementation](http://www.laurentluce.com/posts/python-dictionary-implementation/)

[Fluent Python](https://www.amazon.com/Fluent-Python-Concise-Effective-Programming/dp/1491946008/ref=sr_1_1?ie=UTF8&qid=1500368395&sr=8-1&keywords=fluent+python)
