---
title: 时间序列模型
date: 2019-04-15 21:45:43
tags: 机器学习
---
时间序列是平稳，则不会有趋势和季节性。

检验的方式：

a)自相关系数和偏相关系数，画自相关系数图，如果快速衰减则是一个平稳序列，否则不是

b) adfuller，看p值

如果不平稳，则需要进行差分，arima要求时间序列必须是一个平稳序列。



1. ma,wma

   moving average有一个参数，是向前滑动**k** periods。

   设当前点的可以存在实际值$Z_t,t \in(1,N)$ ，其预测值$S_t,  t\in(N+1, N+M)$

   

   $$ \left\{   \begin {aligned} S_t = \frac{1}{k}(Z_{t-k} + …+Z_{t-1}), t\in(1,N)\\S_t = \frac{1}{k}(S_{t-k} + …+S_{t-1}), t\in(N+1,N+M)  \end {aligned}   \right. $$

   所以可以根据前N个值计算出准确度。

   

   如果是 weight moving average，向前滑动n，则需要有n个参数$w_i, i\in(1,n)$

   存在条件: $ \sum_i^n{w_i} = 1$

   $S_i = w_1* Z_{t-n} + w_2* Z_{t-k+1} + …+w_n* Z_{t-1}$

   跟ma类似

   

2. arima

   ARIMA（p，d，q）模型中选择合适模型，其中p为自回归项，d为差分阶数，q为移动平均项数

   为了得到平稳序列，差分次数d可以作为arima参数

   根据自相关图和偏自相关图，尝试参数p和q，评估指标为aic,bic,hqic（越小越好）。

   ​	

   ​	model = sm.tsa.ARMA(y, (p, q)).fit()

   ​	print(model.aic, model.bic, model.hqic)

   ​	# 残差是否为标准正态分布，可以用qqplot

   ​	qqplot(model.resid, line='q',fit=True)

   

   ​	

3. holt-winter

4. prophet

5. croston

6. VECM
   <!-- 
   预测模型，向量的预测模型，通常可以对多个有类似趋势的时序一起预测

   

利用他们的相关性，互项挟制着运动的信息，来预测
    -->
    <https://www.statsmodels.org/dev/generated/statsmodels.tsa.vector_ar.vecm.VECM.html>

7. 马尔科夫过程，非线性模型
<!--，用于捕捉一些大行情的转移。举个例子，预测GDP的增长率，会有隐藏层的经济周期作为影响（衰退期，增长期循环出现），这里就把这个隐藏的过程用马尔科夫过程来描述，借助对隐藏层的刻画来帮助预测表面的GDP。零售品的预测，也有一些底层的比如消费旺季、淡季这种，人为划定很主观，让模型自己给你识别，

横截面扩充sku维度来增加信息。Markov那个是非线性，自己训练自己的历史，但是能识别什么时候需要参数调整
-->
<https://www.statsmodels.org/dev/examples/notebooks/generated/markov_autoregression.html>

8. Threshold autoregressive 模型，是非线性的ARIMA模型
<!--
能识别threshold的这种，这个很像markov非线性那个，是通过历史（可观测的销量）值判定模型发生改变了，也就是一段时间是模型1，一段时间是模型2，来回改变，这样比一成不变的arima要灵活
-->
9. GARCH(p,q)， ARCH(p)
<!--根据历史销量预测未来的标准差，

GARCH(p,q)， ARCH(p)看这两部分就行
-->
<http://www.blackarbs.com/blog/time-series-analysis-in-python-linear-models-to-garch/11/1/2016>
10. 动态因子模型
<!--看下这个：动态因子模型，Stock and Watson 那个很著名，着可以多个sku一起训练，找到他们共同的影响因子，降低维度， 用因子预测sku销量
-->
<http://www.chadfulton.com/topics/dfm_coincident.html>

