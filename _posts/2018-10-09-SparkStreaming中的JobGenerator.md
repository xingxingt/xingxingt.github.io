---
layout:     post
title:      谈谈SparkStreaming中的JobGenerator
subtitle:   谈谈SparkStreaming中的JobGenerator
date:       2018-10-08
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>SparkStreaming中的JobGenerator篇
> 

### JobGenerator概述
    主要作用就是生成SparkStreaming Job 并且驱动checkpoint的产生和清理Dstream的元数据
    This class generates jobs from DStreams as well as drives checkpointing and cleaning
    up DStream metadata.

### JobGenerator是如何实例化的并且如何启动
其实是在JobScheduler这个类中进行初始化的
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw1z8tmtkkj31k40h0abl.jpg)
并且在JobScheduler这个类启动的时候也调用了JobGenerator的Start方法
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw1zbcyk62j31ie0w240k.jpg)
这个是JobGenerator的start方法，在这个方法中实例化了一个消息循环体，并启动了这个消息循环体(EventLoop[JobGeneratorEvent])
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw1zcaean3j31ks0w20u1.jpg)
其实在JobGenerator中有两个比较重要的成员，一个是定时器Timer，另一个是消息循环体EventLoop
![](https://ws1.sinaimg.cn/large/006tNbRwgy1fw1zgol1rvj31je0883yw.jpg)
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw1zh6t261j31iw05kglt.jpg)    



