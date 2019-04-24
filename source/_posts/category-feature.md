---
title: 处理类别特征和降维
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

- 降维，因为one hot后会特征变多，可以考虑降维。

- mean encoding

     

     