---
layout:     post
title:      SparkStreaming源码之JobGenerator
subtitle:   SparkStreaming源码之JobGenerator
date:       2018-10-08
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>SparkStreaming源码之JobGenerator篇
> 

### JobGenerator概述

```scala
主要作用就是生成SparkStreaming Job 并且驱动checkpoint的产生和清理Dstream的元数据  
This class generates jobs from DStreams as well as drives checkpointing and cleaning  
up DStream metadata.  
```

### JobGenerator是如何实例化的并且如何启动
其实是在JobScheduler这个类中进行初始化的  
```scala
  //todo 实例化JobGenerator
  private val jobGenerator = new JobGenerator(this)
  val clock = jobGenerator.clock
  val listenerBus = new StreamingListenerBus()
```
并且在JobScheduler这个类启动的时候也调用了JobGenerator的Start方法
```scala
  def start(): Unit = synchronized {
    if (eventLoop != null) return // scheduler has already been started

    //todo 内部的消息循环体
    logDebug("Starting JobScheduler")
    eventLoop = new EventLoop[JobSchedulerEvent]("JobScheduler") {
      override protected def onReceive(event: JobSchedulerEvent): Unit = processEvent(event)

      override protected def onError(e: Throwable): Unit = reportError("Error in job scheduler", e)
    }
    eventLoop.start()

    // attach rate controllers of input streams to receive batch completion updates
    for {
      inputDStream <- ssc.graph.getInputStreams
      rateController <- inputDStream.rateController
    } ssc.addStreamingListener(rateController)

    listenerBus.start(ssc.sparkContext)
    receiverTracker = new ReceiverTracker(ssc)
    inputInfoTracker = new InputInfoTracker(ssc)
    receiverTracker.start()
    //todo jobGenerator的启动
    jobGenerator.start()
    logInfo("Started JobScheduler")
  
```
这个是JobGenerator的start方法，在这个方法中实例化了一个消息循环体，并启动了这个消息循环体(EventLoop[JobGeneratorEvent])
```scala
  /** Start generation of jobs */
  def start(): Unit = synchronized {
    if (eventLoop != null) return // generator has already been started

    // Call checkpointWriter here to initialize it before eventLoop uses it to avoid a deadlock.
    // See SPARK-10125
    checkpointWriter
    //todo 初始化消息循环体
    eventLoop = new EventLoop[JobGeneratorEvent]("JobGenerator") {
      override protected def onReceive(event: JobGeneratorEvent): Unit = processEvent(event)

      override protected def onError(e: Throwable): Unit = {
        jobScheduler.reportError("Error in job generator", e)
      }
    }
    eventLoop.start()

    if (ssc.isCheckpointPresent) {
      restart()
    } else {
      startFirstTime()
    }
  }

```
### JobGenerator内部探究
其实在JobGenerator中有两个比较重要的成员，一个是定时器Timer，Timer根据Interval time不断向自己发送GenerateJobs消息，
另一个是消息循环体EventLoop
```scala
  //todo 定时器
  private val timer = new RecurringTimer(clock, ssc.graph.batchDuration.milliseconds,
    longTime => eventLoop.post(GenerateJobs(new Time(longTime))), "JobGenerator")
    
  //todo 消息循环体
  private var eventLoop: EventLoop[JobGeneratorEvent] = null    
```
 
