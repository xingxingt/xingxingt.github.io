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
>SparkStreaming中的Dstream和DstreamGraph篇
> 


## 先谈DstreamGraph，
#### 在DstreamGraph中有两个ArrayBuffer，
    private val inputStreams = new ArrayBuffer[InputDStream[_]]()
    private val outputStreams = new ArrayBuffer[DStream[_]]()
    
#### inputStreams的作用就是存放一个流的inputDstream，例如SocketInputDStream,他是在父类InputDStream中执行具体的存放操作
![](https://ws4.sinaimg.cn/large/0069RVTdgy1fv0r8hs89ej314k0c674y.jpg)

#### 那么InputDStream接收的数据又是如何进行存储的呢?
![](https://ws3.sinaimg.cn/large/0069RVTdgy1fv0rth3yb8j31cc12otal.jpg)
![](https://ws2.sinaimg.cn/large/0069RVTdgy1fv0rucb4l7j31be0ai0t3.jpg)
#### 经过代码追踪发现接受的数据实际上是以block的形式存放，BlockGenerator以spark.streaming.blockInterval作为时间单位来生成block块，内部有一个Timer来定时生成block块：我觉得这里的RecurringTimer做的挺好，同一个Timer根据不同的callback方法来执行不同的任务，get到了新技能,点赞！
![](https://ws3.sinaimg.cn/large/0069RVTdgy1fv0ryyiv1wj31j0048mxe.jpg)
![](https://ws4.sinaimg.cn/large/0069RVTdgy1fv0rx26jfej30qm0hkmy8.jpg)

#### 上面说了inputStream,接下来看下outputStream,以print操作为例:
![](https://ws2.sinaimg.cn/large/006tNc79ly1fvp23bois9j31fk0lg3zi.jpg)
![](https://ws4.sinaimg.cn/large/006tNc79ly1fvp269ygn8j31hy08yt94.jpg)
#### 在这里将输出操作的Dstream注册进入了DstreamGraph的outputDstream中
![](https://ws1.sinaimg.cn/large/006tNc79ly1fvp27z5xixj31gi0byaan.jpg)


#### 还有就是Dstream中outputStreamArray中的action是如何触发job的,其实在jobGenertor中通过定时器RecurringTimer来实现的，那就再来看下这个定时器，RecurringTimer是在JobGneretor中进行实例化的
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

***


## 再看DStream

#### 第一：inputDStream是如何产生RDD的，还是以SocketInputDStraem为例:
![](https://ws4.sinaimg.cn/large/0069RVTdgy1fv0s7r256jj31im0y2wgl.jpg)

#### 第二:Transform级别的DStream:例如:FlatMappedDStream,在它的comput方法中,使用parent.getOrCompute来获取父Dstream产生的RDD，然后使用父Dstream产生的RDD来执行map方法（此map方法是基于RDD的map方法）；此处可以发现SparkStreaming是对SparkCore的一层抽象，而SparkStreaming的实际执行还是基于sparkCore实体来执行的；
![](https://ws1.sinaimg.cn/large/0069RVTdgy1fv0s9t5w67j314204gmx7.jpg)

#### 第三:再看Action级别的DStream: 例如:print(), 在foreachFunc方法中就是基于RDD进行操作的；
![](https://ws2.sinaimg.cn/large/0069RVTdgy1fv0sbwnnnrj31c80qgabc.jpg)
#### 而foreachDStream中的compute方法为空,是因为foreachDStream是job中最后的Action操作，而generateJob内执行的执行发放foreachFunc中执行的还是RDD的输出操作；
![](https://ws2.sinaimg.cn/large/0069RVTdgy1fv0sdl5ytwj31gs0vkq4i.jpg)

#### 在JobGenerator中,定时器RecurringTimer不停的执行triggerActionForNextInterval的callback方法
![](https://ws3.sinaimg.cn/large/0069RVTdgy1fv0sgg1rk5j30qw05wmxg.jpg)
#### callback方法具体执行的就是DStreamGraph中的generateJobs方法，
![](https://ws1.sinaimg.cn/large/0069RVTdgy1fv0sh8e4d3j30qm0aegm8.jpg)
#### DStreamGraph中的generateJobs方法执行的是DStream的generateJob方法，在此方法中最终执行的是SparkCore的runJob方法;
![](https://ws1.sinaimg.cn/large/0069RVTdgy1fv0si09bimj30qk090dg8.jpg)
#### 而Dstream的generateJob方法中调用DStream的gerorcompute,在此方法中根据时间在generatedRDDs中存储对应Time的RDD数组,其他每个DStream都有一个这样的数据结构来根据Time来存储对应的RDD；
![](https://ws2.sinaimg.cn/large/0069RVTdgy1fv0siskd3kj30qm0eswg3.jpg)

#### 接下来就是将基于RDD产生的Job提交给cluster进行执行……………

#### 总结:其实DStream只是基于RDD的一个抽象的模板,而DstreamGreaph就是生成DAG的模板，最终每个Dstream都会生成一个以time为key，RDD[T]为value的数据结构用来存储基于模板生成的RDD，SparkStreaming最终做执行操作的还是SparkCore的RDD；


















