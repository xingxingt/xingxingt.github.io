---
layout:     post
title:      SparkStreaming源码分析起始篇
subtitle:   SparkStreaming源码分析起始
date:       2018-09-06
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>SparkStreaming源码分析起始
> 

### SparkStreaming开端
    SparkStreaming作为spark的流数据处理框架，并且SparkStreaming以spark-core作为底层，并在spark-core之上
    做了封装，那么SparkStreaming是如何工作的？并且是如何将SparkStreaming Api转化为SparkCore的呢？接下来的
    文章我们通过分析SparkStreaming的内部源码来解读它的工作原理；
    
下图是sparkStreaming-kafka的一个分析案例图，可以直接参考此图作为源码分析的起点:
![](https://ws3.sinaimg.cn/large/006tNbRwly1fw2xhlzdzej31kw0kd0w1.jpg)
    
