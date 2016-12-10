---
layout: post
title: "【总结】Trim Galore用法及参数考量"
categories: 生物信息
tags: 生信软件 NGS
author: ZY
---

* content
{:toc}

Trim Galore是一个非常流行的用于「去接头序列」的软件，用于处理**高通量测序得到的原始数据**。通常我们从测序公司拿到数据后，第一步就是**评估数据的质量以及对raw data去接头处理**。公司拿来的数据通常附带了clean data以及去接头的说明文件，我自己重新实现了一下trim的过程。参数都是根据公司的说明文件来设定的。




## 软件说明

### 版本信息
1. Trim Galore version: 0.4.1
2. Cutadapt version: 1.11
3. FastQC version：0.11.3

### 依赖环境
1. [FastQC](http://www.bioinformatics.bbsrc.ac.uk/projects/fastqc/)
2. [Cutadapt](https://pypi.python.org/pypi/cutadapt/)


### 软件安装
Trim Galore直接在[官网](http://www.bioinformatics.bbsrc.ac.uk/projects/download.html#trim_galore)下载解压后即可使用（perl文件，无需任何安装）。<br>


### 参数概览
这里只讨论了<font color="red">部分参数</font>（与我的数据相关的部分，数据情况请参照下面）。其余参数的设定可以参考「官方文档」（[Trim_Galore_User_Guide](http://www.bioinformatics.bbsrc.ac.uk/projects/trim_galore/)）。

- -q/--quality <INT>：控制的质量分数阈值
- --length <INT>：丢弃小于此长度的读段
- -e：允许的错误率
- --stringency：限定最少与adaptor序列重叠的碱基数（用来trim的标准）
- -o：输出文件路径


## 案例分析

### 测序数据
Illumina Hiseq3000 <br> 
Paired-end RNA-seq 

### 代码展示
```
/.../trim_galore /.../*_R1.fastq /.../*_R2.fastq -q 25 --length 50 -e 0.1 --stringency 5 -o /.../ -a adapter1 -a2 adapter2 --paired
```

### 软件输出
> Trimming mode: paired-end
<br>Trim Galore version: 0.4.1
<br>Cutadapt version: 1.11
<br>Quality Phred score cutoff: 25
<br>Quality encoding type selected: ASCII+33
<br>Adapter sequence: ...
<br>Maximum trimming error rate: 0.1 (default)
<br>Optional adapter 2 sequence (only used for read 2 of paired-end files): ...
<br>Minimum required adapter overlap (stringency): 5 bp
<br>Minimum required sequence length for both reads before a sequence pair gets removed: 50 bp

## 参考资料
> [http://www.bioinformatics.bbsrc.ac.uk/projects/trim_galore/](http://www.bioinformatics.bbsrc.ac.uk/projects/trim_galore/)

## Update Log
- 2016/11/18