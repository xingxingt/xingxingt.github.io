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
    2,在Master内部维护了一大堆数据结构，用于存放资源组件的元数据信息,例如注册的worker,application,Driver等；
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


###  资源调度分配
    Master中的资源分配尤为重要，所以我们着重查探Master是如何进行资源的调度分配的；
    Master中的Schedule()方法就是用于资源分配，Schedule()将可用的资源分配给等待的Applications,
    这个方法随时都要被调用，比如说Application的加入或者可用资源的变化
    再看Schedule具体执行的内容:
    1,先判断master的状态，如果不是alive状态，就什么都不做；
    2,将注册进来的所有worker进行shuffle,随机打乱，以便做到负载均衡；
    3,然后在打乱的worker中过滤掉状态不是alive的worker；
    4,将waitingDrivers中的worker一个一个的在worker上启动；
    5,启动Driver后才能启动executor；

![](https://ws3.sinaimg.cn/large/006tNbRwly1fw4cpiqe6aj31ho0ik3zu.jpg)  
    
    接着看下Driver是如何启动的，在launchDriver()中向worker发送一个消息
    worker.endpoint.send(LaunchDriver(driver.id, driver.desc))让woker启动一个driver线程;
    Driver启动完成将该Driver的状态改为Running状态;

![](https://ws1.sinaimg.cn/large/006tNbRwly1fw4cvui20gj31b60bcaaj.jpg)    
![](https://ws3.sinaimg.cn/large/006tNbRwgy1fw4d12tqchj31fi0mugmk.jpg)

    Ok,Driver启动完成后就可以启动Executor了，因为Executor是注册给Driver的，所以要先把Driver启动完毕；
    接着我们进入startExecutorsOnWorkers()方法中，此方法的作用就是调度和启动Executor在worker上；
    1,遍历等待分配资源的Application，并且过滤掉所有一些不需要继续分配资源的Application；
    2,过滤掉状态不为alive和资源不足的worker，并根据资源大小对workers进行排序；
    3,调用scheduleExecutorsOnWorkers()方法，指定executor是在哪些workers上启动，并返回一个为每个worker
      指定cores的数组;
   
![](https://ws3.sinaimg.cn/large/006tNbRwly1fw4djkikklj31ja0pkgns.jpg)

    具体的资源分配还是要看scheduleExecutorsOnWorkers(),进入该方法；
    为应用程序分配Executor有两种方式：
    第一种方式是尽可能在集群的所有节点的worker上分配Executor，这种方式往往会带来潜在的更好，有利于数据本地性，
    第二种是尽量在一个节点的worker上分配Executor，这样的效率可想而知非常低下；
    coresPerExecutor：每个Executor上分配的core;
    minCoresPerExecutor:每个Executor最少分配的core数;
    oneExecutorPerWorker:一个worker是否只启动一个Executor；
    memoryPerExecutor:一个Executor分配到momery大小；
    numUsable：可用的worker；    
    assignedCores：指的是在某个worker上已经被分配了多少个cores；
    assignedExecutors指的是已经被分配了多少个Executor；
    coresToAssign：最少为Application分配的core数；
    
![](https://ws2.sinaimg.cn/large/006tNbRwly1fw4g51rmzbj31ho0mi40b.jpg)   

    canLaunchExecutor()此函数判断该worker上是否可以启动一个Executor；
    首先要判断该worker上是否有充足的资源，usableWorkers(pos)代表一个worker；
    接着判断在这个方法内如果允许在一个worker上启动多个Executor，那么他将总是启动新的Executor，否则，如果有之前启动
    的Executor，就在这个Executor上不断的增加cores；如下代码所示：
    
![](https://ws4.sinaimg.cn/large/006tNbRwly1fw4gdjvva9j31h60q4q4s.jpg)

    接着看具体的执行方法，
    利用canLaunchExecutor()过滤出numUsable中可用的worker；然后遍历每个worker，为每个worker上的Executor分配core；
    这里有个参数spreadOutApps，如果在默认的情况下spreadOutApps=true它会每次给我们的Executor分配一个core；
    如果在默认的情况下（spreadOutApps=true）它会每次给我们的Executor分配一个core，但是如果spreadOutApps=false它也是
	每次给我们的Executor分配一个core；
    具体的算法：
	如果是spreadOutApps=false则会不断循环使用当前worker上的这个Executor的所有freeCores；
	实际上的工作原理：假如有四个worker，如果是spreadOutApps=true，它会在每个worker上启动一个Executor然后先循环一轮，
	给每个woker上的Executor分配一个core，然后再次循环再给每个executor 分配一个core，依次循环分配;
       
![](https://ws3.sinaimg.cn/large/006tNbRwly1fw4gr8n0pzj31hi12kgof.jpg)

    从下面的代码其实我们可以看出：
    如果是每个worker下面只能够为当前的应用程序分配一个Executor的话，每次为这个executor只分配一个core！
    在应用程序提交时指定每个Executor分配多个cores，其实在实际分配的时候没有什么用，因为在为每个Executor
    分配core的时候是一个一个的分配的，但是指定的cores在前面做条件过滤时有用，只用在满足应用程序指定的资
    源条件情况下才能进行分配；

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fw4gxbr16aj31eq0acwet.jpg)

    上面scheduleExecutorsOnWorkers()方法中已经讲述了Executor在worker上分配的原则，并且会返回一个
    每个worker上分配资源的数组assignedCores，接下来就可以根据这个资源分配的数组去为Executor分配资源，并且启动Executor；
    
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw4h5rf84rj31jg0l2wfr.jpg)
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw4h5rf84rj31jg0l2wfr.jpg)

    接下来看下launchDriver()方法，
    1,worker.endpoint.send(LaunchExecutor(masterUrl.....
    向worker发送一个LaunchExecutor消息，在worker中启动一个ExecutorRunner线程；
    2,exec.application.driver.send(ExecutorAdded(exec.... 向driver中添加一个Executor；
    
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fw4ky40g2cj31dw0bot9f.jpg) 


#### 至此Master中的重要内容已经叙述完毕!
