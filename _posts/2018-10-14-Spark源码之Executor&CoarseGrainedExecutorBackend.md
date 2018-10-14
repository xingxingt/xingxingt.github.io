---
layout:     post
title:      Spark源码之Executor&CoarseGrainedExecutorBackend
subtitle:   Spark源码之Executor&CoarseGrainedExecutorBackend介绍
date:       2018-10-14
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>Spark源码之Executor&CoarseGrainedExecutorBackend介绍篇
> 

### CoarseGrainedExecutorBackend和Executor的关系
    我们先说下CoarseGrainedExecutorBackend和Executor这两者的关系，CoarseGrainedExecutorBackend比较直观
    因为我们在启动Spark集群运行任务通过JPS命令,可以看到有一个CoarseGrainedExecutorBackend这样的进程，其实
    CoarseGrainedExecutorBackend就是一个进程，而Executor则是一个实例对象，并且Executor是运行在
    CoarseGrainedExecutorBackend进程中的；再者CoarseGrainedExecutorBackend和Executor是一一对应的；
    
### CoarseGrainedExecutorBackend内幕
    既然Executor是运行在CoarseGrainedExecutorBackend进程中，那就先说下这个CoarseGrainedExecutorBackend，
    进入它的源代码，在这个类具有入口main方法，我们进入main方法，发现它进行了一系列的参数初始化之后进入了run()方法，
    在run()方法中进行环境参数配置后启动RPC通信，并且实例化出CoarseGrainedExecutorBackend；
    
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw7lehyoi9j31hq13wabt.jpg)   
![](https://ws1.sinaimg.cn/large/006tNbRwgy1fw7lhb3t8dj31kw0msq4o.jpg)    

    
