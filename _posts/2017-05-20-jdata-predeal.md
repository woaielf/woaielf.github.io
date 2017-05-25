---
layout: post
title: "【实战】电商数据篇（一）数据初探"
categories: 数据科学
tags: 实例分析 
author: ZY
---

* content
{:toc}

**本文是电商数据实战系列第一篇。**
数据来源：[京东JData算法大赛-高潜用户购买意向预测](http://www.datafountain.cn/#/competitions/247/intro/)。数据集以京东商城真实的用户、商品和行为数据（脱敏后）为基础。一直苦于缺乏实践，正好拿这个项目练练手。接下来的几篇实战博文，我会尽可能仔细的写下一些体会，结合之前学习的数据分析工具（以Python及相关包为主），探索一下电商数据的世界。由于本人也是数据分析的小白，希望在实践中和大家一起成长。




# Update Log
- 2017/05/20

# 数据描述

以下数据描述来自[数据介绍](http://www.datafountain.cn/#/competitions/247/data-intro)。

**符号定义：**
- S：提供的商品全集；
- P：候选的商品子集（JData_Product.csv），P是S的子集；
- U：用户集合；
- A：用户对S的行为数据集合；
- C：S的评价数据

**训练数据部分：**
（这一部分就是我们拿到的数据）
提供2016-02-01到2016-04-15日用户集合U中的用户，对商品集合S中部分商品的行为、评价、用户数据；提供部分候选商品的数据P。

**预测数据部分：**
2016-04-16到2016-04-20用户是否下单P中的商品，每个用户只会下单一个商品。

**数据表（四个）：**
1. 用户数据
2. 商品数据
3. 评价数据
4. 行为数据

**我们接下来将逐一查看四个表各个字段值及特点。思考后续该如何挖掘数据最大价值？**

# 准备工作


```python
# 数据处理
import pandas as pd
import numpy as np

# 可视化
import matplotlib.pyplot as plt
import seaborn as sns

# 辅助类
import re 
import os

%matplotlib inline
```


```python
# 所有数据表
ACTION_2 = "JData_Action_201602.csv"
ACTION_3 = "JData_Action_201603.csv"
ACTION_4 = "JData_Action_201604.csv"
COMMENT = "JData_Comment.csv"
PRODUCT = "JData_Product.csv"
USER = "JData_User.csv"
NEW_USER = "JData_User_New.csv"
```


```python
# 控制小数位数显示
pd.options.display.float_format = '{:,.3f}'.format
```

# 数据探查

初步估计：用户数据、商品数据、评价数据可直接读入内存，行为数据很大（我的电脑内存 8G），只能先读入一部分看看。

## 用户数据

![png](https://raw.githubusercontent.com/woaielf/woaielf.github.io/master/_posts/Pic/1705/jdata/0520-1.png)


```python
df = pd.read_csv('data_ori/' + USER, header=0, encoding="gbk")
```


```python
df.shape
```




    (105321, 5)




```python
df.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>user_id</th>
      <th>age</th>
      <th>sex</th>
      <th>user_lv_cd</th>
      <th>user_reg_tm</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>200001</td>
      <td>56岁以上</td>
      <td>2.000</td>
      <td>5</td>
      <td>2016-01-26</td>
    </tr>
    <tr>
      <th>1</th>
      <td>200002</td>
      <td>-1</td>
      <td>0.000</td>
      <td>1</td>
      <td>2016-01-26</td>
    </tr>
    <tr>
      <th>2</th>
      <td>200003</td>
      <td>36-45岁</td>
      <td>1.000</td>
      <td>4</td>
      <td>2016-01-26</td>
    </tr>
    <tr>
      <th>3</th>
      <td>200004</td>
      <td>-1</td>
      <td>2.000</td>
      <td>1</td>
      <td>2016-01-26</td>
    </tr>
    <tr>
      <th>4</th>
      <td>200005</td>
      <td>16-25岁</td>
      <td>0.000</td>
      <td>4</td>
      <td>2016-01-26</td>
    </tr>
  </tbody>
</table>
</div>



最终数据要处理成矩阵形式，所以我们先把中文字符去除。

### age



字段描述：-1表示未知


```python
df.age.value_counts()
```




    26-35岁    46570
    36-45岁    30336
    -1        14412
    16-25岁     8797
    46-55岁     3325
    56岁以上      1871
    15岁以下         7
    Name: age, dtype: int64




```python
def convert(age):
    if age == '-1':
        return -1
    elif age == '15岁以下':
        return 0
    elif age == '16-25岁':
        return 1
    elif age == '26-35岁':
        return 2
    elif age == '36-45岁':
        return 3
    elif age == '46-55岁':
        return 4
    elif age == '56岁以上':
        return 5
    else:
        return -1
```


```python
df.age = df.age.map(convert)
```


```python
x = df.age.value_counts().sort_index()
x
```




    -1    14415
     0        7
     1     8797
     2    46570
     3    30336
     4     3325
     5     1871
    Name: age, dtype: int64




```python
sns.barplot(x=x.index, y=x.values)
```




    <matplotlib.axes._subplots.AxesSubplot at 0x1cf9db75080>




![png](https://raw.githubusercontent.com/woaielf/woaielf.github.io/master/_posts/Pic/1705/jdata/0520_output_20_1.png)



确实是中年人的购买力最旺盛呢。
接着我们需要计算一个指标user_reg_diff：即用户注册时间（参照为数据表中最早注册用户）。需要计算时间差值。


```python
df['user_reg_tm'] = pd.to_datetime(df['user_reg_tm'])
min_date = min(df['user_reg_tm'])
df['user_reg_diff'] = [i for i in (df['user_reg_tm'] - min_date).dt.days]
```


```python
df.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>user_id</th>
      <th>age</th>
      <th>sex</th>
      <th>user_lv_cd</th>
      <th>user_reg_tm</th>
      <th>user_reg_diff</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>200001</td>
      <td>5</td>
      <td>2.000</td>
      <td>5</td>
      <td>2016-01-26</td>
      <td>4,607.000</td>
    </tr>
    <tr>
      <th>1</th>
      <td>200002</td>
      <td>-1</td>
      <td>0.000</td>
      <td>1</td>
      <td>2016-01-26</td>
      <td>4,607.000</td>
    </tr>
    <tr>
      <th>2</th>
      <td>200003</td>
      <td>3</td>
      <td>1.000</td>
      <td>4</td>
      <td>2016-01-26</td>
      <td>4,607.000</td>
    </tr>
    <tr>
      <th>3</th>
      <td>200004</td>
      <td>-1</td>
      <td>2.000</td>
      <td>1</td>
      <td>2016-01-26</td>
      <td>4,607.000</td>
    </tr>
    <tr>
      <th>4</th>
      <td>200005</td>
      <td>1</td>
      <td>0.000</td>
      <td>4</td>
      <td>2016-01-26</td>
      <td>4,607.000</td>
    </tr>
  </tbody>
</table>
</div>



现在再来看一下数据表的基本信息。


```python
df.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 105321 entries, 0 to 105320
    Data columns (total 6 columns):
    user_id          105321 non-null int64
    age              105321 non-null int64
    sex              105318 non-null float64
    user_lv_cd       105321 non-null int64
    user_reg_tm      105318 non-null datetime64[ns]
    user_reg_diff    105318 non-null float64
    dtypes: datetime64[ns](1), float64(2), int64(3)
    memory usage: 4.8 MB
    

可以看到三个字段有缺失值。

### sex 字段

字段描述：0表示男，1表示女，2表示保密


```python
df.ix[df.sex.isnull(), 'sex']
```




    34072   nan
    38905   nan
    67704   nan
    Name: sex, dtype: float64




```python
df.sex.value_counts()
```




    2.000    54735
    0.000    42846
    1.000     7737
    Name: sex, dtype: int64



将缺失值用 2 填充。


```python
df.ix[df.sex.isnull(), 'sex'] = 2
```


```python
df.sex.value_counts()
```




    2.000    54738
    0.000    42846
    1.000     7737
    Name: sex, dtype: int64



### 输出数据


```python
df.to_csv('data_ori/' + NEW_USER, index=False) 
```

## 商品数据

![png](https://raw.githubusercontent.com/woaielf/woaielf.github.io/master/_posts/Pic/1705/jdata/0520-2.png)



```python
df = pd.read_csv('data_ori/' + PRODUCT, header=0) 
```


```python
df.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>sku_id</th>
      <th>a1</th>
      <th>a2</th>
      <th>a3</th>
      <th>cate</th>
      <th>brand</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>10</td>
      <td>3</td>
      <td>1</td>
      <td>1</td>
      <td>8</td>
      <td>489</td>
    </tr>
    <tr>
      <th>1</th>
      <td>100002</td>
      <td>3</td>
      <td>2</td>
      <td>2</td>
      <td>8</td>
      <td>489</td>
    </tr>
    <tr>
      <th>2</th>
      <td>100003</td>
      <td>1</td>
      <td>-1</td>
      <td>-1</td>
      <td>8</td>
      <td>30</td>
    </tr>
    <tr>
      <th>3</th>
      <td>100006</td>
      <td>1</td>
      <td>2</td>
      <td>1</td>
      <td>8</td>
      <td>545</td>
    </tr>
    <tr>
      <th>4</th>
      <td>10001</td>
      <td>-1</td>
      <td>1</td>
      <td>2</td>
      <td>8</td>
      <td>244</td>
    </tr>
  </tbody>
</table>
</div>




```python
df.shape
```




    (24187, 6)




```python
df.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 24187 entries, 0 to 24186
    Data columns (total 6 columns):
    sku_id    24187 non-null int64
    a1        24187 non-null int64
    a2        24187 non-null int64
    a3        24187 non-null int64
    cate      24187 non-null int64
    brand     24187 non-null int64
    dtypes: int64(6)
    memory usage: 1.1 MB
    


```python
df.describe()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>sku_id</th>
      <th>a1</th>
      <th>a2</th>
      <th>a3</th>
      <th>cate</th>
      <th>brand</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>count</th>
      <td>24,187.000</td>
      <td>24,187.000</td>
      <td>24,187.000</td>
      <td>24,187.000</td>
      <td>24,187.000</td>
      <td>24,187.000</td>
    </tr>
    <tr>
      <th>mean</th>
      <td>85,398.737</td>
      <td>2.177</td>
      <td>0.939</td>
      <td>1.180</td>
      <td>8.000</td>
      <td>435.864</td>
    </tr>
    <tr>
      <th>std</th>
      <td>49,238.799</td>
      <td>1.176</td>
      <td>0.970</td>
      <td>1.046</td>
      <td>0.000</td>
      <td>225.749</td>
    </tr>
    <tr>
      <th>min</th>
      <td>6.000</td>
      <td>-1.000</td>
      <td>-1.000</td>
      <td>-1.000</td>
      <td>8.000</td>
      <td>3.000</td>
    </tr>
    <tr>
      <th>25%</th>
      <td>42,476.000</td>
      <td>1.000</td>
      <td>1.000</td>
      <td>1.000</td>
      <td>8.000</td>
      <td>214.000</td>
    </tr>
    <tr>
      <th>50%</th>
      <td>85,616.000</td>
      <td>3.000</td>
      <td>1.000</td>
      <td>1.000</td>
      <td>8.000</td>
      <td>489.000</td>
    </tr>
    <tr>
      <th>75%</th>
      <td>127,774.000</td>
      <td>3.000</td>
      <td>2.000</td>
      <td>2.000</td>
      <td>8.000</td>
      <td>571.000</td>
    </tr>
    <tr>
      <th>max</th>
      <td>171,224.000</td>
      <td>3.000</td>
      <td>2.000</td>
      <td>2.000</td>
      <td>8.000</td>
      <td>922.000</td>
    </tr>
  </tbody>
</table>
</div>




```python
len(df.sku_id.value_counts())
```




    24187



确认没有重复值，后续导入MySQL可以作为主键。

## 评价数据

![png](https://raw.githubusercontent.com/woaielf/woaielf.github.io/master/_posts/Pic/1705/jdata/0520-3.png)


```python
df = pd.read_csv('data_ori/' + COMMENT, header=0)
df.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>dt</th>
      <th>sku_id</th>
      <th>comment_num</th>
      <th>has_bad_comment</th>
      <th>bad_comment_rate</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2016-02-01</td>
      <td>1000</td>
      <td>3</td>
      <td>1</td>
      <td>0.042</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2016-02-01</td>
      <td>10000</td>
      <td>2</td>
      <td>0</td>
      <td>0.000</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2016-02-01</td>
      <td>100011</td>
      <td>4</td>
      <td>1</td>
      <td>0.038</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2016-02-01</td>
      <td>100018</td>
      <td>3</td>
      <td>0</td>
      <td>0.000</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2016-02-01</td>
      <td>100020</td>
      <td>3</td>
      <td>0</td>
      <td>0.000</td>
    </tr>
  </tbody>
</table>
</div>




```python
df.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 558552 entries, 0 to 558551
    Data columns (total 5 columns):
    dt                  558552 non-null object
    sku_id              558552 non-null int64
    comment_num         558552 non-null int64
    has_bad_comment     558552 non-null int64
    bad_comment_rate    558552 non-null float64
    dtypes: float64(1), int64(3), object(1)
    memory usage: 21.3+ MB
    


```python
len(df.sku_id.value_counts())
```




    46546




```python
df.dt.value_counts().sort_index()
```




    2016-02-01    46546
    2016-02-08    46546
    2016-02-15    46546
    2016-02-22    46546
    2016-02-29    46546
    2016-03-07    46546
    2016-03-14    46546
    2016-03-21    46546
    2016-03-28    46546
    2016-04-04    46546
    2016-04-11    46546
    2016-04-15    46546
    Name: dt, dtype: int64



这里似乎发现，其实这个数据集就是对46546个数据条目，每周进行一次统计。

## 行为数据

![png](https://raw.githubusercontent.com/woaielf/woaielf.github.io/master/_posts/Pic/1705/jdata/0520-4.png)


```python
# 读入1000行。
df = pd.read_csv('data_ori/' + ACTION_2, header=0, nrows=10000)
```


```python
df.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>user_id</th>
      <th>sku_id</th>
      <th>time</th>
      <th>model_id</th>
      <th>type</th>
      <th>cate</th>
      <th>brand</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>266,079.000</td>
      <td>138778</td>
      <td>2016-01-31 23:59:02</td>
      <td>nan</td>
      <td>1</td>
      <td>8</td>
      <td>403</td>
    </tr>
    <tr>
      <th>1</th>
      <td>266,079.000</td>
      <td>138778</td>
      <td>2016-01-31 23:59:03</td>
      <td>0.000</td>
      <td>6</td>
      <td>8</td>
      <td>403</td>
    </tr>
    <tr>
      <th>2</th>
      <td>200,719.000</td>
      <td>61226</td>
      <td>2016-01-31 23:59:07</td>
      <td>nan</td>
      <td>1</td>
      <td>8</td>
      <td>30</td>
    </tr>
    <tr>
      <th>3</th>
      <td>200,719.000</td>
      <td>61226</td>
      <td>2016-01-31 23:59:08</td>
      <td>0.000</td>
      <td>6</td>
      <td>8</td>
      <td>30</td>
    </tr>
    <tr>
      <th>4</th>
      <td>263,587.000</td>
      <td>72348</td>
      <td>2016-01-31 23:59:08</td>
      <td>nan</td>
      <td>1</td>
      <td>5</td>
      <td>159</td>
    </tr>
  </tbody>
</table>
</div>




```python
len(df.model_id.value_counts())
```




    37




```python
df.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 10000 entries, 0 to 9999
    Data columns (total 7 columns):
    user_id     10000 non-null float64
    sku_id      10000 non-null int64
    time        10000 non-null object
    model_id    5511 non-null float64
    type        10000 non-null int64
    cate        10000 non-null int64
    brand       10000 non-null int64
    dtypes: float64(2), int64(4), object(1)
    memory usage: 547.0+ KB
    

没看懂这个model_id是什么，似乎有很多缺失值。

# 总结

至此，我们已经对每个数据表进行了初步的探索和分析。因为数据表很大，一般的个人PC可能不足以支撑。在下一篇中，我将结合关系型数据库管理系统——MySQL来创建训练数据用的表。

# Reference
> [京东JData算法大赛](http://www.datafountain.cn/projects/jdata_beta/) <br>
[foursking1/jd](https://github.com/foursking1/jd) <br>
[ottsion/JData](https://github.com/ottsion/JData)
[daoliker/JData](https://github.com/daoliker/JData)