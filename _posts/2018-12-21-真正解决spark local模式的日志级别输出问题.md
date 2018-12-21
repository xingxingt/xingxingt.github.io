---
layout:     post
title:      真正解决spark local模式的日志级别输出问题
subtitle:   真正解决spark local模式的日志级别输出问题,亲测，靠谱
date:       2018-12-21
author:     XINGXING
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Spark
---

>
>真正解决spark local模式的日志级别输出问题
> 

在IDEA中开发Spark程序，程序一执行密密麻麻的Info日志一大堆，这让人很恶心，如下图：
![](https://ws3.sinaimg.cn/large/006tNbRwgy1fyedm312lej32d00mi77j.jpg)


很早以前就解决过一次，不过谷歌百度都没有真正的解决这个问题，试过无数遍的将log4j.properties文件放在工程的resources目录下都没用，绝望之际打开spark
