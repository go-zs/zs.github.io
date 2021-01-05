---
title: Python强大的数据分析利器——Pandas(2)
date: 2021-01-05 11:54:51
tags:
---

这篇主要总结下`Pandas`数据分析相关操作。

<!-- more -->

## 统计

```python
# series
np.random.seed(1234)
d1 = pd.Series(2*np.random.normal(size = 100)+3)
d2 = np.random.f(2,4,size = 100)
d3 = np.random.randint(1,100,size = 100)

d1.count()          #非空元素计算
d1.min()            #最小值
d1.max()            #最大值
d1.idxmin()         #最小值的位置，类似于R中的which.min函数
d1.idxmax()         #最大值的位置，类似于R中的which.max函数
d1.quantile(0.1)    #10%分位数
d1.sum()            #求和
d1.mean()           #均值
d1.median()         #中位数
d1.mode()           #众数
d1.var()            #方差
d1.std()            #标准差
d1.mad()            #平均绝对偏差
d1.skew()           #偏度
d1.kurt()           #峰度
d1.describe()       #一次性输出多个描述性统计指标

# DataFrame
In [45]: df = pd.DataFrame(np.array([d1,d2,d3]).T, columns=['x1','x2','x3'])
In [47]: df.describe()
Out[47]: 
               x1          x2          x3
count  100.000000  100.000000  100.000000
mean     3.070225    2.028608   51.490000
std      2.001402    3.194753   27.930106
min     -4.127033    0.014330    3.000000
25%      2.040101    0.249580   25.000000
50%      3.204555    1.000613   54.500000
75%      4.434788    2.101581   73.000000
max      7.781921   18.791565   98.000000

```


## 处理字符串

```python
In [1]: s = pd.Series(['A', 'B', 'C', 'Aaba', 'Baca', np.nan, 'CABA', 'dog', 'cat'])

In [2]: s.str.lower()
Out[2]: 
0       a
1       b
2       c
3    aaba
4    baca
5     NaN
6    caba
7     dog
8     cat
dtype: object

In [3]: s.str.upper()
Out[3]: 
0       A
1       B
2       C
3    AABA
4    BACA
5     NaN
6    CABA
7     DOG
8     CAT
dtype: object

In [4]: s.str.len()
Out[4]: 
0    1.0
1    1.0
2    1.0
3    4.0
4    4.0
5    NaN
6    4.0
7    3.0
8    3.0
dtype: float64
```

## 多表连接

类似SQL的`join`

默认是内连接

```python
In [50]: student = {'Name':['Bob','Alice','Carol','Henry','Judy','Robert','William'],
    ...:            'Age':[12,16,13,11,14,15,24],
    ...:                       'Sex':['M','F','M','M','F','M','F']}

In [51]: score = {'Name':['Bob','Alice','Carol','Henry','William'],
    ...:          'Score':[75,35,87,86,57]}

In [52]: df_student = pd.DataFrame(student)

In [53]: df_score = pd.DataFrame(score)

In [54]: stu_score1 = pd.merge(df_student, df_score, on='Name')

In [55]: stu_score1
Out[55]: 
      Name  Age Sex  Score
0      Bob   12   M     75
1    Alice   16   F     35
2    Carol   13   M     87
3    Henry   11   M     86
4  William   24   F     57
```

还可以左连接、右连接、外连接

```python
# 左连接
In [56]: stu_score2 = pd.merge(df_student, df_score, on='Name',how='left')

In [57]: stu_score2
Out[57]: 
      Name  Age Sex  Score
0      Bob   12   M   75.0
1    Alice   16   F   35.0
2    Carol   13   M   87.0
3    Henry   11   M   86.0
4     Judy   14   F    NaN
5   Robert   15   M    NaN
6  William   24   F   57.0

# 右连接
In [59]: stu_score3 = pd.merge(df_student, df_score, on='Name',how='right')

In [60]: stu_score3
Out[60]: 
      Name  Age Sex  Score
0      Bob   12   M     75
1    Alice   16   F     35
2    Carol   13   M     87
3    Henry   11   M     86
4  William   24   F     57

# 外连接
In [63]: stu_score4 = pd.merge(df_student, df_score, on='Name',how='outer')
In [65]: stu_score4
Out[65]: 
      Name  Age Sex  Score
0      Bob   12   M   75.0
1    Alice   16   F   35.0
2    Carol   13   M   87.0
3    Henry   11   M   86.0
4     Judy   14   F    NaN
5   Robert   15   M    NaN
6  William   24   F   57.0
```


