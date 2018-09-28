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

### 再看下job的产生者jobGenerator是如何将生成的job传递给JobScheduler的
![](https://ws4.sinaimg.cn/large/006tNc79gy1fvp3ixpgh0j31fm0osmz6.jpg)

### JobScheduler处理提交上来的job，并将job存放在jobSet的数据结构中
![](https://ws2.sinaimg.cn/large/006tNc79ly1fvp55329opj31cu0dudgo.jpg)

