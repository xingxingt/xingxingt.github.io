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
    既然Executor是运行在CoarseGrainedExecutorBackend进程中，那就先说下这个CoarseGrainedExecutorBackend;
    首先我们应该知道CoarseGrainedExecutorBackend是什么时候被实例出来的,我们在【spark源码之SparkContext】中
    介绍过AppLication的注册和Driver的产生，在APPClient实例化时候传入了一个command，而这个command就是
    CoarseGrainedExecutorBackend这个类，如下图所示，Application在注册时把这个command也提交给了Master，
    master发指令给Worker去启动Excutor所在的进程的时候加载main方法所在的入口类，就是command中的
    CoarseGrainedExcutorBackend;

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fwa25yznh3j31j80f4ta4.jpg)
![](https://ws4.sinaimg.cn/large/006tNbRwly1fwa33vgtr8j31ks10w0v8.jpg)

    Master在启动一个Excutor所在的进程的时候加载了CoarseGrainedExecutorBackend的main方法，我们进入main方法，
    先进行进行了一系列的参数初始化之后进入了run()方法，在run()方法中进行环境参数配置后启动RPC通信，并且实例化出
    CoarseGrainedExecutorBackend；
    
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw7lehyoi9j31hq13wabt.jpg)   
![](https://ws1.sinaimg.cn/large/006tNbRwgy1fw7lhb3t8dj31kw0msq4o.jpg)    

    
    



#### 需要注意的是,我们现在主要说的是spark的StandAlone模式;


