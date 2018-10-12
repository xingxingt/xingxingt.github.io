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
