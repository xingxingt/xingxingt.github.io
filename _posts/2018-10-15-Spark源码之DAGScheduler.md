---
layout:     post
title:      Spark源码之DAGScheduler
subtitle:   Spark源码之DAGScheduler介绍
date:       2018-10-15
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>Spark源码之DAGScheduler介绍篇
> 

    Spark Application中的RDD经过一系列的Transformation操作后由Action算子导致了SparkContext.runjob的执行,
    之后执行DAGScheduler.runJob(),最终导致了DAGScheduler中的submitJob的执行,在DAGScheduler中完成sparkJob
    的DAG划分,并将生成的TaskSet交给taskScheduler处理，如下图所示:

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fwaee7mjwfj314i0k8q81.jpg)

### 深入DAGScheduler源码

    我们从RDD的Action操作产生的SparkContext.runjob说起,在SparkContext.runjob()中最终调用了
    dagScheduler.runJob()方法；如下图所示:
    
![](https://ws3.sinaimg.cn/large/006tNbRwly1fwaz2ykk33j318i09iq37.jpg)  
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fwaetpjg0dj31js0omta7.jpg)

    接着看SparkContext.runjob()方法,在方法里面调用了submitJob()方法，并且返回一个JobWaiter监听submitJob的
    结果，并对结果做出相应的处理;
    
![](https://ws4.sinaimg.cn/large/006tNbRwly1fwaz4k1b11j31jq0tu406.jpg)

    进入submitJob方法，如下图所示，先生成一个jobId，紧接着使用eventProcessLoop发送一个JobSubmitted的消息，那我
    们就要看下这个eventProcessLoop是什么了；
    
![](https://ws1.sinaimg.cn/large/006tNbRwgy1fwaf3ej89vj31ey13smzc.jpg)

    查看源码发现eventProcessLoop是一个消息循环体，而且他还继承了EventLoop，再看下EventLoop的代码，发现EventLoop
    是一个时间处理器，在内部使用BlockingQueue去存储接受到的消息事件，用一个守护线程去执行onReceive,而onReceive方法
    在DAGSchedulerEventProcessLoop中已经被重写，而在onReceive方法中调用doOnReceive方法做具体的事件处理;

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fwaf5bggkzj31ee04i3yl.jpg)
![](https://ws2.sinaimg.cn/large/006tNbRwly1fwaz79gvz9j31kw0yvtau.jpg)
![](https://ws2.sinaimg.cn/large/006tNbRwly1fwaz67tr1dj31km14075x.jpg)