再来看下消息循环体的具体内容
```scala
  /** Processes all events */
  private def processEvent(event: JobGeneratorEvent) {
    logDebug("Got event " + event)
    event match {
      case GenerateJobs(time) => generateJobs(time)//todo 产生job
      case ClearMetadata(time) => clearMetadata(time) //todo 清理元数据
      case DoCheckpoint(time, clearCheckpointDataLater) => //todo 做checkpoint操作
        doCheckpoint(time, clearCheckpointDataLater)
      case ClearCheckpointData(time) => clearCheckpointData(time)  //todo 清理checkPoint数据
    }
  }
```
我们主要看下generateJobs方法
```scala
  /** Generate jobs and perform checkpoint for the given `time`.  */
  private def generateJobs(time: Time) {
    // Set the SparkEnv in this thread, so that job generation code can access the environment
    // Example: BlockRDDs are created in this thread, and it needs to access BlockManager
    // Update: This is probably redundant after threadlocal stuff in SparkEnv has been removed.
    SparkEnv.set(ssc.env)
    Try {
      jobScheduler.receiverTracker.allocateBlocksToBatch(time) // allocate received blocks to batch
      graph.generateJobs(time) // generate jobs using allocated block
    } match {
      case Success(jobs) =>
        val streamIdToInputInfos = jobScheduler.inputInfoTracker.getInfo(time)
        //todo 将生成的job提交给jobScheduler
        jobScheduler.submitJobSet(JobSet(time, jobs, streamIdToInputInfos))
      case Failure(e) =>
        jobScheduler.reportError("Error generating jobs for time " + time, e)
    }
    eventLoop.post(DoCheckpoint(time, clearCheckpointDataLater = false))
  }
```
在allocateBlocksToBatch方法中获取根据interval time划分的block块数据
```scala
  jobScheduler.receiverTracker.allocateBlocksToBatch(time) // allocate received blocks to batch
    
  def allocateBlocksToBatch(batchTime: Time): Unit = synchronized {
    if (lastAllocatedBatchTime == null || batchTime > lastAllocatedBatchTime) {
      val streamIdToBlocks = streamIds.map { streamId =>
          (streamId, getReceivedBlockQueue(streamId).dequeueAll(x => true))
      }.toMap
      val allocatedBlocks = AllocatedBlocks(streamIdToBlocks)
      if (writeToLog(BatchAllocationEvent(batchTime, allocatedBlocks))) {
        timeToAllocatedBlocks.put(batchTime, allocatedBlocks)
        lastAllocatedBatchTime = batchTime
      } else {
        logInfo(s"Possibly processed batch $batchTime need to be processed again in WAL recovery")
      }
    } else {
      // This situation occurs when:
      // 1. WAL is ended with BatchAllocationEvent, but without BatchCleanupEvent,
      // possibly processed batch job or half-processed batch job need to be processed again,
      // so the batchTime will be equal to lastAllocatedBatchTime.
      // 2. Slow checkpointing makes recovered batch time older than WAL recovered
      // lastAllocatedBatchTime.
      // This situation will only occurs in recovery time.
      logInfo(s"Possibly processed batch $batchTime need to be processed again in WAL recovery")
    }
  }    
```    
在获取到属于该job的数据后开始产生job
```scala
  /** Generate jobs and perform checkpoint for the given `time`.  */
  private def generateJobs(time: Time) {
    // Set the SparkEnv in this thread, so that job generation code can access the environment
    // Example: BlockRDDs are created in this thread, and it needs to access BlockManager
    // Update: This is probably redundant after threadlocal stuff in SparkEnv has been removed.
    SparkEnv.set(ssc.env)
    Try {
      jobScheduler.receiverTracker.allocateBlocksToBatch(time) // allocate received blocks to batch
      //todo 产生job
      graph.generateJobs(time) // generate jobs using allocated block
    } match {
      case Success(jobs) =>
        val streamIdToInputInfos = jobScheduler.inputInfoTracker.getInfo(time)
        //todo 将生成的job提交给jobScheduler
        jobScheduler.submitJobSet(JobSet(time, jobs, streamIdToInputInfos))
      case Failure(e) =>
        jobScheduler.reportError("Error generating jobs for time " + time, e)
    }
    eventLoop.post(DoCheckpoint(time, clearCheckpointDataLater = false))
  }
  
  def generateJobs(time: Time): Seq[Job] = {
    logDebug("Generating jobs for time " + time)
    val jobs = this.synchronized {
      outputStreams.flatMap { outputStream =>
        //todo  根据最后的action操作产生job
        val jobOption = outputStream.generateJob(time)
        jobOption.foreach(_.setCallSite(outputStream.creationSite))
        jobOption
      }
    }
    logDebug("Generated " + jobs.length + " jobs for time " + time)
    jobs
  }
```

