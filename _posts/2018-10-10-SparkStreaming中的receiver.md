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
    6，调用startReceiver(receiver, executors)方法，启动receiver；
    详细源代码见下图:
![](https://ws3.sinaimg.cn/large/006tNbRwly1fw35cnvhwwj31kc0dmgmf.jpg)

    进入startReceiver方法，该方法主要是启动一个receiver和对应的executor；
    查询源码可见，receiver是被封装成了一个rdd，然后以Job的形式通过SparkContext启动，并且用future监听这个reveiver 
    job的状态，因为每个job都是一个线程，一旦启动失败，或者执行异常都会重新发送self.send(RestartReceiver(receiver))
    消息给自己，重新启动该reveicer；
![](https://ws3.sinaimg.cn/large/006tNbRwly1fw363ki62ij31eo14g77c.jpg)

    这个receiver rdd中重要的是startReceiverFunc这个方法，这个方法里面详细描述着receiver启动和它工作的内容，
    在这里它会实例化一个supervisor；
![](https://ws1.sinaimg.cn/large/006tNbRwly1fw36daeznwj31bi0pimyg.jpg)    
    
    然后调用ReceiverSupervisor的start方法    
![](https://ws1.sinaimg.cn/large/006tNbRwly1fw36h5ybs7j30vm07e0sp.jpg)   

    我们先看下ReceiverSupervisorImpl实现的onStart()方法，接着进入start方法，可以看到在start方法中做了两件事，
    1，启动blockIntervalTimer，它是一个定时器，具体执行的是updateCurrentBuffer方法，前面说过SocketInputDStream
       的例子，通过网络接受数据，并将数据store到currentBuffer中，而updateCurrentBuffer方法的作用就是将currentBuffer
       中的数据转换成block，然后存储到blocksForPushing数据结构中；
    2，启动blockPushingThread，这个线程具体执行的方法是keepPushingBlocks()，在keepPushingBlocks()方法中
       将生成的block通过blockManager进行存储;
![](https://ws1.sinaimg.cn/large/006tNbRwgy1fw36js1psgj319u04s74c.jpg)   
![](https://ws3.sinaimg.cn/large/006tNbRwgy1fw36l37x6uj31ii0h2wfa.jpg)
![](https://ws3.sinaimg.cn/large/006tNbRwgy1fw36pef9wpj318o0wmmz2.jpg)
![](https://ws1.sinaimg.cn/large/006tNbRwly1fw372ekuhhj31kw12rq58.jpg)

    ok，ReceiverSupervisor的onStart()方法介绍完毕了，回过头来再看下startReceiver()方法，在这个方法内启动了receiver；
![](https://ws4.sinaimg.cn/large/006tNbRwly1fw37606iasj31kg0n00to.jpg)   
     
    我们再来看下receiver.onStart()具体的实现，还以SocketReceiver为列，可以看到receive方法中通过socket
    一条一条的接收数据，并将接收到的数据做存储，其实存储就是存储在上面所说的currentBuffer中了，这样就衔接上了，
    reveiver不停的接受数据并存储在内存中，ReceiverSupervisor将内存中的数据转换成block并用blockManager存
    储在内存或者磁盘中；
 ![](https://ws4.sinaimg.cn/large/006tNbRwly1fw3796cxrwj312g0a4q36.jpg)  
 ![](https://ws2.sinaimg.cn/large/006tNbRwly1fw37afts3oj31ak13ggni.jpg)
     
 
 #### 至此receiver介绍完毕；    
