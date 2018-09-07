---
layout:     post
title:      谈谈SparkStreaming中的Dstream和DstreamGraph
subtitle:   谈谈SparkStreaming中的Dstream和DstreamGraph这些魔板类
date:       2018-09-07
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>SparkStreaming中的Dstream和DstreamGraBuffe篇
> 


## 先谈DstreamGraph，
#### 在DstreamGraph中有两个ArrayBuffer，
    private val inputStreams = new ArrayBuffer[InputDStream[_]]()
    private val outputStreams = new ArrayBuffer[DStream[_]]()
    
#### inputStreams的作用就是存放一个流的inputDstream，例如SocketInputDStream,他是在父类InputDSt中执行具体的存放操作
![](https://ws4.sinaimg.cn/large/0069RVTdgy1fv0r8hs89ej314k0c674y.jpg)

#### 那么InputDStream接收的数据又是如何进行存储的呢?
![](https://ws3.sinaimg.cn/large/0069RVTdgy1fv0rth3yb8j31cc12otal.jpg)
![](https://ws2.sinaimg.cn/large/0069RVTdgy1fv0rucb4l7j31be0ai0t3.jpg)
#### 经过代码追踪发现接受的数据实际上是以block的形式存放，BlockGenerator以spark.streaming.blockInterval来生成block块，内部有一个Timer来定时生成block块：
![](https://ws3.sinaimg.cn/large/0069RVTdgy1fv0ryyiv1wj31j0048mxe.jpg)
![](https://ws4.sinaimg.cn/large/0069RVTdgy1fv0rx26jfej30qm0hkmy8.jpg)


#### 而outputStreams是通过定时器RecurringTimer来添加，那就再来看下这个定时器，RecurringTimer是在JobGneretor中进行实例化的
![](https://ws3.sinaimg.cn/large/0069RVTdgy1fv0rd9oep8j31hy058q34.jpg)

#### 来看下RecurringTimer执行的内容
![](https://ws3.sinaimg.cn/large/0069RVTdgy1fv0rehksm8j313c0gk74m.jpg)
![](https://ws3.sinaimg.cn/large/0069RVTdgy1fv0rfm66rgj316o09i0t2.jpg)

#### 里面的callback方法是关键，我们顺着来看下callback方法执行的内容
![](https://ws2.sinaimg.cn/large/0069RVTdgy1fv0rhtj2ngj31iy044jrk.jpg)
![](https://ws4.sinaimg.cn/large/0069RVTdgy1fv0rieo6psj31120e4js6.jpg)
![](https://ws4.sinaimg.cn/large/0069RVTdgy1fv0rjmy3brj31kw0nnabw.jpg)
![](https://ws4.sinaimg.cn/large/0069RVTdgy1fv0rk9is65j316o0fymxx.jpg)

#### 在上面已经将要输出的DStream存放于DStreamGraph的outputStreams数组中，接下来就是具体的执行
![](https://ws3.sinaimg.cn/large/0069RVTdgy1fv0rqd4k96j31f80n4gmy.jpg)





