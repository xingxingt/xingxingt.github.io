---
layout:     post
title:      Spark源码之Master
subtitle:   Spark源码之Master介绍
date:       2018-10-11
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>Spark源码之Master介绍篇
> 

### Master介绍
    Master作为资源管理和分配的组件，所以今天我们重点来看Spark Core中的Master如何实现资源的注册，状态的维护以及调度分配;
    
### Master内部代码概览
    1,Master继承了ThreadSafeRpcEndpoint所以它本身也就是一个消息循环体，可以接受其他组件发送的消息并处理;
    2,在Master内部维护了一大堆数据结构，用于存放资源组件的元数据信息；
    3,处理Master所管理的一些组件；参看下图源码：
    
![](https://ws1.sinaimg.cn/large/006tNbRwgy1fw4a5x36w4j31ew0qw40x.jpg)    
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fw4a7f443jj31kw05emxa.jpg)
![](https://ws3.sinaimg.cn/large/006tNbRwly1fw4aj0bfw1j31ec0najsh.jpg)

### Master对其他组件的注册处理
    1，Master接受注册的对象主要是Worker，Driver，Application；而Excutor不会注册给master，Excutor是注册给Driver
       中的SchedulerBackend的；
    2，再者Worker是再启动后主动向Master注册的，所以如果生产环境中加入新的worker到正在运行的spark集群中，此时不需要重新
       启动spark集群就能够使用新加入的worker，以提升处理能力；

    我们以Worker注册为例，
    Master在接收到worker注册到的请求后，首先会判断一下当前的master是否是standby模式，如果是就不处理；
    然后会判断当前Master的内存数据结构idToWorker中是否已经存在该worker，如果有的话就不会重复注册；
    Master如果决定接受注册的worker，首先会创建WorkerInfo对象来保存注册当前的worker信息；
    
![](https://ws2.sinaimg.cn/large/006tNbRwly1fw4awvc3lzj31fa0xwtb2.jpg)

    然后在执行registerWorker(worker)方法执行具体的注册流程，如果worker的状态是DEAD的话就直接过滤掉，
    对于UNKOWN状态的内容调用removeWorker进行清理（也包括清理该worker下的Excutors和Drivers）,其实这
    里就是检查该worker的状态然后将woker信息存放在Master维护的数据结构中；参看下图代码:
    
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw4b1f8l0fj31k0110abx.jpg)    
    
    1，如果registerWoker成功，则将worker信息进行持久化，可以将worker信息持久化在zookeeper中,或者系统文件
    (fileSystem),用户自定义的文件persistenceEngine中，以便于处理灾难后的数据恢复； 
    2，worker信息持久化后遍回复worker完成注册的消息；
    3，接着进行schudule，给worker分配资源，schudule我们单独分析；
    参看下图两段代码：

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fw4b6oajglj31du0xkdib.jpg)
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fw4balje8ij31c20qm40g.jpg)

### Master对资源组件状态变化的处理：

     如下源码中对Driver状态的处理，先检查Driver的状态，如果Driver出现error,finished,killed,failed状态，
     则将此Driver移除掉，在处理executor的状态变化时也是先检查executor的状态，然后在进行移除或者资源重分配的
     操作；详细看下图源码：
     
![](https://ws4.sinaimg.cn/large/006tNbRwly1fw4c49p7t5j31i60be3yz.jpg)
![](https://ws1.sinaimg.cn/large/006tNbRwly1fw4c4rtrnqj31j612edi7.jpg)
