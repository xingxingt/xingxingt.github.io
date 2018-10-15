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
    SparkContext作为spark的主入口类，SparkContext表示一个spark集群的链接,它会用在创建RDD,计数器以及广播变量
    在Spark集群；
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
    TaskScheduler是一个接口，根据具体的cluster manager的不同会有不同的实现，standalone模式下具体的实现
    是TaskSchedulerImpl;
    SchedulerBackend是一个接口，根据具体的ClusterManager的不同会有不同的实现，Standalone模式下具体的实现
    是SparkDeploySchedulerBackend；

### 深入三大核心对象
    下面我们从源代码层面看下三大核心对象是如何在SparkContext中产生的；
    根据下面代码可以看到DAGScheduler，TaskScheduler,ScheduleBackend在SparkContext内部的实例化,DAGScheduler
    在这里直接实例化出来，而TaskScheduler,ScheduleBackend则在createTaskScheduler()方法中根据集群类型来实例化,
    因为这跟随后的任务调度有关;
    
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw8wtvpns2j318a0g43zg.jpg)  
  
    再进入createTaskScheduler()方法,在这个方法内是根据集群以什么方式启动的来实例出相应的TaskScheduler，
    ScheduleBackend的子类；

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw8wz0uu9lj31ho12itbb.jpg)

    我们以Standalone模式为例讲解,参看下图源码，可见TaskScheduler实例的子类为TaskSchedulerImpl，SchedulerBackend的
    实例化的子类是SparkDeploySchedulerBackend;
       
![](https://ws1.sinaimg.cn/large/006tNbRwgy1fw8x01u0foj31aa0aa3yx.jpg)    

    我们随后会单独讲解TaskSchedulerImpl和DAGScheduler,现在我们先专注于SparkDeploySchedulerBackend；
    SparkDeploySchedulerBackend核心功能:
    1,负责与Master连接注册当前程序！
	2,接收集群中为当前应用程序分配的计算资源，Executor的注册并且管理Executors！
	3,负责发送Task到具体的Executor执行！
    4,它的父类CoarseGrainedSchedulerBackend中的DriverEndpoint就是Spark应用程序中的Driver;
    
    我们进入SparkDeploySchedulerBackend代码中，首先看它的start()方法；
    
![](https://ws2.sinaimg.cn/large/006tNbRwly1fw8y7mw9ayj31j011iq5t.jpg)
    
    在SparkDeploySchedulerBackend的Start()方法中，super.start()其实执行的是它的父类CoarseGrainedSchedulerBackend
    的start方法，如下图代码所示,
    可以看到执行了createDriverEndpoint(properties)方法，我们进入看下，在createDriverEndpoint()方法中实例化了一个
    DriverEndpoint，而DriverEndpoint其实是一个消息循环体,其实这个DriverEndPoint就是Spark应用程序中的Driver，他内
    部可以接收处理其他组件发来的消息；
    
![](https://ws2.sinaimg.cn/large/006tNbRwly1fw8y9m1mvij31f80fkjs0.jpg)
![](https://ws1.sinaimg.cn/large/006tNbRwly1fw8ycr5zt5j31he04o0sw.jpg)
![](https://ws2.sinaimg.cn/large/006tNbRwly1fw8yen1zd4j31km0s440f.jpg)

    在SparkDeploySchedulerBackend的Start()方法中执行完super.start()后，进行一系列的参数配置后，就开始了Spark
    应用程序的处理,如下图所示:
    在这里实例化出APPClient对象,并调用client.start()方法,进入APPClient的start()方法,在这个方法里new了一个
    ClientEndpoint实例,其实ClientEndpoint他也是一个消息循环体;

![](https://ws1.sinaimg.cn/large/006tNbRwly1fw8yqooo9fj31go0jkjta.jpg)
![](https://ws1.sinaimg.cn/large/006tNbRwly1fw8yw7h4hzj31fi06o0sx.jpg)

    再看ClientEndpoint的onStart()方法的,这里有个重要的地方，之前我也很疑惑就是Application是如何注册给当前的集群中的
    master,在onStart()中执行registerWithMaster(1)，向master注册Application;
    如下图代码所示:
    这个跟worker向Master注册差不多，向每个Master发送RegisterApplication的消息,Master接收到注册消息并处理;
  
![](https://ws1.sinaimg.cn/large/006tNbRwly1fw8zclo83ej31040ewq38.jpg)
![](https://ws2.sinaimg.cn/large/006tNbRwly1fw8zeie575j31i00oo75p.jpg)
![](https://ws3.sinaimg.cn/large/006tNbRwly1fw8zh6gmcwj31h00raq4g.jpg)
![](https://ws1.sinaimg.cn/large/006tNbRwgy1fw8zo23vqmj318k0j0myc.jpg)

    回过头来再看SparkDeploySchedulerBackend的Start()方法,具体完成了Driver的启动和Application的注册;
    在Application提交时有个重要的地方，在appDesc中有个command,是这样的:
    当通过SparkDeploySchedulerBackend注册程序给Master的时候会把上述command提交给master，master发指令给Worker
    去启动Excutor所在的进程的时候加载main方法所在的入口类，就是command中的CoarseGrainedExcutorBackend，当然你也
    可以自己实现excutorBackend！在CoarseGrainedExecutorBackend中启动Executor（Excutor是先注册再实例化的）,
    Excutor通过线程池并发执行Task！关于Executor部分我们随后详细阐述；

![](https://ws2.sinaimg.cn/large/006tNbRwly1fw91zmte9ij31i20fkq4c.jpg)

    
    
    
    
    
    
    
