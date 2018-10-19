---
layout:     post
title:      Spark源码之BlockManager
subtitle:   Spark源码之BlockManager介绍
date:       2018-10-17
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>Spark源码之BlockManager篇
>


BlockManage作为对外提供的统一访问block的接口,而且它在spark中是以Master-Slave的模式存在,既然它在spark中承担着数据管理的大任，那么此篇就详细的探讨下BlockManager;


### BlockManager的生成,以及组织架构的形成

BlockManager是在SparkEnv构建的时候生成的，我们打开SparkEnv的源码,可以看到先实例化出BlockManagerMaster以及BlockManagerMaster的消息通讯体BlockManagerMasterEndpoint，在实例化出BlockManagerMaster后开始创建BlockManager，因为BlockManager在spark中是以Master-Slave的形式存在的，那么它的slava又是在哪里分布的呢？

```scala
//todo 创建blockTransferService
val blockTransferService = new NettyBlockTransferService(conf, securityManager, numUsableCores)
//todo 创建BlockManagerMaster,并且实例化出BlockManagerMasterEndpoint
val blockManagerMaster = new BlockManagerMaster(registerOrLookupEndpoint(
  BlockManagerMaster.DRIVER_ENDPOINT_NAME,
  new BlockManagerMasterEndpoint(rpcEnv, isLocal, conf, listenerBus)),
  conf, isDriver)
// NB: blockManager is not valid until initialize() is called later.
//todo 根据Executor创建出BlockManager
val blockManager = new BlockManager(executorId, rpcEnv, blockManagerMaster,
  serializer, conf, memoryManager, mapOutputTracker, shuffleManager,
  blockTransferService, securityManager, numUsableCores)

```

我们在打开我们的ExecutorBackEnd代码,可以看到在实例化CoarseGrainedExecutorBackend实例的时候传入了一个SparkEnv,其实在每个ExecutorBackend中都会分布一个BlockManager，并以slave的形式存在，这里调用`SparkEnv.createExecutorEnv(......）`方法，将ExecutorBackend的executorid传入进入，而在SparkEnv的createExecutorEnv方法中创建BlockManager时，是根据传入的executorId创建的,而在BlockManager中会实例化出BlockManagerSlaveEndpoint实例,和BlockManagerMaster能够建立通讯，这就形成了BlockManager的通讯结构;

```scala
 //todo 创建基于当前Executor的SparkEnv
 val env = SparkEnv.createExecutorEnv(
   driverConf, executorId, hostname, port, cores, isLocal = false)
 // SparkEnv will set spark.executor.port if the rpc env is listening for incoming
 // connections (e.g., if it's using akka). Otherwise, the executor is running in
 // client mode only, and does not accept incoming connections.
 val sparkHostPort = env.conf.getOption("spark.executor.port").map { port =>
     hostname + ":" + port
   }.orNull
 env.rpcEnv.setupEndpoint("Executor", new CoarseGrainedExecutorBackend(
   env.rpcEnv, driverUrl, executorId, sparkHostPort, cores, userClassPath, env))
```
```scala
//todo 在BlockManager中会实例化出BlockManagerSlaveEndpoint实例
private val slaveEndpoint = rpcEnv.setupEndpoint(
  "BlockManagerEndpoint" + BlockManager.ID_GENERATOR.next,
  new BlockManagerSlaveEndpoint(rpcEnv, this, mapOutputTracker))
```


