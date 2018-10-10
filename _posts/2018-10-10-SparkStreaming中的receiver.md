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
    
    
