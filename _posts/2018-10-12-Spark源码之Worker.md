---
layout:     post
title:      Spark源码之Worker
subtitle:   Spark源码之Worker介绍
date:       2018-10-12
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>Spark源码之Worker介绍篇
> 

### Worker介绍
    Worker作为工作节点,一般Driver以及Executor都会在这Worker上分布;
    
### Worker代码概览   
    1,Worker继承了ThreadSafeRpcEndpoint,所以本身就是一个消息循环体,可以直接跟其他组件进行通信；
    2,内部封装一堆数据结构，用于记录存储Driver,Executor，Application等信息；
    3,Worker内部对自身的资源维护；
    4,与其他组件通信的通信结构；
    5,与Master之间的协助等
    见下图源码:
 
![](https://ws2.sinaimg.cn/large/006tNbRwly1fw59b9benfj313a0hiq3l.jpg) 
![](https://ws4.sinaimg.cn/large/006tNbRwly1fw59a67bqzj31d809yaax.jpg)    
![](https://ws3.sinaimg.cn/large/006tNbRwly1fw59dv0qggj317e07o74g.jpg)
![](https://ws3.sinaimg.cn/large/006tNbRwly1fw59f17r33j31ku05y0sx.jpg)
![](https://ws1.sinaimg.cn/large/006tNbRwgy1fw59igrfffj31i20viq4o.jpg)

### Worker内部分析
    还会从Woker的初始化和启动开始，如下代码所示:

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fw59oipxelj31is0q40ud.jpg)
    
    再看Worker的启动onStart()方法,因为Worker是主动向Master注册的，所以在WorKer启动的方法内就直接
    向Master注册；
    
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw59qdsovdj31j40mm75u.jpg) 

    进入registerWithMaster() 方法，
    
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw59rwfxogj31fo0t4wg5.jpg)  

    继续往下走,进入tryRegisterAllMasters()，为什么会有tryRegisterAllMasters()方法呢？因为如何Master是
    HA的情况下就会出现多个Master，所以Worker要将它的信息注册给每个Master,在下面的代码中可见，在Worker里用了
    一个线程池registerMasterThreadPool生成一条线程去与Master通信；
   
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw59uf2e05j31i20met9t.jpg)   

    继续进入registerWithMaster()方法，在这个方法内你会看到具体想Master发送的注册请求,以及对请求的响应状态的处理;
    Worker向Master注册，在[Spark源码之Master中已经详细叙述过]，这里就不再累赘；
    
![](https://ws4.sinaimg.cn/large/006tNbRwly1fw59z8apk1j31fs0iygmw.jpg) 

    进入handleRegisterResponse()方法，看下具体是如何处理这些Worker向Master注册后的事件；
    这个方法里处理的事件：
    1,Worker向Master注册成功,在worker注册成功后会调用changeMaster(masterRef, masterWebUiUrl)将当前的Master
      信息保存在Woker内部的数据结构中；
    2,Worker向Master注册失败;
    3,Master出现StandBy的情况;
    
![](https://ws4.sinaimg.cn/large/006tNbRwly1fw5a3gd6bsj31jk142q5p.jpg) 
![](https://ws2.sinaimg.cn/large/006tNbRwly1fw5a9dp99vj31g20c8js2.jpg)

### Woker的其他工作
    在Worker还维护着Driver和Executor的变化，如下图所示：
 
 ![](https://ws1.sinaimg.cn/large/006tNbRwly1fw5ade254tj31ka08u74i.jpg)
 
 
####  Worker
