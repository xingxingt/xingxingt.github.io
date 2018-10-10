---
layout:     post
title:      SparkStreaming中的receiver
subtitle:   SparkStreaming中的receiver
date:       2018-10-10
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>SparkStreaming中的receiver篇
> 

### ReceiverTracker简介
    ReceiverTracker管理ReceiverInputDStreams接受者的执行，接受输入流数据并以block的形式将输入的数据
    以block的形式存储；
    
### ReceiverTracker的实例化和启动
    ReceiverTracker是在JobScheduler的start方法中进行初始化和启动的，详细代码见下图;
![](https://ws2.sinaimg.cn/large/006tNbRwly1fw2z8mgh7bj31kw0vxjtg.jpg)
    