再向下看就是Dstream的generatorJob的方法了，其实这个方法会被子类的实现所覆盖，例如print操作产生的ForeachDstream
```scala
  private[streaming] def generateJob(time: Time): Option[Job] = {
    getOrCompute(time) match {
      case Some(rdd) => {
        val jobFunc = () => {
          val emptyFunc = { (iterator: Iterator[T]) => {} }
          //todo 这里调用了SparkContext的runJob方法以RDD的形式执行
          context.sparkContext.runJob(rdd, emptyFunc)
        }
        Some(new Job(time, jobFunc))
      }
      case None => None
    }
  }
```
Dstream的子类ForeachDstream实现方法,可见是通过从后向前回溯的方法来生成一个job，特别是Some(new Job(time, jobFunc))
中的jobFunc方法，就是自定义的输出方法，可以去看下Dstream里面的Print方法是如何传入的
```scala
  override def generateJob(time: Time): Option[Job] = {
    parent.getOrCompute(time) match {
      case Some(rdd) =>
        val jobFunc = () => createRDDWithLocalProperties(time, displayInnerRDDOps) {
          foreachFunc(rdd, time)
        }
        Some(new Job(time, jobFunc))
      case None => None
    }
  }
```
ok!基于interval time生成的job就已经ok了，接下来就是如何将生成的job向下传递了，根据代码可见，最后是将生成的job交给JobScheduler进行处理；
```scala
/** Generate jobs and perform checkpoint for the given `time`.  */
  private def generateJobs(time: Time) {
    // Set the SparkEnv in this thread, so that job generation code can access the environment
    // Example: BlockRDDs are created in this thread, and it needs to access BlockManager
    // Update: This is probably redundant after threadlocal stuff in SparkEnv has been removed.
    SparkEnv.set(ssc.env)
    Try {
      jobScheduler.receiverTracker.allocateBlocksToBatch(time) // allocate received blocks to batch
      //todo 产生job
      graph.generateJobs(time) // generate jobs using allocated block
    } match {
      case Success(jobs) =>
        val streamIdToInputInfos = jobScheduler.inputInfoTracker.getInfo(time)
        //todo 将生成的job提交给jobScheduler
        jobScheduler.submitJobSet(JobSet(time, jobs, streamIdToInputInfos))
      case Failure(e) =>
        jobScheduler.reportError("Error generating jobs for time " + time, e)
    }
    eventLoop.post(DoCheckpoint(time, clearCheckpointDataLater = false))
  }
```
***

### 上面介绍了JobGenerator这个角色，接下来再引申一个问题，为什么SparkStreaming不会处理到半条数据的情况？
    
    不会出现处理半条数据的原因有两点:
    1，receiver接收数据是一条一条的接收的；
    2，receive的接收数据，向driver端汇报数据，为batch分配数据这三者其实时间不是一致的

    其实存在着这样的一种情况:
    Batch1接收了1000条数据，batch2接收了500条数据，  在处理时会出现batch1处理了900条数据，batch2处理了600条数据；
    原因：
    在给job分配数据的时候使用了synchronized修饰，那么就会造成batch在分配数据时并不一定能分配1000条数；
    所以revice接收到的这条数据分配给哪个batch是由何时获取锁的时间来决定的；
```scala
  /**
   * Allocate all unallocated blocks to the given batch.
   * This event will get written to the write ahead log (if enabled).
   */
  def allocateBlocksToBatch(batchTime: Time): Unit = synchronized {
    if (lastAllocatedBatchTime == null || batchTime > lastAllocatedBatchTime) {
      val streamIdToBlocks = streamIds.map { streamId =>
          (streamId, getReceivedBlockQueue(streamId).dequeueAll(x => true))
      }.toMap
      val allocatedBlocks = AllocatedBlocks(streamIdToBlocks)
      if (writeToLog(BatchAllocationEvent(batchTime, allocatedBlocks))) {
        timeToAllocatedBlocks.put(batchTime, allocatedBlocks)
        lastAllocatedBatchTime = batchTime
      } else {
        logInfo(s"Possibly processed batch $batchTime need to be processed again in WAL recovery")
      }
    } else {
      // This situation occurs when:
      // 1. WAL is ended with BatchAllocationEvent, but without BatchCleanupEvent,
      // possibly processed batch job or half-processed batch job need to be processed again,
      // so the batchTime will be equal to lastAllocatedBatchTime.
      // 2. Slow checkpointing makes recovered batch time older than WAL recovered
      // lastAllocatedBatchTime.
      // This situation will only occurs in recovery time.
      logInfo(s"Possibly processed batch $batchTime need to be processed again in WAL recovery")
    }
  }
```

#### OK! jobGenrator介绍完毕,其实比较重要的就是数据的分配和job的产生；






