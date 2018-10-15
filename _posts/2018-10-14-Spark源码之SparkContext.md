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
    SparkContext特点:
	1,Spark的程序编写时基于SparkContext的，具体包括两方面:
	  Spark编程的核心基础--RDD，是由SparkContext来最初创建（第一个RDD一定是由SparkContext来创建的）；
	  Spark程序的调度优化优势基于SparkContext；
	2,Spark程序的注册是通过SparkContext实例化的时候产生的对象来完成的（其实是SchedulerBackend来注册程序）
	3,Spark程序运行的时候通过Cluster Manager获取具体的计算资源，计算资源的获取也是通过SparkContext产生的对象来申请的（
      其实是SchedulerBackend来获取计算资源的）;
	4,sparkContext崩溃或者结束的时候整个Spark程序也就结束！
    
### SparkContext内容探索
    
