---
layout:     post
title:      spark BroadCast
subtitle:   
date:       2018-09-02
author:     XINGXING
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Spark
---

>
>spark BroadCast篇
> 

### BroadCast的作用:
    1,BroadCast就是将数据从一个节点发送到其它的节点上；例如Driver上有一张表，而Executor中的每个并行执行的Task都要查询这张表，那我们通过BroadCast方式
      就只需要往每个Executor中把这张表发送一次就行了，Executor中每个运行的Task查询这张唯一的表，而不是每次执行的时候都从Driver端获取这张表；
    2,这就好像ServletContext的具体作用（获取上下文中资源），只是BroadCast是分布式的共享数据，默认情况下只要程序运行BroadCast变量就会存在，
      因为BroadCast在底层是通过BlockManager管理的，但是你可以手动指定或者配置具体周期来销毁BroadCast变量；
      
### BroadCast特性:
    1,BroadCast一般用于处理配置文件，通用的DataSet，常用的数据结构等等；但是不适合存放太大的数据在BroadCast中，BroadCast不会内存溢出，
      因为其数据的保存的StroraLevel是MEMORY_AND_DISK的方式，虽然如此，我们也不可以放入太大的数据在BroadCast中，因为网络IO和可能的单点压力会非常大；
    2,广播BroadCast变量是只读变量， 因为很难保证数据的一致性；  

### BroadCastManager的实例化:
    在sparkContext初始化SparkEnv时创建,在sparkEnv的create方法中完成初始化；
    在实例化BroadCastManager的时候会创建BroadCastFactory工厂来构建具体实际的BroadCast类型，默认情况下是TorrentBroadCastFactory；
    private def initialize() {
      synchronized {
        if (!initialized) {
          val broadcastFactoryClass =
            conf.get("spark.broadcast.factory", "org.apache.spark.broadcast.TorrentBroadcastFactory")
           broadcastFactory =
            Utils.classForName(broadcastFactoryClass).newInstance.asInstanceOf[BroadcastFactory]

          // Initialize appropriate BroadcastFactory and BroadcastObject
          broadcastFactory.initialize(isDriver, conf, securityManager)

          initialized = true
        }
      }
    }
    

### BroadCast的两种广播方式:
    1,HttpBroadCast方式： 最开始的时候数据放在Driver的本地文件系统中，Driver在本地会创建一个文件夹来存放BroadCast中的Data，
      然后启动HttpServer来访问文件夹中的数据，同时写入到BlockManager（StorageLevel级别是Memory_and_disk）中获得BlockId（BroadCastId），
      当第一次Executor中的Task要访问BroadCast变量的时候，会向Driver通过HttpServer来访问数据，然后会在Executor中的BlockManager中注册该
      BroadCast中的数据BlockManager,这样后续Task需要访问BroadCast的变量的时候会首先查询BlockManager中有没有该数据，如果有就直接使用；
    2,TorrentBroadCast方式：首先Driver作为数据源，如果A节点请求获取BroadCast数据，那么A节点也会作为一个数据源，再B节点请求获取BroadCast
      数据时可以从多个节点获取数据，之后B节点也会成为一个数据源点，随着请求的增多，数据源点也会越来越多，TorrentBroadCast内部也是使用NIO的机制；

### 两种广播方式对比:
    1,HttpBroadCast会出现单点故障，网络IO瓶颈（所有的节点都去请求Driver中的BroadCast数据）；
    2,TorrentBroadCast按照Block_Size（默认是4M）将BroadCast的数据划分成不同的Block，然后将分块信息也就是Meta信息存放在Driver的BlockManager
      中（StroreLevel是Memory_and_disk），同时会告诉BlockManagerMaster说明Meta信息存放完毕（从而使BroadCast数据做到全局化）；
      
      
      
      
      
      
