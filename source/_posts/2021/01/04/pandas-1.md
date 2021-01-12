---
title: Python强大的数据分析利器——Pandas(1)
date: 2021-01-04 08:10:40
tags:
- python
- pandas
categories:
- python
---

`Pandas`是我最喜欢的`python`库之一，不只是功能强大，它的`API`也是非常的优雅。

<!-- more -->

## Series

`Series`和原生的字典很像，`index`相当于字典的`key`。

```python
In [3]: s = pd.Series([1, 3, 5, np.nan, 6, 8])

In [4]: s
Out[4]: 
0    1.0
1    3.0
2    5.0
3    NaN
4    6.0
5    8.0
dtype: float64
```

`index`如果不指定就会默认用`0,1,2,3...`这个自然序。当然也是可以指定的。

```python
In [4]: index = ["a", "b", "c"]

In [5]: s = pd.Series([1, 2, 3], index)

In [6]: s
Out[6]: 
a    1
b    2
c    3
dtype: int64
```

## Dataframe输入

`DataFrame`是`pandas`中最重要的数据结构，它有着非常多灵活的创建方式。

### by字典

```python
# Series
In [9]: d = {'one': pd.Series([1., 2., 3.], index=['a', 'b', 'c']),
   ...:  'two': pd.Series([1., 2., 3., 4.], index=['a', 'b', 'c', 'd'])}

In [12]: df = pd.DataFrame(d)

In [13]: df
Out[13]: 
   one  two
a  1.0  1.0
b  2.0  2.0
c  3.0  3.0
d  NaN  4.0

# list
In [14]: d = {'one': [1, 2, 3, 4], 'two': [2, 3, 4, 1]}

In [15]: pd.DataFrame(d)
Out[15]: 
   one  two
0    1    2
1    2    3
2    3    4
3    4    1

# from_dict
In [57]: pd.DataFrame.from_dict(dict([('A', [1, 2, 3]), ('B', [4, 5, 6])]))
Out[57]: 
   A  B
0  1  4
1  2  5
2  3  6
```

### by字典数组

```python
In [16]: l = [{'a': 5, 'b': 6}, {'a': 11, 'b': 12, 'c': 0}]

In [17]: pd.DataFrame(l)
Out[17]: 
    a   b    c
0   5   6  NaN
1  11  12  0.0
```

### by多维数组

```python
In [19]: d = [[1,2], [3,4]]

In [20]: pd.DataFrame(d)
Out[20]: 
   0  1
0  1  2
1  3  4

```

### by SQL

```python
# pip install pymysql
# pip install sqlalchemy
In [1]: from sqlalchemy import create_engine
In [2]: import pandas as pd
In [3]: import numpy as np
In [4]: engine = create_engine('mysql+pymysql://root:test@localhost:3306/spark')
In [5]: sql = "select * from student"
In [6]: df = pd.read_sql_query(sql, engine)
In [7]: df
Out[7]: 
   id      name gender  age
0   1   Xueqian      F   23
1   2  Weiliang      M   24

```

### by CSV,EXCEL,HDF

```python
# read_excel, read_hdf同理
In [10]: df = pd.read_csv('data.csv', sep=",")

In [11]: df
Out[11]: 
   name  age
0  jack   18
1  lily   20
```

从上面的示例我们可以发现，`pandas`自动将缺失值处理成了`NaN`。

## DataFrame数据输出

```python
df.to_hdf('foo.h5', 'df')
df.to_csv('foo.csv')
df.to_excel('foo.xlsx', sheet_name='Sheet1')
```

## DataFrame访问数据

数据初始化

```python
In [1]: dates = pd.date_range('1/1/2020', periods=8)

In [2]: df = pd.DataFrame(np.random.randn(8, 4),
   ...:                   index=dates, columns=['A', 'B', 'C', 'D'])
   ...: 

In [12]: df
Out[12]: 
                   A         B         C         D
2020-01-01  1.540781  1.056957  2.368582 -1.388904
2020-01-02  0.319485  0.199442 -1.693985  0.605021
2020-01-03 -0.708602  0.961322 -0.210920  0.670033
2020-01-04 -0.952370 -2.306830  0.078056 -0.090901
2020-01-05 -0.143606  1.434278  1.168140  1.079450
2020-01-06  0.665699 -1.892689  1.632554  0.243764
2020-01-07 -0.317191  0.541485  1.037746  0.263215
2020-01-08  0.095929  0.657138 -1.048893 -0.904121
```

