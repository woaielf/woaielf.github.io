---
layout: post
title: "Trim Galore用法及参数考量"
categories: 生物信息
tags: 生信软件 NGS
author: ZY
---

* content
{:toc}

近期一直在学习如何处理二代测序数据，第一步就是将从测序公司拿到的raw data去接头处理。公司拿来的数据附带了clean data以及去接头的说明文件，对文件进行分析后我自己重新实现了一下trim的过程。




## 软件说明

#### 版本信息
Trim Galore version: 0.4.1

Cutadapt version: 1.11

#### 参数概览
这篇文章只讨论了部分参数（与我的数据相关的部分）。其余的可以参考官方文档。[Trim_Galore_User_Guide](http://www.bioinformatics.bbsrc.ac.uk/projects/trim_galore/)

- -q/--quality <INT> 控制的质量分数阈值

- --length <INT> 丢弃小于此长度的读段

- -e 允许的错误率

- --stringency Overlap with adapter sequence required to trim a sequence.

- -o 输出文件路径


## 案例分析

#### 测序数据
Illumina Hiseq3000

RNA-seq 

paired-end

#### 代码展示
```
/.../trim_galore /.../*_R1.fastq /.../*_R2.fastq -q 25 --length 50 -e 0.1 --stringency 5 -o /.../ -a adapter1 -a2 adapter2 --paired
```

#### 软件输出

![](https://raw.githubusercontent.com/woaielf/woaielf.github.io/master/_posts/Pic/7-trim.png)
