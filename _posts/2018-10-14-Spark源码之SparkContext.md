---
layout:     post
title:      Spark源码之SparkContext
subtitle:   Spark源码之SparkContext介绍
date:       2018-10-14
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>Spark源码之SparkContext介绍篇
> 

### SparkContext介绍
    SparkContext作为spark的主入口类，SparkContext表示一个spark集群的链接,它会用在创建RDD,计数器以及广播变量在Spark集群；
    SparkContext特性:
    1,Spark的程序编写时基于SparkContext的，具体包括两方面:
      Spark编程的核心基础--RDD，是由SparkContext来最初创建（第一个RDD一定是由SparkContext来创建的）；
      Spark程序的调度优化优势基于SparkContext；
    2,Spark程序的注册是通过SparkContext实例化的时候产生的对象来完成的（其实是SchedulerBackend来注册程序）
    3,Spark程序运行的时候通过Cluster Manager获取具体的计算资源，计算资源的获取也是通过SparkContext产生的对象来申请的
      (其实是SchedulerBackend来获取计算资源的）;
    4,sparkContext崩溃或者结束的时候整个Spark程序也就结束！
    
### SparkContext构建的三大核心对象
    SparkContext构建的顶级三大核心对象：DAGScheduler，TaskScheduler，ScheduleBackend；
    DAGScheduler是面向job的stage的高层调度器；
    TaskScheduler是一个接口，根据具体的cluster manager的不同会有不同的实现，standalone模式下具体的实现是TaskSchedulerImpl;
    SchedulerBackend是一个接口，根据具体的ClusterManager的不同会有不同的实现，Standalone模式下具体的实现是SparkDeploySchedulerBackend；

### 深入三大核心对象
    下面我们从源代码层面看下三大核心对象是如何在SparkContext中产生的；
    根据下面代码可以看到DAGScheduler，TaskScheduler,ScheduleBackend在SparkContext内部的实例化,DAGScheduler在这里直接
    实例化出来，而TaskScheduler,ScheduleBackend则在createTaskScheduler()方法中根据集群类型来实例化,因为这跟随后的任务调度有关;
    
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw8wtvpns2j318a0g43zg.jpg)  
  
    再进入createTaskScheduler()方法,在这个方法内是根据集群以什么方式启动的来实例出相应的TaskScheduler，
    ScheduleBackend的子类；

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw8wz0uu9lj31ho12itbb.jpg)

    我们以Standalone模式为例讲解,参看下图源码，可见TaskScheduler实例的子类为TaskSchedulerImpl，SchedulerBackend的实例化的
    子类是SparkDeploySchedulerBackend;
       
![](https://ws1.sinaimg.cn/large/006tNbRwgy1fw8x01u0foj31aa0aa3yx.jpg)    

    我们随后会单独讲解TaskSchedulerImpl和DAGScheduler,现在我们先专注于SparkDeploySchedulerBackend，
    
    
    
