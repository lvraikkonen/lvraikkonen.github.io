---
layout: post
title: "开始使用Scikit-Learn"
date: 2015-07-23 18:14:35 +0800
comments: true
tag: [Machine Learning, Python, Scikit-Learn]
categories: [算法, Python]
---


Python和R是做数据分析、数据挖掘、机器学习非常好的两门语言，在这儿不去讨论谁更好这个问题，没有最好，只有合适上手。对于码农出身，非科班统计学的我来说，使用Python相当习惯和顺手。

## Python数据科学栈

Python有很多做数据的类库，先列出常用的几个：

- Numpy、Scipy 基础数据类型
- Matplotlib 绘图库
- Pandas
- Ipython notebook
- Scikit-learn、MLlib 机器学习库



<!--more-->



## 使用Scikit-learn的过程

### 数据加载

首先，将数据加载到内存中

```python
	import numpy as np
	import urllib
	url = "http://archive.ics.uci.edu/ml/machine-learning-databases/pima-indians-diabetes/pima-indians-diabetes.data"
	# download the file
	raw_data = urllib.urlopen(url)
	dataset = np.loadtxt(raw_data, delimiter=',')
	X = dataset[:, 0:7]
	y = dataset[:, 8]
```

`X`为特征数组，`y`为目标变量

### 数据标准化 (Data Normalization)

大多数的梯度算法对数据的缩放很敏感，比如列A是体重数据(50kg等等)，列B是身高数据(170cm等等)，简单地说就是两个属性尺度不一样，所以在运行算法前要进行标准化或者叫归一化。

```python
	from sklearn import preprocessing
	# normalize the data attributes
	normalized_X = preprocessing.normalize(X)
```

### 特征选取

虽然特征工程是一个相当有创造性的过程，有时候更多的是靠直觉和专业的知识，但对于特征的选取，已经有很多的算法可供直接使用。Scikit-Learn中的`Recursive Feature Elimination Algorithm`算法：

```python
from sklearn.feature_selection import RFE
from sklearn.linear_model import LogisticRegression
model = LogisticRegression()
# create RFE model and select 3 attributes
rfe = RFE(model, 3)
rfe = rfe.fit(X, y)
print rfe.support_
print rfe.ranking_
```

### 机器学习算法

看一看Scikit-learn库中所带的算法：

- 逻辑回归算法 Logistic Regression
- 朴素贝叶斯算法 Naive Bayes
- k-最邻算法 KNN
- 决策树 Decision Tree
- 支持向量机 SVM

**逻辑回归**

大多数情况下被用来解决分类问题（二元分类），但多类的分类（所谓的一对多方法）也适用。这个算法的优点是对于每一个输出的对象都有一个对应类别的概率。

```python
from sklearn import metrics
from sklearn.linear_model import LogisticRegression
model = LogisticRegression()
model.fit(X, y)
print(model)
# make predictions
expected = y
predicted = model.predict(X)
# summarize the fit of the model
print(metrics.classification_report(expected, predicted))
print(metrics.confusion_matrix(expected, predicted))
```

**朴素贝叶斯**

它也是最有名的机器学习的算法之一，它的主要任务是恢复训练样本的数据分布密度。这个方法通常在多类的分类问题上表现的很好。

```python
from sklearn import metrics
from sklearn.naive_bayes import GaussianNB
model = GaussianNB()
model.fit(X, y)
print(model)
# make predictions
expected = y
predicted = model.predict(X)
# summarize the fit of the model
print(metrics.classification_report(expected, predicted))
print(metrics.confusion_matrix(expected, predicted))
```

**k-最近邻**

kNN（k-最近邻）方法通常用于一个更复杂分类算法的一部分。例如，我们可以用它的估计值做为一个对象的特征。有时候，一个简单的kNN算法在良好选择的特征上会有很出色的表现。当参数（主要是metrics）被设置得当，这个算法在回归问题中通常表现出最好的质量。

```python
from sklearn import metrics
from sklearn.neighbors import KNeighborsClassifier
# fit a k-nearest neighbor model to the data
model = KNeighborsClassifier()
model.fit(X, y)
print(model)
# make predictions
expected = y
predicted = model.predict(X)
# summarize the fit of the model
print(metrics.classification_report(expected, predicted))
print(metrics.confusion_matrix(expected, predicted))
```

**决策树**

分类和回归树（CART）经常被用于这么一类问题，在这类问题中对象有可分类的特征且被用于回归和分类问题。决策树很适用于多类分类。

```python
from sklearn import metrics
from sklearn.tree import DecisionTreeClassifier
# fit a CART model to the data
model = DecisionTreeClassifier()
model.fit(X, y)
print(model)
# make predictions
expected = y
predicted = model.predict(X)
# summarize the fit of the model
print(metrics.classification_report(expected, predicted))
print(metrics.confusion_matrix(expected, predicted))
```

**支持向量机SVM**

SVM（支持向量机）是最流行的机器学习算法之一，它主要用于分类问题。同样也用于逻辑回归，SVM在一对多方法的帮助下可以实现多类分类。

```python
from sklearn import metrics
from sklearn.svm import SVC
# fit a SVM model to the data
model = SVC()
model.fit(X, y)
print(model)
# make predictions
expected = y
predicted = model.predict(X)
# summarize the fit of the model
print(metrics.classification_report(expected, predicted))
print(metrics.confusion_matrix(expected, predicted))
```

## 评价算法

TBD

评价算法的好坏大约有几个方面：

- precision
- recall
- F1 score

## 优化算法的参数

TBD