## 数据透视表

```python
In [66]: df = pd.DataFrame({'A': ['one', 'one', 'two', 'three'] * 3,
    ...: 'B': ['A', 'B', 'C'] * 4,
    ...: 'C': ['foo', 'foo', 'foo', 'bar', 'bar', 'bar'] * 2,
    ...: 'D': np.random.randn(12),
    ...: 'E': np.random.randn(12)})

In [67]: df
Out[67]: 
        A  B    C         D         E
0     one  A  foo  0.587385 -0.631974
1     one  B  foo -0.445773 -0.933491
2     two  C  foo  0.260970  0.549515
3   three  A  bar  0.794878  0.712311
4     one  B  bar  1.356132  0.196722
5     one  C  bar  0.018715 -3.491726
6     two  A  foo  0.996970  0.905736
7   three  B  foo  1.249336  0.199805
8     one  C  foo -0.797634 -1.270725
9     one  A  bar  0.508636  0.393941
10    two  B  bar -0.990498 -1.230368
11  three  C  bar  0.353513 -0.263934

In [68]: pd.pivot_table(df, values='D', index=['A', 'B'], columns=['C'])
Out[68]: 
C             bar       foo
A     B                    
one   A  0.508636  0.587385
      B  1.356132 -0.445773
      C  0.018715 -0.797634
three A  0.794878       NaN
      B       NaN  1.249336
      C  0.353513       NaN
two   A       NaN  0.996970
      B -0.990498       NaN
      C       NaN  0.260970

```

## 缺失值处理

默缺失值会设置成`NaN`，但是如果需要参与运算需要我们对数据进行处理。

```python
In [67]: df = pd.DataFrame([[1,2,3],[3,4,np.nan],
    ...:                   [12,23,43],[55,np.nan,10],
    ...:                   [np.nan,np.nan,np.nan],[np.nan,1,2]],
    ...:                   columns=['a1','a2','a3'])
```

### 删除法

```python
In [68]: df.dropna()
Out[68]: 
     a1    a2    a3
0   1.0   2.0   3.0
2  12.0  23.0  43.0

```

### 填充数据

```python
# 填0值
In [69]: df.fillna(0)
Out[69]: 
     a1    a2    a3
0   1.0   2.0   3.0
1   3.0   4.0   0.0
2  12.0  23.0  43.0
3  55.0   0.0  10.0
4   0.0   0.0   0.0
5   0.0   1.0   2.0

# 用前一个数据填充 df.fillna(method='bfill') 后一个值
In [70]: df.fillna(method='ffill')
Out[70]: 
     a1    a2    a3
0   1.0   2.0   3.0
1   3.0   4.0   3.0
2  12.0  23.0  43.0
3  55.0  23.0  10.0
4  55.0  23.0  10.0
5  55.0   1.0   2.0

# 变量填充
In [71]: df.fillna({'a1':100,'a2':200,'a3':300})
Out[71]: 
      a1     a2     a3
0    1.0    2.0    3.0
1    3.0    4.0  300.0
2   12.0   23.0   43.0
3   55.0  200.0   10.0
4  100.0  200.0  300.0
5  100.0    1.0    2.0
```


## 数据打乱

```python
df = df.sample(frac=0.3) # frac 返回的比例 0.3代表30%
df.sample(frac=0.1).reset_index(drop=True) # 重建索引
```
