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
    
    
