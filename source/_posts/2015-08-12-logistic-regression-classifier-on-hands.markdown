---
layout: post
title: "Logistic Regression原理以及应用"
date: 2015-08-19 17:54:09 +0800
comments: true
tag: [Algorithm, Machine Learning, Logistic Regression, Python, Scikit-Learn]
categories: [算法, Python]
---

逻辑回归算法是一个很有用的分类算法，这篇文章总结一下逻辑回归算法的相关内容。数据使用scikit-learn自带的`Iris`数据集。

## Iris dataset
Iris数据集，里面包含3种鸢尾花品种的4各属性，这个分类问题可以描述成使用鸢尾花的属性，来判断这个品种倒地属于哪个品种类别。为了简单，这里使用两个类别：`Setosa`和`Versicolor`，两个属性：`Length`和`Width`

```python
from sklearn import datasets
import matplotlib.pyplot as plt
import pandas as pd
import numpy as np

data = datasets.load_iris()
X = data.data[:100, : 2]
y = data.target[:100]
setosa = plt.scatter(X[:50, 0], X[:50, 1], c='b')
versicolor = plt.scatter(X[50:, 0], X[50:, 1], c='r')
plt.xlabel("Sepal Length")
plt.ylabel("Seqal Width")
plt.legend((setosa, versicolor), ("Setosa", "Versicolor"))
```

![Iris dataset](http://7xkfga.com1.z0.glb.clouddn.com/iris_data.JPG)

可以看出来，两个品种可以被区分开，接下来要使用一种算法，让计算机把这两个类别区分开。可以想象，可以使用线性回归，也就是画一条线来把两个类别分开，但是这种分割很粗暴，准确性也不高，所以接下来要使用的算法要使用概率的方法区分两个类别，比如，算法返回0.9，那么代表属于类别A的概率是90%

<!--more-->

## 逻辑函数 Logistic Function
这里使用的逻辑函数正好符合概率的定义，即函数返回值在[0, 1]区间内，函数又被称作sigmod函数：

$$y=\dfrac {1} {1+e^{-x}}$$

```python
x_values = np.linspace(-5, 5, 100)
y_values = [1 / (1 + np.exp(-x)) for x in x_values]
plt.plot(x_values, y_values)
plt.xlabel("X")
plt.ylabel("y")
```

![sigmod function](http://7xkfga.com1.z0.glb.clouddn.com/sigmod.png)

### 将逻辑函数应用到数据上
现在，数据集有两个属性`Sepal Length`和`Sepal Width`，这两个属性可以写到如下的等式中：

$$x=\theta\_{0}+\theta\_{1}SW +\theta\_{2}SL$$

SL代表`Sepal Length`这个特征，SW代表`Sepal Width`这个特征，假如神告诉我们 $\theta\_{0} = 1$，$\theta\_{1} = 2$，$\theta\_{2} = 4$，那么，长度为5并且宽度为3.5的这个品种，$x=1+\left\( 2\ast 3.5\right) +\left( 4\ast 5\right) = 28 $，代入逻辑函数：

$$\dfrac\{1} {1+e^{-28}}=0.99$$

说明这个品种数据Setosa的概率为99%。那么，告诉我们 $\theta$的取值的神是谁呢？

## 算法学习

### Cost Function
在学习线性回归时候，当时使用的是`Square Error`作为损失函数，那么在逻辑回归中能不能也用这种损失函数呢？当然可以，不过在逻辑回归算法中，使用`Square Error`作为损失函数是非凸函数，也就是说有多个局部最小值，不能取到全局最小值，所以这里应该使用其他的损失函数。

想象一下，我们假设求出来一个属性的结果值是1，也就是预测为`Setosa`类别，那么预测为`Versicolor`类别的概率为0，在全部的数据集上，假设数据都是独立分布的，那么我们的目标就是：把每个单独类别的概率结果值累乘起来，并求最大值：

$$\prod\_{Setosa}\frac{1}{1 + e^{-(\theta\_{0} + \theta{1}SW + \theta\_{2}SL)}}\prod\_{Versicolor}1 - \frac{1}{1 + e^{-(\theta\_{0} + \theta{1}SW + \theta_{2}SL)}}$$

参考上面定义的逻辑函数：

$$h(x) = \frac{1}{1 + e^{-x}}$$

那么我们的目标函数就是求下面函数的最大值：

$$\prod\_{Setosa}h(x)\prod_{Versicolor}1 - h(x)$$

解释一下，加入类别分别为0和1，回归结果$h\_\theta(x)$表示样本属于类别1的概率，那么样本属于类别0的概率为 $1-h_\theta(x)$，则有

$$p(y=1|x,\theta)=h_\theta(x)$$

$$p(y=0|x,\theta)=1-h_\theta(x)$$

可以写为下面公式，含义为：某一个观测值的概率

$$p(y|x,\theta)=h\_\theta(x)^y(1-h_\theta(x))^{1-y}$$

由于各个观测值相互独立，那么联合分布可以表示成各个观测值概率的乘积：

$$L(\theta)=\prod\_{i=1}^m{h\_\theta(x^{(i)})^{y^{(i)}}(1-h_\theta(x^{(i)}))^{1-y^{(i)}}}$$

上式称为n个观测的似然函数。我们的目标是能够求出使这一似然函数的值最大的参数估计。对上面的似然函数取对数

$$\begin{aligned} l(\theta)=log(L(\theta))=log(\prod\_{i=1}^m{h\_\theta(x^{(i)})^{y^{(i)}}(1-h\_\theta(x^{(i)}))^{1-y^{(i)}}})
=\sum\_{i=1}^m{(y^{(i)}log(h\_\theta(x^{(i)}))+(1-y^{(i)})log(1-h\_\theta(x^{(i)})))} \end{aligned}$$

最大化似然函数，使用梯度下降法，求出$\theta$值，稍微变换一下，那就是求下面式子的最小值

$$J\left( \theta \right) = -\sum_{i=1}^{m}y^{(i)}log(h(x^{(i)})) + (1-y^{(i)})log(1-h(x^{(i)}))$$

### 梯度下降算法

梯度下降算法为：

$$\begin{aligned} \theta_j:=\theta_j-\alpha\frac{\partial J(\theta)}{\partial\theta_j} \end{aligned}$$

### 梯度下降算法的推导
对$\theta$参数求导，可得

$$\begin{aligned} \frac{\partial logh\_\theta(x^{(i)})}{\partial\theta_j}&=\frac{\partial log(g(\theta^T x^{(i)}))}{\partial\theta\_j}\\&=\frac{1}{g(\theta^T x^{(i)})}{ g(\theta^T x^{(i)})) (1-g(\theta^T x^{(i)}))x\_j^{(i)}}\\&=(1- g(\theta^T x^{(i)}))) x\_j^{(i)}\\&=(1-h\_\theta(x^{(i)}))x_j^{(i)} \end{aligned}$$

