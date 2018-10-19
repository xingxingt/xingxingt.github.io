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


//todo 在BlockManager中会实例化出BlockManagerSlaveEndpoint实例
private val slaveEndpoint = rpcEnv.setupEndpoint(
  "BlockManagerEndpoint" + BlockManager.ID_GENERATOR.next,
  new BlockManagerSlaveEndpoint(rpcEnv, this, mapOutputTracker))
```

### BlockManager的注册

我们再来看下BlockManager是如何向BlockManagerMaster注册的,在BlockManager实例化后调用initialize方法,在此方法内执行了BlockManagerMaster.registerBlockManager方法；

```scala
//blockManager的initialize方法
def initialize(appId: String): Unit = {
  blockTransferService.init(this)
  shuffleClient.init(appId)
  blockManagerId = BlockManagerId(
    executorId, blockTransferService.hostName, blockTransferService.port)
  shuffleServerId = if (externalShuffleServiceEnabled) {
    logInfo(s"external shuffle service port = $externalShuffleServicePort")
    BlockManagerId(executorId, blockTransferService.hostName, externalShuffleServicePort)
  } else {
    blockManagerId
  }
  master.registerBlockManager(blockManagerId, maxMemory, slaveEndpoint)
  // Register Executors' configuration with the local shuffle service, if one should exist.
  if (externalShuffleServiceEnabled && !blockManagerId.isDriver) {
    registerWithExternalShuffleServer()
  }
```

进入BlockManagerMaster的registerBlockManager方法,可以看到它向master endpoint发送消息,其实这里的Driver就是BlockManagerMasterEndpoint

```scala
 /** Register the BlockManager's id with the driver. */
 def registerBlockManager(
     blockManagerId: BlockManagerId, maxMemSize: Long, slaveEndpoint: RpcEndpointRef): Unit = {
   logInfo("Trying to register BlockManager")
   tell(RegisterBlockManager(blockManagerId, maxMemSize, slaveEndpoint))
   logInfo("Registered BlockManager")
 }  
 
 /** Send a one-way message to the master endpoint, to which we expect it to reply with true. */
 private def tell(message: Any) {
   if (!driverEndpoint.askWithRetry[Boolean](message)) {
     throw new SparkException("BlockManagerMasterEndpoint returned false, expected true.")
   }
 }
```

我们再进入BlockManagerMasterEndpoint中,可以看到这里接收到BlockManager的注册，然后调用register方法;

```scala
override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
  case RegisterBlockManager(blockManagerId, maxMemSize, slaveEndpoint) =>
    register(blockManagerId, maxMemSize, slaveEndpoint)
    context.reply(true)
```

进入register方法中会看到有一个blockManagerInfo数据结构，这个数据结构存储所有slave注册的BlockManager信息;到这里ExecutorBackend中的BlockManager信息向Driver(BlockManagerMasterEndpoint)注册完毕;Driver端会维护所有ExecutorBackend上的BlockManager信息;

```scala
private def register(id: BlockManagerId, maxMemSize: Long, slaveEndpoint: RpcEndpointR
  val time = System.currentTimeMillis()
  if (!blockManagerInfo.contains(id)) {
    blockManagerIdByExecutor.get(id.executorId) match {
      case Some(oldId) =>
        // A block manager of the same executor already exists, so remove it (assumed 
        logError("Got two different block manager registrations on same executor - "
            + s" will replace old one $oldId with new one $id")
        removeExecutor(id.executorId)
      case None =>
    }
    logInfo("Registering block manager %s with %s RAM, %s".format(
      id.hostPort, Utils.bytesToString(maxMemSize), id))
    blockManagerIdByExecutor(id.executorId) = id
    blockManagerInfo(id) = new BlockManagerInfo(
      id, System.currentTimeMillis(), maxMemSize, slaveEndpoint)
  }
  listenerBus.post(SparkListenerBlockManagerAdded(time, id, maxMemSize))
}
```

### BlockManager的内部工作

BlockManager之block信息上报：  
1.在BlockManager中的reportAllBlocks方法,遍历所有的的blockInfo准备上报给Master;   
2.接着进入tryToReportBlockStatus方法,在该方法中调用BlockManagerMaster的updateBlockInfo方法;  
3.在BlockManager的updateBlockInfo方法中可以看到向driverEndpoint发送UpdateBlockInfo消息;  
4.UpdateBlockInfo收到UpdateBlockInfo消息，然后再进一步调用自身内部的UpdateBlockInfo操作,具体方法这里不再叙述，主要是去更改维护driverEndpoint中的blockManagerInfo信息;  
参照如下代码: 
```scala
//1.blockManager中的reportAllBlocks方法
private def reportAllBlocks(): Unit = {
  logInfo(s"Reporting ${blockInfo.size} blocks to the master.")
  for ((blockId, info) <- blockInfo) {
    val status = getCurrentBlockStatus(blockId, info)
    if (!tryToReportBlockStatus(blockId, info, status)) {
      logError(s"Failed to report $blockId to master; giving up.")
      return
    }
  }
}

//2.blockManager中的tryToReportBlockStatus方法
private def tryToReportBlockStatus(
    blockId: BlockId,
    info: BlockInfo,
    status: BlockStatus,
    droppedMemorySize: Long = 0L): Boolean = {
  if (info.tellMaster) {
    val storageLevel = status.storageLevel
    val inMemSize = Math.max(status.memSize, droppedMemorySize)
    val inExternalBlockStoreSize = status.externalBlockStoreSize
    val onDiskSize = status.diskSize
    master.updateBlockInfo(
      blockManagerId, blockId, storageLevel, inMemSize, onDiskSize, inExternalBlockStoreSize)
  } else {
    true
  }
}

//3.BlockManagerMaster中的updateBlockInfo方法
def updateBlockInfo(
    blockManagerId: BlockManagerId,
    blockId: BlockId,
    storageLevel: StorageLevel,
    memSize: Long,
    diskSize: Long,
    externalBlockStoreSize: Long): Boolean = {
  val res = driverEndpoint.askWithRetry[Boolean](
    UpdateBlockInfo(blockManagerId, blockId, storageLevel,
      memSize, diskSize, externalBlockStoreSize))
  logDebug(s"Updated info of block $blockId")
  res
}

//4,driverEndpoint收到UpdateBlockInfo的信息
case _updateBlockInfo @ UpdateBlockInfo(
  blockManagerId, blockId, storageLevel, deserializedSize, size, externalBlockStoreSize) 
  context.reply(updateBlockInfo(
    blockManagerId, blockId, storageLevel, deserializedSize, size, externalBlockStoreSize
  listenerBus.post(SparkListenerBlockUpdated(BlockUpdatedInfo(_updateBlockInfo)))
```

BlockManager之block数据写入到指定StroreLevel：  
不管是putArray还是putBytes，内部都是调用doPut来完成的,那么我们来看下doPut是如何完成数据的写入的,这个方法比较长我们分解解释;   
1.先判断该block和storeLevel是否为空;  
2.构建putBlockInfo,既即将put的数据对象;  
3.根据storeLevel判断使用哪种Blockstore，以及是否返回put操作的值;  
4.开始使用blockStore真正的put数据;  
5.如果使用的内存，则要将溢出的部分添加到updatedBlocks中;  
6.执行putBlockInfo.markReady(size),表示put数据结束，并唤醒其他线程;  
7.如果副本个数>1就开始异步复制数据到其他节点;  

```scala
private def doPut(
      blockId: BlockId,
      data: BlockValues,
      level: StorageLevel,
      tellMaster: Boolean = true,
      effectiveStorageLevel: Option[StorageLevel] = None)
    : Seq[(BlockId, BlockStatus)] = {
     
    //todo 1.判断该block和storeLevel是否为空;
    require(blockId != null, "BlockId is null")
    require(level != null && level.isValid, "StorageLevel is null or invalid")
    effectiveStorageLevel.foreach { level =>
      require(level != null && level.isValid, "Effective StorageLevel is null or invalid")
    }

    //todo 2.构建putBlockInfo
    val putBlockInfo = {
      //todo 生成BlockInfo
      val tinfo = new BlockInfo(level, tellMaster)
      // Do atomically !
      //todo 将tinfo存入内存中
      val oldBlockOpt = blockInfo.putIfAbsent(blockId, tinfo)
      //todo 判断该blockInfo是否存在
      if (oldBlockOpt.isDefined) {
        if (oldBlockOpt.get.waitForReady()) {
          logWarning(s"Block $blockId already exists on this machine; not re-adding it")
          return updatedBlocks
        }
        // TODO: So the block info exists - but previous attempt to load it (?) failed.
        // What do we do now ? Retry on it ?
        oldBlockOpt.get
      } else {
        tinfo
      }
    }
    
    //todo 将这个putBlockInfo加锁，防止其他线程操作该blockInfo
    putBlockInfo.synchronized {
      logTrace("Put for block %s took %s to get into synchronized block"
        .format(blockId, Utils.getUsedTimeMs(startTimeMs)))
      var marked = false
      try {
        // returnValues - Whether to return the values put
        // blockStore - The type of storage to put these values into
        //todo 3.根据storeLevel判断使用哪种Blockstore，以及是否返回put操作的值
        val (returnValues, blockStore: BlockStore) = {
          if (putLevel.useMemory) {
            // Put it in memory first, even if it also has useDisk set to true;
            // We will drop it to disk later if the memory store can't hold it.
            (true, memoryStore)
          } else if (putLevel.useOffHeap) {
            // Use external block store
            (false, externalBlockStore)
          } else if (putLevel.useDisk) {
            // Don't get back the bytes from put unless we replicate them
            (putLevel.replication > 1, diskStore)
          } else {
            assert(putLevel == StorageLevel.NONE)
            throw new BlockException(
              blockId, s"Attempted to put block $blockId without specifying storage level!")
          }
        }
        // Actually put the values
        //todo 4.开始使用blockStore真正的put数据
        val result = data match {
          case IteratorValues(iterator) =>
            blockStore.putIterator(blockId, iterator, putLevel, returnValues)
          case ArrayValues(array) =>
            blockStore.putArray(blockId, array, putLevel, returnValues)
          case ByteBufferValues(bytes) =>
            bytes.rewind()
            blockStore.putBytes(blockId, bytes, putLevel)
        }
        size = result.size
        result.data match {
          case Left (newIterator) if putLevel.useMemory => valuesAfterPut = newIterator
          case Right (newBytes) => bytesAfterPut = newBytes
          case _ =>
        }
        // Keep track of which blocks are dropped from memory
        //todo 5.如果使用的内存，则要将溢出的部分添加到updatedBlocks中
        if (putLevel.useMemory) {
          result.droppedBlocks.foreach { updatedBlocks += _ }
        }
        //todo 获取当前block的状态
        val putBlockStatus = getCurrentBlockStatus(blockId, putBlockInfo)
        if (putBlockStatus.storageLevel != StorageLevel.NONE) {
          // Now that the block is in either the memory, externalBlockStore, or disk store,
          // let other threads read it, and tell the master about it.
          marked = true
          //todo 6.表示该block已经put完成
          putBlockInfo.markReady(size)
          if (tellMaster) {
            //todo 向Master汇报block信息
            reportBlockStatus(blockId, putBlockInfo, putBlockStatus)
          }
          updatedBlocks += ((blockId, putBlockStatus))
        }
      } finally {
        // If we failed in putting the block to memory/disk, notify other possible readers
        // that it has failed, and then remove it from the block info map.
        if (!marked) {
          // Note that the remove must happen before markFailure otherwise another thread
          // could've inserted a new BlockInfo before we remove it.
          blockInfo.remove(blockId)
          putBlockInfo.markFailure()
          logWarning(s"Putting block $blockId failed")
        }
      }
    }
    
    //todo 7.如果副本个数>1就开始异步复制数据
    if (putLevel.replication > 1) {
      data match {
        case ByteBufferValues(bytes) =>
          if (replicationFuture != null) {
            Await.ready(replicationFuture, Duration.Inf)
          }
        case _ =>
          val remoteStartTime = System.currentTimeMillis
          // Serialize the block if not already done
          if (bytesAfterPut == null) {
            if (valuesAfterPut == null) {
              throw new SparkException(
                "Underlying put returned neither an Iterator nor bytes! This shouldn't happen.")
            }
            bytesAfterPut = dataSerialize(blockId, valuesAfterPut)
          }
          //todo 将数复制到其他节点上
          replicate(blockId, bytesAfterPut, putLevel)
          logDebug("Put block %s remotely took %s"
            .format(blockId, Utils.getUsedTimeMs(remoteStartTime)))
      }
    }
    
    ......
```






