---
title: 时间序列模型
date: 2019-04-15 21:45:43
tags: 机器学习
---

1. VECM预测模型，向量的预测模型，通常可以对多个有类似趋势的时序一起预测

利用他们的相关性，互项挟制着运动的信息，来预测

<https://www.statsmodels.org/dev/generated/statsmodels.tsa.vector_ar.vecm.VECM.html>



2. 非线性模型，用于捕捉一些大行情的转移。举个例子，预测GDP的增长率，会有隐藏层的经济周期作为影响（衰退期，增长期循环出现），这里就把这个隐藏的过程用马尔科夫过程来描述，借助对隐藏层的刻画来帮助预测表面的GDP。零售品的预测，也有一些底层的比如消费旺季、淡季这种，人为划定很主观，让模型自己给你识别，

横截面扩充sku维度来增加信息。Markov那个是非线性，自己训练自己的历史，但是能识别什么时候需要参数调整

<https://www.statsmodels.org/dev/examples/notebooks/generated/markov_autoregression.html>



3. Threshold autoregressive 模型，是非线性的ARIMA模型

能识别threshold的这种，这个很像markov非线性那个，是通过历史（可观测的销量）值判定模型发生改变了，也就是一段时间是模型1，一段时间是模型2，来回改变，这样比一成不变的arima要灵活

 

4. 根据历史销量预测未来的标准差，

GARCH(p,q)， ARCH(p)看这两部分就行

<http://www.blackarbs.com/blog/time-series-analysis-in-python-linear-models-to-garch/11/1/2016>



5. 看下这个：动态因子模型，Stock and Watson 那个很著名，着可以多个sku一起训练，找到他们共同的影响因子，降低维度， 用因子预测sku销量

<http://www.chadfulton.com/topics/dfm_coincident.html>