同理可得，

$$\begin{aligned} \frac{\partial(1-logh\_\theta(x^{(i)}))}{\partial\theta\_j}=-h\_\theta(x^{(i)})x_j^{(i)} \end{aligned}$$

所以

$$\begin{aligned} \frac{\partial l(\theta)}{\partial\theta\_j}&=\sum\_{i=1}^m{(y^{(i)}(1-h\_\theta(x^{(i)}))x\_j^{(i)}+(1-y^{(i)})(-h\_\theta(x^{(i)})x\_j^{(i)}))}\\&=\sum\_{i=1}^m{(y^{(i)}-h\_\theta(x^{(i)}))x\_j^{(i)}} \end{aligned}$$

那么，最终梯度下降算法为：

$$\begin{aligned} \theta\_j:=\theta\_j-\alpha\sum\_{i=1}^m{(y^{(i)}-h\_\theta(x^{(i)}))x\_j^{(i)}} \end{aligned}$$

注：虽然得到的梯度下降算法表面上看去与线性回归一样，但是这里 的 $h\_{\theta }\left( x\right) =\dfrac {1} {1+e^{-\theta ^{T}x}}$ 与线性回归中不同。

### 梯度下降算法的技巧

- 变量缩放 (Normalize Variable)
- $\alpha$选择
- 设定收敛条件

## 实现Logigtic Regrssion算法
以上，介绍了Logistic Regression算法的详细推导过程，下面就用Python来实现这个算法

首先是逻辑回归函数，也就是sigmoid函数

```python
def sigmoid(theta, x):
    return 1.0 / (1 + np.exp(-x.dot(theta)))
```

然后使用梯度下降算法估算$\theta$值，首先是gradient值

$$(y^{(i)}-h_\theta(x^{(i)}))x_j^{(i)}$$

```python
def gradient(theta, x, y):
    first_part = sigmoid(theta, x) - np.squeeze(y)
    return first_part.T.dot(x)
```

损失函数cost function

$$-\sum_{i=1}^{m}y^{(i)}log(h(x^{(i)})) + (1-y^{(i)})log(1-h(x^{(i)}))$$

```python
def cost_function(theta, x, y):
    h_theta = sigmoid(theta, x)
    y = np.squeeze(y)
    first = y * np.log(h_theta)
    second = (1 - y) * np.log(1 - h_theta)
    return np.mean(-first - second)
```

梯度下降算法，这种梯度下降算法也叫批量梯度下降(Batch gradient descent)

$$\begin{aligned} \theta_j:=\theta\_j-\alpha\sum\_{i=1}^m{(y^{(i)}-h\_\theta(x^{(i)}))x\_j^{(i)}} \end{aligned}$$

```python
def gradient_descent(theta, X, y, alpha=0.001, converge):
    X = (X - np.mean(X, axis=0)) / np.std(X, axis=0)
    cost_iter = []
    cost = cost_function(theta, X, y)
    cost_iter.append([0, cost])
    i = 1
    while(change_cost > converge):
        theta = theta - (alpha * gradient(theta, X, y))
        cost = cost_function(theta, X, y)
        cost_iter.append([i, cost])
        i += 1
    return theta, np.array(cost_iter)
```

预测方法

```python
def predict_function(theta, x):
    x = (x - np.mean(x, axis=0)) / np.std(x, axis=0)
    pred_prob = sigmoid(theta, x)
    pred_value = np.where(pred_prob >= 0.5, 1, 0)
    return pred_value
```

### 损失函数变化趋势
画出cost function的变化趋势，看看是不是已经收敛了

![cost_trend](http://7xkfga.com1.z0.glb.clouddn.com/cost_trend.png)

看来梯度下降算法已经收敛了。


## 使用Scikit-Learn中的Logistic Regression算法
Scikit-Learn库中，已经包含了逻辑回归算法，下面用这个工具集来体验一下这个算法。

```python
from sklearn import linear_model
model = linear_model.LogisticRegression()
model.fit(X, y)
model.predict(X_test)
```

### 其他优化算法

- BFGS
- 随机梯度下降 Stochastic gradient descent
- L-BFGS
- Conjugate Gradient

