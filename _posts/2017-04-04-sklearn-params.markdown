---
layout:     post
title:      "sklearn.svm.SVC参数记录"
subtitle:   "在使用svm过程中，参数最为重要，会经常性的调参，做一下记录，以后要是用到了就可以翻来看看"
date:       2017-04-04 12:00:00
author:     "zfl"
header-img: "img/post-sklearn-svm.jpg"
header-mask: 0.6
catalog:    true
tags:
    - python
    - 机器学习
    - svm
    - sklearn
    - python库
---

# sklearn.svm.SVC参数记录  
在使用svm过程中，参数最为重要，会经常性的调参，做一下记录，以后要是用到了就可以翻来看看。
**SVC(C=1.0, kernel='rbf', degree=3, gamma='auto', coef0=0.0, shrinking=True, probability=False,tol=0.001, cache_size=200, class_weight=None, verbose=False, max_iter=-1, decision_function_shape=None,random_state=None)**  

| 参数名 | 类型 | 默认值|  描述 |
| :------ | :------ | :------:| :------------------: |
| **C** | float | 1.0 | 惩罚参数，C越大对误分类的惩罚越大，越来越趋向于把训练的数据正确分类，但是泛化能力弱；C越小对误分类的惩罚越小,相对应泛化能力较强|
| **kernel**| string | 高斯核函数（rbf） | 还可以选择多项式核函数（poly），线性核函数(linear，用于线性可分，速度快), Sigmoid核函数（sigmoid）, 自定义核函数（precomputed） |
| **degree** | int | 3 | 多项式poly函数的维度，默认是3，选择其他核函数时会被忽略|
| **gamma** | float | auto = 1/n_features| ‘rbf’,‘poly’ 和‘sigmoid’的核函数参数 |
| **coef0** | float | 0.0 | poly 和 sigmoid 核函数的常数项|
| probability | boolean | false | 是否采用概率估计|
| shrinking | boolean | True | 是否采用shrinking heuristic方法|
| tol | float | 1e-3 | 停止训练的最小误差值|
| cache_size | float　| 没有默认值 | 核函数cache缓存大小（MB）|
| class_weight | {dict,'balanced'} | 没有默认值 | |
| verbose | boolean | False |  是否允许冗余输出；|
| max_iter | int | 1 | 最大迭代次数。-1为无限制 || 
| decision_function_shape | 'ovo', 'ovr' or None | None | |
| random_state | int | | 数据洗牌时的种子值|