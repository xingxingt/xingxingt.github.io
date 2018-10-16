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
    
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fwa369baktj31e80emmyl.jpg)    
![](https://ws4.sinaimg.cn/large/006tNbRwly1fwa33vgtr8j31ks10w0v8.jpg)

    Master在启动一个Excutor所在的进程的时候加载了CoarseGrainedExecutorBackend的main方法，我们进入main方法，
    先进行进行了一系列的参数初始化之后进入了run()方法，在run()方法中进行环境参数配置后启动RPC通信，并且实例化出
    CoarseGrainedExecutorBackend；
    
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw7lehyoi9j31hq13wabt.jpg)   
![](https://ws1.sinaimg.cn/large/006tNbRwgy1fw7lhb3t8dj31kw0msq4o.jpg)    

    CoarseGrainedExecutorBackend实例化出来后我们再看它的onStart()方法，在CoarseGrainedExecutorBackend启动
    后就立即向Driver注册Executor,开始着手启动Executor如下图所示;
    
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fwa39blzumj31a60o80ub.jpg)

    打开Driver的源码,找到RegisterExecutor部分，如下图所示：
    在Driver里面有一个数据结构executorDataMap，用于存储注册的Executor信息，先判断该Executor是否在该数据结构中存在，
    如果不存在则继续往下执行，然后将Executor的信息存于各种数据结构中，接下来就是调用通知CoarseGrainedExecutorBackend
    注册Executor成功，再调用makeOffers()方法。makeOffers()方法主要是给Executor分配资源，并且启动Task任务,Task的内
    容会在TaskScheduler部分叙述,我们在这里暂且不过多讲述;
        
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fwa3hlilyvj31e01600vs.jpg)
![](https://ws2.sinaimg.cn/large/006tNbRwly1fwa4yqfrcbj31fs0ea0te.jpg)

    如下图所示，CoarseGrainedExecutorBackend在接到RegisteredExecutor消息后立即实例化了一个executor对象
    
![](https://ws3.sinaimg.cn/large/006tNbRwly1fwa4tmpwgcj31g6076wet.jpg)    

#### 需要注意的是,我们现在主要说的是spark的StandAlone模式;CoarseGrainedExecutorBackend进程的产生和Executor对象的实例化都阐述完毕，最后放出这篇的分析图：
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fwa5cpgiv5j31kw0eyte9.jpg)
