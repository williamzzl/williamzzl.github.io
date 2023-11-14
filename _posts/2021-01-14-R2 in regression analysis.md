---
layout: post
title:  "R2 in regression analysis"
date:   2021-01-14 08:43:59
author: William Zhang
categories: MachineLearning
tags:	sklearn regression R2 MSE RMSE MAE ML
cover:  "/assets/images/linear_regression.png"
---

## Introduction
The coefficient of determination, also named as R2 (R sequared) is a metric in regression model to determine the quality of model's prediction. Other similar metrics are MSE, RMSE and MAE.
![R2](/assets/images/R2.png)

## R2 values with different meanings
R2 value normally lands in range (0,1) and in sklearn can be divided in
* R2=1
Ideal condition but hard to be true in real case. Every predicted y hat equals the real y. 
* 0<R2<1
For most cases. The deviation between predicted value and real one is less than the gap between its mean and real value. The higher it is, the better the result is.
* R2<0
For bad modeling case and should be improved. The prediction is not even better than taking is mean directly.

## Usage in sklearn with example
{% highlight python %}
clf = LinearRegression().fit(Xtrain, Ytrain)
yhat = clf.predict(Xtest) #Predicted value on test
clf.score(Xtest, Ytest)
0.6043668160178814
{% endhighlight %}
It is same to using r2_score metrics
{% highlight python %}
from sklearn.metrics import r2_score
(Ytest, yhat)
0.6043668160178814
{% endhighlight %}
> **_NOTE:_** The order of input parameters in r2_score is important. Keep the 1st as real target
value and 2nd as predicted one. Otherwise it will return the wrong R2 score. Also applies to all
the other metics below
in below.

## Other metrics
#### MSE (Mean Squared Error)
MSE is a metric by taking the means of every predicted value and its coresponding one in sequare.
![MSE](/assets/images/MSE.png)
{% highlight python %}
from sklearn.metrics import mean_squared_error
mean_squared_error(Ytest, yhat)
0.5309012639324575
{% endhighlight %}

#### RMSE (Root Mean Squared Error)
Root of MSE, so in case of MSE is too big, the root value can be in the same level of test data.
![RMSE](/assets/images/RMSE.png)
There is no ''root_mean_squared_error'' in sklearn, but can set squared=False in mean_squared_error
to get RMSE.
{% highlight python %}
from sklearn.metrics import mean_squared_error
mean_squared_error(Ytest, yhat, squared=False)
0.7286297166136292
{% endhighlight %}

#### MAE (Mean Absolut Error)
![MAE](/assets/images/MAE.png)
{% highlight python %}
from sklearn.metrics import mean_absolute_error
mean_absolute_error(Ytest, yhat)
0.5307069814636165
{% endhighlight %}