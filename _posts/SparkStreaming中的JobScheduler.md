---
layout:     post
title:      谈谈SparkStreaming中的JobScheduler
subtitle:   谈谈SparkStreaming中的JobScheduler
date:       2018-09-08
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>SparkStreaming中的JobScheduler篇
> 

### 首先看下JobScheduler这个类是在什么时候被实例化的
打开StreamingContext代码可见：
![](https://ws1.sinaimg.cn/large/006tNc79gy1fvp3olf2mnj31bm0fat9g.jpg)

