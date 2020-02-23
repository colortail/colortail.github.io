---
title: 常见损失函数
date: 2019-03-03 13:59:55
tags: machine learning
mathjax: true
---
### regression

**MSE**： mean square error

mse = $\frac{1}{m} \sum_i^n(y_i - \hat{y_i})^2$



**MAE**: mean absolute error

mae = $\frac{1}{m} \sum_i^n|y_i - \hat{y_i}|$



**RMSE**: root mean

mse = $\sqrt{ \frac{1}{m} \sum_i^n(y_i - \hat{y_i})^2}$

mse和rmse被称为regression L2 loss， mae是L1 loss



**possion loss**

泊松回归假设：1）y服从泊松分布，即期望为参数$\lambda$,且P(y=k)= $ e^{-\lambda}\lambda^k / k! $ ，2）y的期望的对数可以通过线性组合来建模,也就是 $ E(y) = e^{w^Tx} $

根据假设得: $ P(y|x;w)=\frac{e^{yw^Tx}e^{-e^{w^Tx}}}{y!} $,然后最大似然估计后得到

损失函数：$ L_{possion\_reg}=\sum_i^n{y_i*w^Tx_i - e^{w^Tx_i}}  $

可以用于拟合计数问题，其他的广义线性模型会假设y服从其他的分布。

<!-- more -->


**quantile**

要估计y的$\tau$分位数$\xi$

定义损失函数：

$\rho(y_i - \xi) = (y_i - \xi) (\tau - I_{(y_i<\xi)})$

这个损失函数可以表述为另一种形式：

$\rho(y_i - \xi) =   (\tau - 1)(y_i - \xi)^+   + \tau(\xi - y_i )^+ $



### classification

**confusion matrix**

|          |          | prediction     |                |
| -------- | -------- | -------------- | -------------- |
| **real** |          | Positive       | Negative       |
|          | Positive | True Positive  | False Negative |
|          | Negative | False Positive | True Negative  |

- 混淆矩阵，True/False表示是否分类正确，Positive/Negative指示出**预测**的结果
- accuracy: $\frac{TP + TN} {TP + FN + FP + TN}$
- precision:  $\frac{TP}{TP + FP}$, 预测的正例中正确分类的比例
- recall：$\frac{TP}{TP + FN}$，将实际正例区分出来的比例
- F1-score:  $$\frac{1}{f1\_score} = \frac{1}{precision} + \frac{1}{recall}$$



**ROC**: receiver operating characteristic

TPR = $\frac{TP}{TP + FN}$ ，真正率：预测正例正确的占总正的比例

FPR = $\frac{FP}{FP + TN}$，假正率：预测正例错误（实际为负）的占总负例的比例

ROC的X轴是FPR，Y轴是TPR。AUC为ROC曲线面积。

ROC曲线均在左上角，AUC越大越准确。



** 0-1 loss **

$$ L(\hat{y}, y) = I(\hat{y} \neq y) $$



