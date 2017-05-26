---
layout: post
title: "【实战】电商数据篇（二）MySQL 实战指南"
categories: 数据科学
tags: 实例分析 
author: ZY
---

* content
{:toc}

**本文是电商数据实战系列第二篇。**关于数据的情况，请参考[【实战】电商数据篇（一）数据初探](https://woaielf.github.io/2017/05/20/jdata-predeal/)。

本篇中，我主要是为了解决以下的问题：
- MySQL 建表
- 练习 SQL 查询：通过提问——解答的模式
**本文不是以解决数据模型问题为中心，更多的是对之前 SQL&MySQL 学习的演练。可以根据需要跳过部分章节，不影响后续数据处理的理解。**可通过[【笔记】SQL & MySQL](https://woaielf.github.io/2017/05/04/sql/) 回顾基础知识。





结合使用 MySQL，一方面是「行为数据」太大，无法读入内存处理；另一方面，一直苦于没有熟练 SQL 的机会。四个表，多个字段，可以衍生出很多查询的问题。

# Update Log
- 2017/05/25

# MySQL 建表

首先我用 Python 分别跑了每个表每个字段的最大长度，供 MySQL 建表时参考。

```
CREATE DATABASE JData;
USE JData;
```

## 用户

```
CREATE TABLE user(
    user_id INT NOT NULL,
    age TINYINT NOT NULL default -1,
    sex TINYINT NOT NULL default -1,
    user_lv_cd TINYINT NOT NULL default -1,
    user_reg_dt DATE,
    user_reg_diff INT NOT NULL default -1,
    PRIMARY KEY(user_id)
) CHARSET='utf8' ENGINE=INNODB;

LOAD DATA LOCAL INFILE "D:/02/data_ori/JData_User_New.txt" INTO TABLE user LINES TERMINATED BY '\r\n' (user_id, age, sex, user_lv_cd, user_reg_dt, user_reg_diff);
```

## 商品

```
CREATE TABLE product(
    sku_id INT NOT NULL,
    attr1 TINYINT NOT NULL,
    attr2 TINYINT NOT NULL,
    attr3 TINYINT NOT NULL,
    cate TINYINT NOT NULL,
    brand SMALLINT NOT NULL,
    PRIMARY KEY(sku_id)
) CHARSET='utf8' ENGINE=INNODB;

LOAD DATA LOCAL INFILE "D:/02/data_ori/JData_Product.txt" INTO TABLE product LINES TERMINATED BY '\r\n' (sku_id, attr1, attr2, attr3, cate, brand);
```

## 评论

```
CREATE TABLE comment(
    id INT auto_increment NOT NULL,
    dt DATE,
    sku_id INT NOT NULL,
    comment_num TINYINT NOT NULL,
    has_bad_comment TINYINT NOT NULL,
    bad_comment_rate FLOAT NOT NULL,
    PRIMARY KEY(id,sku_id)
) CHARSET='utf8' ENGINE=INNODB;

LOAD DATA LOCAL INFILE "D:/02/data_ori/JData_Comment.txt" into table comment LINES TERMINATED BY '\r\n' (dt, sku_id, comment_num, has_bad_comment, bad_comment_rate);
```

## 行为

```
CREATE TABLE actions(
    id INT auto_increment NOT NULL,
    user_id INT NOT NULL,
    sku_id INT NOT NULL,
    action_time DATETIME NOT NULL,
    model_id VARCHAR(10) NOT NULL DEFAULT -1,
    action_type TINYINT,
    cate TINYINT,
    brand SMALLINT,
    PRIMARY KEY(id)
) CHARSET='utf8' ENGINE=INNODB;

LOAD DATA LOCAL INFILE "D:/02/data_ori/JData_Action_201602.txt" INTO TABLE actions LINES TERMINATED BY '\r\n' (user_id, sku_id, action_time, model_id, action_type, cate, brand);

LOAD DATA LOCAL INFILE "D:/02/data_ori/JData_Action_201603.txt" INTO TABLE actions LINES TERMINATED BY '\r\n' (user_id, sku_id, action_time, model_id, action_type, cate, brand);

LOAD DATA LOCAL INFILE "D:/02/data_ori/JData_Action_201604.txt" INTO TABLE actions LINES TERMINATED BY '\r\n' (user_id, sku_id, action_time, model_id, action_type, cate, brand);

ALTER TABLE actions ADD INDEX user_id (user_id);
ALTER TABLE actions ADD INDEX sku_id (sku_id);
```

# MySQL 实操


```python
import pymysql as sql
from collections import Counter
import numpy as np
import pandas as pd
```

Python 连接 MySQL 并查看一下数据表的情况。


```python
db = sql.connect("localhost", "root", "", "jdata")
```


```python
cursor = db.cursor()
```


```python
cursor.execute('SHOW TABLES')
cursor.fetchall()
```




    (('actions',), ('comment',), ('product',), ('test',), ('user',))




```python
cursor.execute('SELECT COUNT(*) FROM actions')
cursor.fetchall()
```




    ((50601736,),)



一些基本操作过后，我们来尝试回答以下的问题（**都是我即兴编造的题目，据此练习一下SQL查询，没有需求的朋友可以直接跳过这部分**）：

## user


```python
# user 表的字段特征
cursor.execute('SHOW COLUMNS FROM user')
cursor.fetchall()
```




    (('user_id', 'int(11)', 'NO', 'PRI', None, ''),
     ('age', 'tinyint(4)', 'NO', '', '-1', ''),
     ('sex', 'tinyint(4)', 'NO', '', '-1', ''),
     ('user_lv_cd', 'tinyint(4)', 'NO', '', '-1', ''),
     ('user_reg_dt', 'date', 'YES', '', None, ''),
     ('user_reg_diff', 'int(11)', 'NO', '', '-1', ''))




```python
# 注册时间超过1000天的用户的等级分布
cursor.execute('SELECT user_lv_cd, COUNT(*) FROM user WHERE user_reg_diff>1000 GROUP BY user_lv_cd')
cursor.fetchall()
```




    ((1, 2663), (2, 9660), (3, 24562), (4, 32326), (5, 35985))




```python
# 用户等级 & 性别
cursor.execute('SELECT user_lv_cd, sex, COUNT(*) FROM user GROUP BY user_lv_cd, sex')
cursor.fetchall()
```




    ((1, 0, 258),
     (1, 1, 72),
     (1, 2, 2336),
     (2, 0, 1290),
     (2, 1, 436),
     (2, 2, 7935),
     (3, 0, 5776),
     (3, 1, 1411),
     (3, 2, 17376),
     (4, 0, 12982),
     (4, 1, 2481),
     (4, 2, 16880),
     (5, 0, 22543),
     (5, 1, 3337),
     (5, 2, 10208))




```python
# 统计每年的注册人数
cursor.execute('SELECT YEAR(user_reg_dt), COUNT(*) FROM user GROUP BY YEAR(user_reg_dt)')
cursor.fetchall()
```




    ((0, 3),
     (2003, 7),
     (2004, 40),
     (2005, 58),
     (2006, 123),
     (2007, 423),
     (2008, 1538),
     (2009, 3352),
     (2010, 7148),
     (2011, 13286),
     (2012, 17528),
     (2013, 15545),
     (2014, 17199),
     (2015, 22064),
     (2016, 7007))




```python
# 最大最小注册时间差
cursor.execute('SELECT DATEDIFF(MAX(user_reg_dt), MIN(user_reg_dt)) AS diff_time FROM user WHERE user_reg_dt != "0000-00-00"')
cursor.fetchall()
```




    ((4911,),)



结果应该和下面相同：


```python
cursor.execute('SELECT MAX(user_reg_diff) from user')
cursor.fetchall()
```




    ((4911,),)




```python
# 2016-04-01 到 2016-05-30 之间注册的用户数
cursor.execute('SELECT CONCAT(sex, "-", COUNT(*)) FROM user WHERE user_reg_dt BETWEEN "2016-04-01" AND "2016-05-30" GROUP BY sex')
cursor.fetchall()
```




    ((b'0-36',), (b'1-19',), (b'2-339',))



## Product & Comment


```python
# product 表有多少不同 brand & cate？
cursor.execute('SELECT COUNT(DISTINCT brand) FROM product')
print(cursor.fetchall())
cursor.execute('SELECT COUNT(DISTINCT cate) FROM product')
print(cursor.fetchall())
```

    ((102,),)
    ((1,),)
    

观察上面可知，数据抽取的仅是一个类目（e.g.食品/电子/服饰），下面有102个品牌。


```python
# 看看 sku_id 在每个表中的信息条数
cursor.execute('SELECT COUNT(DISTINCT sku_id) FROM product')
print("product: %d" % cursor.fetchall()[0][0])
cursor.execute('SELECT COUNT(DISTINCT sku_id) FROM comment')
print("comment: %d" % cursor.fetchall()[0][0])
cursor.execute('SELECT COUNT(DISTINCT sku_id) FROM actions')
print("actions: %d" % cursor.fetchall()[0][0])
```

    product: 24187
    comment: 46546
    actions: 28710
    

这里很清楚的能看出并不是所有sku_id都附带了商品详情的。接下来看看互相之间交集大小：


```python
# product vs comment
cursor.execute('SELECT COUNT(DISTINCT product.sku_id) FROM comment JOIN product WHERE comment.sku_id = product.sku_id ')
cursor.fetchall()
```




    ((6830,),)




```python
# actions vs comment
cursor.execute('SELECT COUNT(DISTINCT comment.sku_id) FROM actions JOIN comment WHERE comment.sku_id = actions.sku_id ')
cursor.fetchall()
```




    ((20067,),)




```python
# actions vs product
cursor.execute('SELECT COUNT(DISTINCT sku_id) FROM actions WHERE sku_id in (SELECT DISTINCT sku_id FROM product)')
cursor.fetchall()
```




    ((3938,),)




```python
# product vs comment vs actions
cursor.execute('SELECT COUNT(DISTINCT sku_id) FROM actions WHERE sku_id in (SELECT DISTINCT product.sku_id FROM product INNER JOIN comment WHERE product.sku_id = comment.sku_id)')
```




    1




```python
cursor.fetchall()
```




    ((2825,),)



## actions


```python
cursor.execute('SELECT COUNT(*) FROM actions')
cursor.fetchall()
```




    ((50601736,),)



注意：部分查询出结果会非常慢……即使之前我对两个字段建索引了（我的电脑性能较弱，花了差不多两天的时间(⊙﹏⊙)b）。


```python
cursor.execute('SELECT COUNT(*), cate FROM actions GROUP BY cate')
cursor.fetchall()
```




    ((11088350, 4),
     (5731403, 5),
     (6554984, 6),
     (4365031, 7),
     (18128055, 8),
     (4005882, 9),
     (627359, 10),
     (100672, 11))



回想一下上面对product表的分析，cate字段只有值8……这个有点麻烦，下一篇文章再来考虑。


```python
# 看一下用户活动的时间区间
cursor.execute('SELECT MAX(action_time), MIN(action_time) FROM actions')
cursor.fetchall()
```




    ((datetime.datetime(2016, 4, 15, 23, 59, 59),
      datetime.datetime(2016, 1, 31, 23, 59, 2)),)




```python
# 活动类别统计
cursor.execute('SELECT action_type, COUNT(*) FROM actions GROUP BY action_type')
cursor.fetchall()
```




    ((1, 18981373),
     (2, 575418),
     (3, 256053),
     (4, 48252),
     (5, 109896),
     (6, 30630744))



# 结尾彩蛋

我们回想一下这次比赛的任务：**预测2016-04-16到2016-04-20用户是否下单P中的商品，每个用户只会下单一个商品。**我们不妨试着直接用MySQL来实现一下。也就是，不使用任何模型，直接拟定一些假设条件，通过SQL调出数据。

比如说，我就限定条件（均为“且关系”）：
- 2016-04-10以后
- 用户对商品有「加购、关注」行为
- 同时排除「删购、下单」行为
- 取行为数最多的商品
- 且商品包含在 P 中

你也可以选择其他条件，反正我们就只是做一个初步筛选。

1: 浏览 2: 加购 3: 删除 4: 购买 5: 收藏 6: 点击；
```
# 创建视图1：初步按照时间、活动类型筛选；商品包含在 P 中；过滤掉无活动的用户
CREATE VIEW check1 AS 
SELECT user_id, actions.sku_id, count(*) as counts 
FROM actions
JOIN product
WHERE (actions.sku_id = product.sku_id) AND (action_type IN (2, 5)) AND (action_time > '2016-04-10')
GROUP BY user_id, actions.sku_id
HAVING counts > 1;

# 创建视图2：这里我就直接选取计数最大的条目，不细究了
CREATE VIEW check2 AS
SELECT user_id, sku_id FROM check1
WHERE counts = (SELECT MAX(counts) FROM check1 AS temp WHERE temp.user_id = check1.user_id);

# 创建视图3：需要排除的用户及商品
CREATE VIEW exclude AS 
SELECT user_id, actions.sku_id 
FROM actions
JOIN product
WHERE (actions.sku_id = product.sku_id) AND (action_time > '2016-04-10') AND (action_type in (3,4))
GROUP BY user_id, actions.sku_id;

# 选取数据
SELECT check2.user_id, check2.sku_id 
FROM check2 LEFT JOIN exclude
ON check2.user_id = exclude.user_id
WHERE check2.sku_id != exclude.sku_id
INTO OUTFILE 'D:/02/test1.csv'
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY ''
LINES TERMINATED BY '\n';

```

# 总结

本文在MySQL建表的基础上，尝试做了各种查询，包括函数的使用，视图的创建。后续在分析数据时，可以更自由的使用。

# Reference
> [京东JData算法大赛](http://www.datafountain.cn/projects/jdata_beta/) <br>
[foursking1/jd](https://github.com/foursking1/jd) <br>
[ottsion/JData](https://github.com/ottsion/JData)
[daoliker/JData](https://github.com/daoliker/JData)