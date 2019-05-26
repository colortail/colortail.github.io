---
title: 处理类别特征
date: 2019-04-22 23:28:34
tags: 机器学习
mathjax: true
---

- one hot encoding

  1. pd.get_dummies(array_like)，得到的是最终的onehot编码。

  2. DictVectorizer，可以sparse=True，结果toarray或者直接sparse=False

     输入df.to_dict(orient='record')，类别的结果通过get_feature_names()获取

  3. LabelEncoder + OneHotEncoder（同样有sparse参数，需要toarray）

     LabelEncoder将单列字符串类型转换为整数类型，如：['a', 'a', 'b', 'c']转换成[0, 0, 1, 2]，结果为classes_属性。

     OneHotEncoder只能将整数类型做onehot编码

     传入为2D的array_like，若只有一列，可以reshape(-1,1)，结果为onehot结果，但不能得到features的名字。

- 降维会损失原始数据的信息，但one hot后会生成较多特征，可以考虑

- mean encoding

  主要思想是，考虑label的值，来计算类别特征的统计值，从而构造新特征。因为生成的特征跟label的数量有关，这种方法在类别变量的基数很大的时，会有效减少新生成的特征数量。

  对于分类问题，若目标有C个类别，则会生成C-1个特征，回归问题生成一个特征。

  对分类问题，生成的特征表示该记录，label的一个贝叶斯估计。这个概率是其先验和后验概率的加权：

  $$ \lambda * prior + (1 - \lambda) * post $$

  先验: $ prior = P(y=c) = \frac{y=c的样本总数}{所有样本个数} ​$

  后验概率：$ post=P(y=c|X=x_i) = \frac{该类别特征和label占所有样本的比例}{类别特征占所有样本的比例} $

  在使用时，$ \lambda = \frac{1}{1 + e^{-\frac{(n-k)}{f}}} $

  这个权重可以根据特征样本的占比来调整先验和后验的占比，若特征样本少，则先验的权重高，若特征样本多，则后验的权重高。f是一个参数，n是样本的总数，k是特征样本数。

  回归问题时，先验和后验的计算转换成求对应值的期望

  即：$ E[P(y=y_i)] ​$和$ E[P(y=y_i|X=x_i)] ​$

  

  catboost是一个可以支持类别特征处理的boosting machine，有评估表明，在有类别特征时较优，但无类别特征时表现一般。

  主要的资料有：

  [A Preprocessing Scheme for High Cardinality Categorical Attributes Classification and Predict Problems](http://helios.mm.di.uoa.gr/~rouvas/ssi/sigkdd/sigkdd.vol3.1/barreca.pdf)

  [Fighting_biases_with_dynamic_boosting](https://www.researchgate.net/profile/Liudmila_Ostroumova_Prokhorenkova/publication/318030603_Fighting_biases_with_dynamic_boosting/links/59b6742d0f7e9bd4a7fbef17/Fighting-biases-with-dynamic-boosting.pdf)

  [CatBoost: gradient boosting with categorical features support](<https://arxiv.org/abs/1810.11363>)

  [CatBoost: unbiased boosting with categorical features](https://arxiv.org/pdf/1706.09516.pdf)

  

  

  

     

     