### 选择列

选择单列，产生 Series，与 df.A 等效：

```python
In [13]: df['A']
Out[13]: 
2020-01-01    1.540781
2020-01-02    0.319485
2020-01-03   -0.708602
2020-01-04   -0.952370
2020-01-05   -0.143606
2020-01-06    0.665699
2020-01-07   -0.317191
2020-01-08    0.095929
Freq: D, Name: A, dtype: float64

```

### 选择行

类似数组切片

```python
In [14]: df[0:3]
Out[14]: 
                   A         B         C         D
2020-01-01  1.540781  1.056957  2.368582 -1.388904
2020-01-02  0.319485  0.199442 -1.693985  0.605021
2020-01-03 -0.708602  0.961322 -0.210920  0.670033

In [15]: df['2020-01-02': '2020-01-04']
Out[15]: 
                   A         B         C         D
2020-01-02  0.319485  0.199442 -1.693985  0.605021
2020-01-03 -0.708602  0.961322 -0.210920  0.670033
2020-01-04 -0.952370 -2.306830  0.078056 -0.090901
```

### 操作符访问

df支持4种方式访问行列，`ix`在新版本`pandas`中已经废弃

* loc  按index标签选择
* at   同loc 只能访问单个元素
* iloc 按索引位置选择
* iat  同iloc 只能访问单个元素


#### loc

```python
In [16]: df.loc['2020-01-02':'2020-01-04', ['A','B']]
Out[16]: 
                   A         B
2020-01-02  0.319485  0.199442
2020-01-03 -0.708602  0.961322
2020-01-04 -0.952370 -2.306830
```

#### at

```python
# at
In [20]: df.at['2020-01-02', 'A']
Out[20]: 0.3194852199254171
```

#### iloc

```python
In [17]: df.iloc[3]
Out[17]: 
A   -0.952370
B   -2.306830
C    0.078056
D   -0.090901
Name: 2020-01-04 00:00:00, dtype: float64

# 二维切片
In [18]: df.iloc[3:5, 0:2]
Out[18]: 
                   A         B
2020-01-04 -0.952370 -2.306830
2020-01-05 -0.143606  1.434278
```

#### iat

```python
# iat
In [19]: df.iat[3,0]
Out[19]: -0.9523699346168295
```

### 布尔运算

#### 支持多个`& |` 运算符

```python
# 列选择
In [27]: df[df.A > 0]
Out[27]: 
                   A         B         C         D
2020-01-01  1.540781  1.056957  2.368582 -1.388904
2020-01-02  0.319485  0.199442 -1.693985  0.605021
2020-01-06  0.665699 -1.892689  1.632554  0.243764
2020-01-08  0.095929  0.657138 -1.048893 -0.904121

# 多个条件
In [28]: df[(df.A > 0)&(df.B < 0)]
Out[28]: 
                   A         B         C         D
2020-01-06  0.665699 -1.892689  1.632554  0.243764

# 整个df
In [29]: df[df > 0 ]
Out[29]: 
                   A         B         C         D
2020-01-01  1.540781  1.056957  2.368582       NaN
2020-01-02  0.319485  0.199442       NaN  0.605021
2020-01-03       NaN  0.961322       NaN  0.670033
2020-01-04       NaN       NaN  0.078056       NaN
2020-01-05       NaN  1.434278  1.168140  1.079450
2020-01-06  0.665699       NaN  1.632554  0.243764
2020-01-07       NaN  0.541485  1.037746  0.263215
2020-01-08  0.095929  0.657138       NaN       NaN
```

#### 支持isin函数

```python
In [31]: df['E'] = [1, 2, 3, 4, 5, 6, 7, 8]

In [32]: df[df['E'].isin([1, 2])]
Out[32]: 
                   A         B         C         D  E
2020-01-01  1.540781  1.056957  2.368582 -1.388904  1
2020-01-02  0.319485  0.199442 -1.693985  0.605021  2
```