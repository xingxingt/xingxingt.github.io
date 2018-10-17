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

    ok，我们已经知道了在DAGScheduler中的消息事件是如何处理的，那么我们还是言归正传，继续看在SubmitJob的方法中使用
    eventProcessLoop发送一个JobSubmitted消息给自己，也就是在doOnReceive方法中找到JobSubmitted事件，在此方法中
    又继续调用了dagScheduler.handleJobSubmitted方法；如下图源代码所示:

![](https://ws1.sinaimg.cn/large/006tNbRwly1fwazeilze8j31kq056q38.jpg)

    那我们就进入handleJobSubmitted方法，我们先看下此方法中的finalStage = newResultStage(....)代码,在这里要说一下
    在一个DAG中最后一个Stage叫做resultStage,而前面的所有stage都叫做shuffleMapStage;而newResultStage(....)方法
    就是根据提供的jobId生成一个ResultStage,如下图源码所示:

![](https://ws1.sinaimg.cn/large/006tNbRwly1fwaziw935hj31fg0wy40d.jpg)

    那我们就要看下ResultStage是如何生成的，我们可以看到,在newResultStage方法中先通过getParentStagesAndId方法获取
    ResultStage的所有父stage，然后在new出一个ResultStage实例来;
    紧接着我们把代码追踪到getParentStages方法中,这个方法可以根据提供的RDD创建一个父stage的列表，我们再来剖析下这个方法; 
    在这个方法中先实例出两个数据结构parents,visited和waitingForVisit,parents是用来存放所有父类stage的数据集，而
    visited使用来存储已被访问的RDD，而waitingForVisit则是等待被访问的RDD数据集;
    在下面代码中,先将传入的RDD放入到waitingForVisit数据集中，然后循环waitingForVisit中所有的RDD，每次循环调用visit
    方法。在visit方法中它利用RDD的dependencies从后向前建立依赖关系，在遍历RDD的dependencies时如果是shufDep就生成
    一个getShuffleMapStage放入到parents数据集中，如果不是就将该dependencie对应的RDD放入到waitingForVisit中，等待
    下一次遍历，最终该方法返回一个父stage的数据集parents给newResultStage方法；
    而且在newResultStage中new出ResultStage,并将stage的数据集parents存放于该ResultStage中;

![](https://ws3.sinaimg.cn/large/006tNbRwly1fwazr8g0dej31fa0g80tg.jpg)
![](https://ws1.sinaimg.cn/large/006tNbRwly1fwb00ckoawj31f006aq37.jpg)
![](https://ws1.sinaimg.cn/large/006tNbRwly1fwb018bh0qj31dk10kwg9.jpg)
     
     
     
