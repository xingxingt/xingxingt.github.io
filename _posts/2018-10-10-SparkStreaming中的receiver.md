---
layout:     post
title:      SparkStreaming中的receiver
subtitle:   SparkStreaming中的receiver
date:       2018-10-10
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>SparkStreaming中的receiver篇
> 

### ReceiverTracker简介
    ReceiverTracker管理ReceiverInputDStreams接受者的执行，接受输入流数据并以block的形式将输入的数据
    以block的形式存储；
    
### ReceiverTracker的实例化和启动
    ReceiverTracker是在JobScheduler的start方法中进行初始化和启动的，详细代码见下图;
![](https://ws2.sinaimg.cn/large/006tNbRwly1fw2z8mgh7bj31kw0vxjtg.jpg)
    
### ReceiverTracker的内部源码
    从start方法入手，在这个方法里面，先实例化了一个消息循环体ReceiverTrackerEndpoint，然后在调用了
    launchReceivers()方法去启动receiver,见下图源代码;
![](https://ws2.sinaimg.cn/large/006tNbRwly1fw34yq04yuj318w0j8wfc.jpg) 
    
    进入launchReceivers()方法,此方法是从ReceiverInputDStreams中获取receiver，并且将这些receiver
    分布到不同的worker节点上运行,而ReceiverInputDStreams就是DstreamGraph中的inputStreams，而
    runDummySparkJob()这个方法其实运行的就是一个空的job，它的目的就是为了能够让所有的slave节点注册进来
    从而能够获取到最多的资源，之后就用endPoint给自己发送StartAllReceivers的消息，详细源代码见下图:
![](https://ws1.sinaimg.cn/large/006tNbRwly1fw351w4z4zj31260g8t99.jpg)   
    
    下面是StartAllReceivers消息体执行的详细内容，
    1，schedulingPolicy.scheduleReceivers(receivers, getExecutors)方法是为了让receiver在最大化数据
       本地性的需求下均匀的分步在各个executor上；
    2，循环遍历每个receiver，采用逐个启动每个receiver的方法；
    3，val executors = scheduledLocations(receiver.streamId)获取该receiver分步在哪些executor上；
    4，updateReceiverScheduledExecutors(receiver.streamId, executors)更新维护receiverTrackingInfos
       数据结构，以receiver id为key，以receiver info为value的数据结构；
    5，receiverPreferredLocations(receiver.streamId) = receiver.preferredLocation，将该receiver的
       位置信息存放在receiverPreferredLocations数据结构中；
    6，调用startReceiver(receiver, executors)方法，启动receiver        
![](https://ws3.sinaimg.cn/large/006tNbRwly1fw35cnvhwwj31kc0dmgmf.jpg)
