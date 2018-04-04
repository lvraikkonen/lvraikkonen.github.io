---
layout: post
title: "单变量线性回归"
date: 2015-08-13 15:10:52 +0800
comments: true
categories: [算法, Python]
tag: [Algorithm, Coursera, Machine Learning, Python]
---

1月份的时候，参加了Coursera上面Andrew Ng的[Machine Learning](https://www.coursera.org/learn/machine-learning/home/welcome)课程，课程断断续续的学，没有透彻的理解、推导，再加上作业使用Octave完成，并且还是有模版的，不是从头到尾做出来的，所以效果很差，虽然拿到了完成证书，但是过后即忘。我觉得是时候从头学习一遍，并且用Python实现所有的作业内容了。

这里写个系列，就当作为这门课程的课程笔记。

<!--more-->

机器学习的本质是首先将训练集“喂给”学习算法，进而学习到一个假设(Hypothesis)，然后将特征值作为输入变量输入给Hypothesis，得出输出结果。

![process](http://7xkfga.com1.z0.glb.clouddn.com/process.png)

## 线性回归

先说一元线性回归，这里只有一个特征值x，Hypothesis可以写为

$$h\_{\theta }\left( x\right) =\theta\_{0}+\theta\_{1}x$$

### 代价函数 Cost Function

现在假设已经有了这个假设，那么如何评价这个假设的准确性呢，这里用模型预测的值减去训练集中实际值来衡量，这个叫建模误差。线性回归的目标就是使建模误差最小化，从而找出能使建模误差最小化的模型参数。代价函数为：

$$J(\theta) = \frac{1}{2m}\sum\_{i=1}^m (h_\theta(x^{(i)})-y^{(i)})^2$$

则目标为求出使得$J(\theta)$最小的$\theta$参数

### 梯度下降算法

现在使用梯度下降算法来求出$\theta$参数，梯度下降算法的推导如下：

$$\begin{aligned} \theta_j:=\theta_j-\alpha\frac{\partial J(\theta)}{\partial\theta_j} \end{aligned}$$

对$\theta_0$的偏导数为：

$$\begin{aligned}
        \frac{\partial}{\partial \theta_0}J(\theta_0, \theta\_1) = \frac\{1}{m} \sum\_{i=1}^m (h_\theta(x^{(i)})-y^{(i)})
\end{aligned}$$

对$\theta_1$的偏导数为：

$$\begin{aligned}
        \frac{\partial}{\partial \theta_1}J(\theta_0, \theta\_1) = \frac\{1}{m} \sum\_{i=1}^m (h_\theta(x^{(i)})-y^{(i)})x^{(i)}
\end{aligned}$$

## 使用Python实现一元线性回归

为了提高性能，使用 `numpy` 包来实现向量化计算，

```python
import numpy as np

def hypothesis(theta, x):
    return np.dot(x, theta)

def cost_function(theta, x, y):
    loss = hypothesis(theta, x) - y
    return np.sum(loss ** 2) / (2 * len(y))
```

实现梯度下降算法，需要计算下面四个部分：
1. 计算假设Hypothesis
2. 计算损失 loss = hypothesis - y，然后求出square root
3. 计算Gradient = X' * loss / m
4. 更新参数theta -= alpha * gradient

```python
def gradient_descent(alpha, x, y, iters):
    # number of training dataset
    m = x.shape[0]
    theta = np.zeros(2)
    cost_iter = []
    for iter in range(iters):
        h_theta = hypothesis(theta, x)
        loss = h_theta - y
        J = np.sum(loss ** 2) / (2 * m)
        cost_iter.append([iter, J])
        # print "iter %s | J: %.3f" % (iter, J)
        gradient = np.dot(x.T, loss) / m
        theta -= alpha * gradient
    return theta, cost_iter
```

接下来造一些假数据试验一下

```python
from sklearn.datasets.samples_generator import make_regression
x, y = make_regression(n_samples=100, n_features=1, n_informative=1, random_state=0, noise=35)
m, n = np.shape(x)
x = np.c_[np.ones(m), x] ## add column value 1 as x0
alpha = 0.01
theta, cost_iter = gradient_descent(alpha, x, y, 1000)
print theta
```

求出的参数值为[-2.8484052 , 43.202331]

将训练集数据和Hypothesis函数画出来

```python
for i in range(x.shape[1]):
    y_predict = theta[0] + theta[1] * x
plt.plot(x[:, 1], y, 'o')
plt.plot(x, y_predict, 'k-')
```

![linear_regression_plot](http://7xkfga.com1.z0.glb.clouddn.com/linear_reg_plot.png)

接下来画出代价函数在每次迭代过程中的变化趋势，可以看出算法是否收敛

```python
plt.plot(cost_iter[:500, 0], cost_iter[:500, 1])
plt.xlabel("Iteration Number")
plt.ylabel("Cost")
```

![cost_trend](http://7xkfga.com1.z0.glb.clouddn.com/cost_trend.png)

接下来有必要好好复习一下线性代数、numpy和向量化计算。
