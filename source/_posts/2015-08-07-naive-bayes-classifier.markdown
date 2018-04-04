---
layout: post
title: "朴素贝叶斯分类器实践"
date: 2015-08-07 16:48:32 +0800
comments: true
categories: [算法, Python]
tag: [Machine Learning, Algorithm, Python]
---


## 实际案例
举个运动员的例子：

> 如果我问你Brittney Griner的运动项目是什么，她有6尺8寸高，207磅重，你会说“篮球”；我再问你对此分类的准确度有多少信心，你会回答“非常有信心”。
> 我再问你Heather Zurich，6尺1寸高，重176磅，你可能就不能确定地说她是打篮球的了，至少不会像之前判定Brittney那样肯定。因为从Heather的身高体重来看她也有可能是跑马拉松的。
> 最后，我再问你Yumiko Hara的运动项目，她5尺4寸高，95磅重，你也许会说她是跳体操的，但也不太敢肯定，因为有些马拉松运动员也是类似的身高体重。   
>       ——选自[A Programmer's Guide to Data Mining](http://guidetodatamining.com/)

这里所说的分类，就用到了所谓的概率模型。

## 朴素贝叶斯算法
朴素贝叶斯算法使用每个属性(特征)属于某个类的概率做出预测，这是一个监督性学习算法，对一个预测性问题进行概率建模。**训练模型的过程可以看做是对条件概率的计算，何以计算每个类别的相应条件概率来估计分类结果。** 这个算法基于一个假设：所有特征相互独立，任意特征的值和其他特征的值没有关联关系，这种假设在实际生活中几乎不存在，但是朴素贝叶斯算法在很多领域，尤其是自然语言处理领域很成功。其他的典型应用还有垃圾邮件过滤等等。

<!--more-->

## 贝叶斯分类器的基本原理
![bayes theory](http://7xkfga.com1.z0.glb.clouddn.com/bayes.jpg)

图片引用自[Matt Buck](https://www.flickr.com/photos/mattbuck007/3676624894)

### 贝叶斯定理

给出贝叶斯定理：

$$p\left( h|D\right) =\dfrac {p\left( D|h\right) .p\left( h\right) } {p\left( D\right) }$$

这个公式是贝叶斯方法论的基石。拿分类问题来说，h代表分类的类别，D代表已知的特征d1, d2, d3...，朴素贝叶斯算法的朴素之处在于，假设了特征d1, d2, d3...相互独立，所以贝叶斯定理又能写成：

$$p\left( h|f\_{1},f\_{2}...f\_{n}\right) =\dfrac {p\left( h\right) \prod\_{i=1}^n p\left( f\_{i}|h\right) } {p\left( f\_{1},f\_{2}...f\_{n}\right) }$$

由于$P\left( f\_{1},\ldots f\_{n}\right) $ 可视作常数，类变量$h$的条件概率分布就可以表达为：

$$p\left( h|f\_{1},...,f\_{n}\right) =\dfrac {1} {Z}.p\left( h\right) \prod \_{i=1}^{n}p\left( F\_{i}|h\right)$$

### 从概率模型中构造分类器

以上，就导出了独立分布的特征模型，也就是朴素贝叶斯模型，使用[最大后验概率](https://zh.wikipedia.org/wiki/%E6%9C%80%E5%A4%A7%E5%90%8E%E9%AA%8C%E6%A6%82%E7%8E%87)MAP(Maximum A Posteriori estimation)选出条件概率最大的那个分类，这就是朴素贝叶斯分类器：

$$\widehat {y}=\arg \max\_{y}p\left( y\right) \prod\_{i=1}^{n}p\left( x\_{i}|y\right) $$

### 参数估计

所有的模型参数都可以通过训练集的相关频率来估计。常用方法是概率的最大似然估计，类的先验概率可以通过假设各类等概率来计算（先验概率 = 1 / (类的数量)），或者通过训练集的各类样本出现的次数来估计（A类先验概率=（A类样本的数量）/(样本总数)）。为了估计特征的分布参数，我们要先假设训练集数据满足某种分布或者非参数模型。

常见的分布模型：高斯分布(Gaussian naive Bayes)、多项分布(Multinomial naive Bayes)、伯努利分布(Bernoulli naive Bayes)等.

## 使用Scikit-learn进行文本分类
目的：使用Scikit-learn库自带的新闻信息数据来进行试验，该数据集有19,000个新闻信息组成，通过新闻文本的内容，使用scikit-learn中的朴素贝叶斯算法，来判断新闻属于什么主题类别。参考：[Scikit-learn Totorial](http://scikit-learn.org/stable/tutorial/text_analytics/working_with_text_data.html#building-a-pipeline)

### 数据集

```python
from sklearn.datesets import fetch_20newsgroups
news = fetch_20newsgroups(subset='all')
print news.keys()
```

查看一下第一条新闻的内容和分组

> ['description', 'DESCR', 'filenames', 'target_names', 'data', 'target']

> From: Mamatha Devineni Ratnam <mr47+@andrew.cmu.edu>
Subject: Pens fans reactions
Organization: Post Office, Carnegie Mellon, Pittsburgh, PA
...

> 10 rec.sport.hockey

划分训练集和测试集，分为80%训练集，20%测试集

```python
split_rate = 0.8
split_size = int(len(news.data) * split_rate)
X_train = news.data[:split_size]
y_train = news.target[:split_size]
X_test  = news.data[split_size:]
y_test  = news.target[split_size:]
```

### 特征提取

为了使机器学习算法应用在文本内容上，首先应该把文本内容装换为数字特征。这里使用词袋模型([Bags of words](https://en.wikipedia.org/wiki/Bag-of-words_model))

#### 词袋模型

在信息检索中，Bag of words model假定对于一个文本，忽略其词序和语法，句法，将其仅仅看做是一个词集合，或者说是词的一个组合，文本中每个词的出现都是独立的，不依赖于其他词是否出现，或者说当这篇文章的作者在任意一个位置选择一个词汇都不受前面句子的影响而独立选择的。

Scikit-learn提供了一些实用工具(`sklearn.feature_extraction.text`)可以从文本内容中提取数值特征

![Scikit-learn feature extraction from text tool](http://7xkfga.com1.z0.glb.clouddn.com/feature_extraction_from_text.jpg)

```python
	from sklearn.feature_extraction.text import CountVectorizer, TfidfVectorizer, HashingVectorizer
	from sklearn.feature_extraction.text import TfidfTransformer
	# Tokenizing text
	count_vect = CountVectorizer()
	X_train_counts = count_vect.fit_transform(X_train)
	# Tf
	tf_transformer = TfidfTransformer(use_idf=False).fit(X_train_counts)
	X_train_tf = tf_transformer.transform(X_train_counts)
	# Tf_idf
	tfidf_transformer = TfidfTransformer()
	X_train_tfidf = tfidf_transformer.fit_transform(X_train_counts)
```

#### 稀疏性

大多数文档通常只会使用语料库中所有词的一个子集，因而产生的矩阵将有许多特征值是0（通常99%以上都是0）。
例如，一组10,000个短文本（比如email）会使用100,000的词汇总量，而每个文档会使用100到1,000个唯一的词。
为了能够在内存中存储这个矩阵，同时也提供矩阵/向量代数运算的速度，通常会使用稀疏表征例如在`scipy.sparse`包中提供的表征。

### 训练模型

上面使用文本中词的出现次数作为数值特征，可以使用多项分布估计这个特征，使用`sklearn.naive_bayes`模块的`MultinomialNB`类来训练模型。

```python
	from sklearn.naive_bayes import MultinomialNB
	# create classifier
	clf = MultinomialNB().fit(X_train_tfidf, y_train)
	docs_new = ['God is love', 'OpenGL on the GPU is fast']
	X_new_counts = count_vect.transform(docs_new)
	X_new_tfidf = tfidf_transformer.transform(X_new_counts)
	# using classifier to predict
	predicted = clf.predict(X_new_tfidf)
	for doc, category in zip(docs_new, predicted):
        print('%r => %s' % (doc, news.target_names[category]))
```

### 使用`Pipline`这个类构建复合分类器

Scikit-learn为了使向量化 => 转换 => 分类这个过程更容易，提供了`Pipeline`类来构建复合分类器，例如：

```python
from sklearn.pipeline import Pipeline
text_clf = Pipeline([('vect', CountVectorizer()),
                     ('tfidf', TfidfTransformer()),
])
```

创建新的训练模型

```python
	from sklearn.naive_bayes import MultinomialNB
	from sklearn.pipeline import Pipeline
	from sklearn.feature_extraction.text import TfidfVectorizer, HashingVectorizer, CountVectorizer	#nbc means naive bayes classifier
	nbc_1 = Pipeline([
	        ('vect', CountVectorizer()),
	        ('clf', MultinomialNB()),
	])
	nbc_2 = Pipeline([
            ('vect', HashingVectorizer(non_negative=True)),
            ('clf', MultinomialNB()),
	])
	nbc_3 = Pipeline([	   
	        ('vect', TfidfVectorizer()),
	        ('clf', MultinomialNB()),
	])
	# classifier
	nbcs = [nbc_1, nbc_2, nbc_3]
```

### 交叉验证

下面是一个交叉验证函数：

```python
	from sklearn.cross_validation import cross_val_score, KFold
	from scipy.stats import sem
	import numpy as np
	# cross validation function
	def evaluate_cross_validation(clf, X, y, K):
	    # create a k-fold croos validation iterator of k folds
	    cv = KFold(len(y), K, shuffle=True, random_state=0)
	    # by default the score used is the one returned by score method of the estimator (accuracy)
	    scores = cross_val_score(clf, X, y, cv=cv)
	    print scores
	    print ("Mean score: {0:.3f} (+/-{1:.3f})").format(np.mean(scores), sem(scores))
```

将训练集分为10份，输出验证分数：

```python
for nbc in nbcs:
	evaluate_cross_validation(nbc, X_train, y_train, 10)
```

结果为：

> CountVectorizer Mean score: 0.849 (+/-0.002)

> HashingVectorizer Mean score: 0.765 (+/-0.006)

> TfidfVectorizer Mean score: 0.848 (+/-0.004)

可以看出：CountVectorizer和TfidfVectorizer特征提取的方法要比HashingVectorizer效果好。

### 优化模型

#### 优化单词提取

在使用TfidfVectorizer特征提取时候，使用正则表达式，默认的正则表达式是`u'(?u)\b\w\w+\b'`，使用新的正则表达式`ur"\b[a-z0-9_\-\.]+[a-z][a-z0-9_\-\.]+\b"`

```python
nbc_4 = Pipeline([
    ('vect', TfidfVectorizer(
                token_pattern=ur"\b[a-z0-9_\-\.]+[a-z][a-z0-9_\-\.]+\b",)
    ),
    ('clf', MultinomialNB()),
])
evaluate_cross_validation(nbc_4, X_train, y_train, 10)
```

分数是：Mean score: 0.861 (+/-0.004) ，结果好了一点

#### 排除停止词

TfidfVectorizer的一个参数stop_words，这个参数指定的词将被省略不计入到标记词的列表中，这里使用鼎鼎有名的NLTK语料库。

```python
	import nltk
	# nltk.download()
	stopwords = nltk.corpus.stopwords.words('english')
	nbc_5 = Pipeline([
    ('vect', TfidfVectorizer(
                stop_words=stop_words,
                token_pattern=ur"\b[a-z0-9_\-\.]+[a-z][a-z0-9_\-\.]+\b",    
    )),
    ('clf', MultinomialNB()),
	])
	evaluate_cross_validation(nbc_5, X_train, Y_train, 10)
```

分数是：Mean score: 0.879 (+/-0.003)，结果又提高了


#### 调整贝叶斯分类器的alpha参数

MultinomialNB有一个alpha参数，该参数是一个平滑参数，默认是1.0，我们将其设为0.01

```python
nbc_6 = Pipeline([
    ('vect', TfidfVectorizer(
                stop_words=stopwords,
                token_pattern=ur"\b[a-z0-9_\-\.]+[a-z][a-z0-9_\-\.]+\b",         
    )),
    ('clf', MultinomialNB(alpha=0.01)),
])
evaluate_cross_validation(nbc_6, X_train, y_train, 10)
```

分数为：Mean score: 0.917 (+/-0.002)，哎呦，好像不错哦！不过问题来了，调整参数优化不能靠蒙，如何寻找最好的参数，使得交叉验证的分数最高呢？

#### 使用Grid Search优化参数

使用GridSearch寻找vectorizer词频统计, tfidftransformer特征变换和MultinomialNB classifier的最优参数

Scikit-learn上关于[GridSearch的介绍](http://scikit-learn.org/stable/modules/classes.html#module-sklearn.grid_search)
![Grid Search](http://7xkfga.com1.z0.glb.clouddn.com/Grid_Search.jpg)

```python
pipeline = Pipeline([
('vect',CountVectorizer()),
('tfidf',TfidfTransformer()),
('clf',MultinomialNB()),
]);
parameters = {
    'vect__max_df': (0.5, 0.75),
    'vect__max_features': (None, 5000, 10000),
    'tfidf__use_idf': (True, False),
    'clf__alpha': (1, 0.1, 0.01, 0.001, 0.0001),
}
grid_search = GridSearchCV(pipeline, parameters, n_jobs=1)
from time import time
t0 = time()
grid_search.fit(X_train, y_train)
print "done in %0.3fs" % (time() - t0)
print "Best score: %0.3f" % grid_search.best_score_
```

#### 输出最优参数

```python
from sklearn import metrics
best_parameters = dict()
best_parameters = grid_search.best_estimator_.get_params()
for param_name in sorted(parameters.keys()):
    print "\t%s: %r" % (param_name, best_parameters[param_name])
pipeline.set_params(clf__alpha = 1e-05,  
                    tfidf__use_idf = True,
                    vect__max_df = 0.5,
                    vect__max_features = None)
pipeline.fit(X_train, y_train)
pred = pipeline.predict(X_test)
```

经过漫长的等待，终于找出了最优参数：

done in 1578.965s
Best score: 0.902

clf__alpha: 0.01
tfidf__use_idf: True
vect__max_df: 0.5
vect__max_features: None

在测试集上的准确率为：0.915，分类效果还是不错的

```python
print np.mean(pred == y_test)
```

### 评价分类效果

在测试集上测试朴素贝叶斯分类器的分类效果

```python
	from sklearn import metrics
	import numpy as np
	#print X_test[0], y_test[0]
	for i in range(20):
	    print str(i) + ": " + news.target_names[i]
	predicted = pipeline.fit(X_train, y_train).predict(X_test)
	print np.mean(predicted == y_test)
	print metrics.classification_report(y_test, predicted)
```

结果是这样的：

id | groupname
-- | ----------------
0  | alt.atheism
1  | comp.graphics
2  | comp.os.ms-windows.misc
3  | comp.sys.ibm.pc.hardware
4  | comp.sys.mac.hardware
5  | comp.windows.x
6  | misc.forsale
7  | rec.autos
8  | rec.motorcycles
9  | rec.sport.baseball
10 | rec.sport.hockey
11 | sci.crypt
12 | sci.electronics
13 | sci.med
14 |sci.space
15 | soc.religion.christian
16 | talk.politics.guns
17 | talk.politics.mideast
18 | talk.politics.misc
19 | talk.religion.misc
   |

准确率：0.922811671088

id  | precision  |  recall | f1-score |  support 
--  | ---------- | ------- | -------- | --------------
0   |    0.94    |  0.87   |   0.91   |    175
1   |    0.85    |  0.87   |   0.86   |    199
2   |    0.91    |  0.84   |   0.88   |    221
3   |    0.81    |  0.87   |   0.84   |    179
4   |    0.87    |  0.92   |   0.89   |    177
5   |    0.91    |  0.92   |   0.91   |    179
6   |    0.88    |  0.79   |   0.83   |    205
7   |    0.94    |  0.95   |   0.94   |    228
8   |    0.96    |  0.98   |   0.97   |    183
9   |    0.96    |  0.95   |   0.96   |    197
10  |    0.98    |  1.00   |   0.99   |    204
11  |    0.96    |  0.98   |   0.97   |    218
12  |    0.93    |  0.92   |   0.92   |    172
13  |    0.93    |  0.95   |   0.94   |    200
14  |    0.96    |  0.96   |   0.96   |    198
15  |    0.93    |  0.97   |   0.95   |    191
16  |    0.92    |  0.97   |   0.94   |    173
17  |    0.98    |  0.99   |   0.98   |    184
18  |    0.95    |  0.92   |   0.94   |    172
19  |    0.83    |  0.78   |   0.81   |    115
avg / total   |    0.92   |   0.92   |   0.92   |   3770
    |         |           |          |          |




#### 参考

1. JasonDing的 [【机器学习实验】使用朴素贝叶斯进行文本的分类](http://www.jianshu.com/p/845b16559431)；
2. [Scikit-Learn Totorial](http://scikit-learn.org/stable/tutorial/text_analytics/working_with_text_data.html#building-a-pipeline)


