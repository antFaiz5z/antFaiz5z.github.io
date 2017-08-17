---
title: 利用Python进行数据分析
comments: true
toc: true
date: 2017-08-15 14:12:07
categories: Python
tags: Python
---

### 常用工具

- numpy
- pandas
- matplotlib
- scikit-learn 

### 常见问题

- [ Matplotlib图表不能在Pycharm中显示的问题](http://blog.csdn.net/xinluqishi123/article/details/63523531)

- ndarray 与list 互转

```python
np.array(list)
np.asarray(list)
np.asanyarray(list)

ndarray.tolist()
```
- using scikit-learn and numpy

    `ValueError: Found input variables with inconsistent numbers of samples: [xx, xxx]`

```python
#convert list or some others to nparray and reshape to (xx, 1)
x_train = np.array(x_train)
# if need
x_train = x_train.reshape(32383,1)

y_train = np.array(y_train)
#if need
y_train = y_train.reshape(32383,1)

svr.fit(x_train, y_train)
# if DataConvertWarning, then ravel back to shape(1, xx)
svr.fit(np.ravel(x_train), np.ravel(y_train)
```