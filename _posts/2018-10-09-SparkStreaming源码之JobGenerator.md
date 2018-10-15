---
layout:     post
title:      SparkStreaming源码之JobGenerator
subtitle:   SparkStreaming源码之JobGenerator
date:       2018-10-08
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>SparkStreaming源码之JobGenerator篇
> 

### JobGenerator概述
    主要作用就是生成SparkStreaming Job 并且驱动checkpoint的产生和清理Dstream的元数据
    This class generates jobs from DStreams as well as drives checkpointing and cleaning
    up DStream metadata.

### JobGenerator是如何实例化的并且如何启动
其实是在JobScheduler这个类中进行初始化的<br/>
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw1z8tmtkkj31k40h0abl.jpg)
并且在JobScheduler这个类启动的时候也调用了JobGenerator的Start方法
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw1zbcyk62j31ie0w240k.jpg)
这个是JobGenerator的start方法，在这个方法中实例化了一个消息循环体，并启动了这个消息循环体(EventLoop[JobGeneratorEvent])
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw1zcaean3j31ks0w20u1.jpg)
### JobGenerator内部探究
其实在JobGenerator中有两个比较重要的成员，一个是定时器Timer，Timer根据Interval time不断向自己发送GenerateJobs消息，
另一个是消息循环体EventLoop
![](https://ws1.sinaimg.cn/large/006tNbRwgy1fw1zgol1rvj31je0883yw.jpg)
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw1zh6t261j31iw05kglt.jpg)    
再来看下消息循环体的具体内容
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw2032p852j31am0fet9r.jpg)
我们主要看下generateJobs方法
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fw204zd7o5j31k00ou40c.jpg)
在allocateBlocksToBatch方法中获取根据interval time划分的block块数据

    jobScheduler.receiverTracker.allocateBlocksToBatch(time) // allocate received blocks to batch
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw20bzi8kwj31hm104gny.jpg)
在获取到属于该job的数据后开始产生job
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fw20f77htij31ka0nojt8.jpg)
![](https://ws3.sinaimg.cn/large/006tNbRwly1fw20gofb5nj316g0g2wfa.jpg)
再向下看就是Dstream的generatorJob的方法了，其实这个方法会被子类的实现所覆盖，例如print操作产生的ForeachDstream
![](https://ws1.sinaimg.cn/large/006tNbRwly1fw20i872a6j31c60o8jsl.jpg)
Dstream的子类ForeachDstream实现方法,可见是通过从后向前回溯的方法来生成一个job，特别是Some(new Job(time, jobFunc))
中的jobFunc方法，就是自定义的输出方法，可以去看下Dstream里面的Print方法是如何传入的
![](https://ws1.sinaimg.cn/large/006tNbRwly1fw20kohhecj31cw0ds3z0.jpg)
ok!基于interval time生成的job就已经ok了，接下来就是如何将生成的job向下传递了，根据代码可见，最后是将生成的job交给JobScheduler进行处理；
![](https://ws4.sinaimg.cn/large/006tNbRwly1fw20omd7agj31hc0nsac0.jpg)

***

### 上面介绍了JobGenerator这个角色，接下来再引申一个问题，为什么SparkStreaming不会处理到半条数据的情况？
    
    不会出现处理半条数据的原因有两点:
    1，receiver接收数据是一条一条的接收的；
    2，receive的接收数据，向driver端汇报数据，为batch分配数据这三者其实时间不是一致的

    其实存在着这样的一种情况:
    Batch1接收了1000条数据，batch2接收了500条数据，  在处理时会出现batch1处理了900条数据，batch2处理了600条数据；
    原因：
    在给job分配数据的时候使用了synchronized修饰，那么就会造成batch在分配数据时并不一定能分配1000条数；
    所以revice接收到的这条数据分配给哪个batch是由何时获取锁的时间来决定的；
![](https://ws2.sinaimg.cn/large/006tNbRwly1fw20x55rmoj31i80mgwfx.jpg)


#### OK! jobGenrator介绍完毕,其实比较重要的就是数据的分配和job的产生；






