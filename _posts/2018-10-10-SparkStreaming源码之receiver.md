---
layout:     post
title:      SparkStreaming源码之receiver
subtitle:   SparkStreaming源码之receiver
date:       2018-10-10
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>SparkStreaming源码之receiver篇
> 

### ReceiverTracker简介
ReceiverTracker管理ReceiverInputDStreams接受者的执行，接受输入流数据并以block的形式将输入的数据以block的形式存储；

### ReceiverTracker的实例化和启动
ReceiverTracker是在JobScheduler的start方法中进行初始化和启动的，详细代码如下;
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
    //todo receiverTracker的实例化
    receiverTracker = new ReceiverTracker(ssc)
    inputInfoTracker = new InputInfoTracker(ssc)
    //todo receiverTracker的启动
    receiverTracker.start()
    //todo jobGenerator的启动
    jobGenerator.start()
    logInfo("Started JobScheduler")
  }
```

### ReceiverTracker的内部源码
从start方法入手，在这个方法里面，先实例化了一个消息循环体ReceiverTrackerEndpoint，然后在调用了launchReceivers()方法去启动receiver,见下源代码;
```scala
  /** Start the endpoint and receiver execution thread. */
  def start(): Unit = synchronized {
    if (isTrackerStarted) {
      throw new SparkException("ReceiverTracker already started")
    }

    if (!receiverInputStreams.isEmpty) {
      endpoint = ssc.env.rpcEnv.setupEndpoint(
        "ReceiverTracker", new ReceiverTrackerEndpoint(ssc.env.rpcEnv))
      if (!skipReceiverLaunch) launchReceivers()
      logInfo("ReceiverTracker started")
      trackerState = Started
    }
  }
```
    
进入launchReceivers()方法,此方法是从ReceiverInputDStreams中获取receiver，并且将这些receiver分布到不同的worker节点上运行,而ReceiverInputDStreams就是DstreamGraph中的inputStreams，而runDummySparkJob()这个方法其实运行的就是一个空的job，它的目的就是为了能够让所有的slave节点注册进来从而能够获取到最多的资源，之后就用endPoint给自己发送StartAllReceivers的消息，详细源代码如下:
```scala
private def launchReceivers(): Unit = {
    val receivers = receiverInputStreams.map(nis => {
      val rcvr = nis.getReceiver()
      rcvr.setReceiverId(nis.id)
      rcvr
    })

    runDummySparkJob()

    logInfo("Starting " + receivers.length + " receivers")
    endpoint.send(StartAllReceivers(receivers))
  }
```
    
下面是StartAllReceivers消息体执行的详细内容:  
1，schedulingPolicy.scheduleReceivers(receivers, getExecutors)方法是为了让receiver在最大化数据本地性的需求下均匀的分步在各个executor上；  
2，循环遍历每个receiver，采用逐个启动每个receiver的方法；  
3，val executors = scheduledLocations(receiver.streamId)获取该receiver分步在哪些executor上；  
4，updateReceiverScheduledExecutors(receiver.streamId, executors)更新维护receiverTrackingInfos数据结构，以receiver id为key，以receiver info为value的数据结构；  
5，receiverPreferredLocations(receiver.streamId) = receiver.preferredLocation，将该receiver的位置信息存放在receiverPreferredLocations数据结构中；  
6，调用startReceiver(receiver, executors)方法，启动receiver；  
详细源代码如下:  

```scala
 override def receive: PartialFunction[Any, Unit] = {
   // Local messages
   case StartAllReceivers(receivers) =>
     val scheduledLocations = schedulingPolicy.scheduleReceivers(receivers, getExecutors)
     for (receiver <- receivers) {
       val executors = scheduledLocations(receiver.streamId)
       updateReceiverScheduledExecutors(receiver.streamId, executors)
       receiverPreferredLocations(receiver.streamId) = receiver.preferredLocation
       startReceiver(receiver, executors)
     }
```

进入startReceiver方法，该方法主要是启动一个receiver和对应的executor；  
查询源码可见，receiver是被封装成了一个rdd，然后以Job的形式通过SparkContext启动，并且用future监听这个reveiver job的状态，因为每个job都是一个线程，一旦启动失败，或者执行异常都会重新发送self.send(RestartReceiver(receiver))消息给自己，重新启动该reveicer；
```scala
    /**
     * Start a receiver along with its scheduled executors
     */
    private def startReceiver(
        receiver: Receiver[_],
        scheduledLocations: Seq[TaskLocation]): Unit = {
      def shouldStartReceiver: Boolean = {
        // It's okay to start when trackerState is Initialized or Started
        !(isTrackerStopping || isTrackerStopped)
      }

      val receiverId = receiver.streamId
      if (!shouldStartReceiver) {
        onReceiverJobFinish(receiverId)
        return
      }

      val checkpointDirOption = Option(ssc.checkpointDir)
      val serializableHadoopConf =
        new SerializableConfiguration(ssc.sparkContext.hadoopConfiguration)

      // Function to start the receiver on the worker node
      val startReceiverFunc: Iterator[Receiver[_]] => Unit =
        (iterator: Iterator[Receiver[_]]) => {
          if (!iterator.hasNext) {
            throw new SparkException(
              "Could not start receiver as object not found.")
          }
          if (TaskContext.get().attemptNumber() == 0) {
            val receiver = iterator.next()
            assert(iterator.hasNext == false)
            val supervisor = new ReceiverSupervisorImpl(
              receiver, SparkEnv.get, serializableHadoopConf.value, checkpointDirOption)
            supervisor.start()
            supervisor.awaitTermination()
          } else {
            // It's restarted by TaskScheduler, but we want to reschedule it again. So exit it.
          }
        }

      // Create the RDD using the scheduledLocations to run the receiver in a Spark job
      //todo 创建一个rdd
      val receiverRDD: RDD[Receiver[_]] =
        if (scheduledLocations.isEmpty) {
          ssc.sc.makeRDD(Seq(receiver), 1)
        } else {
          val preferredLocations = scheduledLocations.map(_.toString).distinct
          ssc.sc.makeRDD(Seq(receiver -> preferredLocations))
        }
      receiverRDD.setName(s"Receiver $receiverId")
      ssc.sparkContext.setJobDescription(s"Streaming job running receiver $receiverId")
      ssc.sparkContext.setCallSite(Option(ssc.getStartSite()).getOrElse(Utils.getCallSite()))
      //todo 以job的形式启动一个Receiver
      val future = ssc.sparkContext.submitJob[Receiver[_], Unit, Unit](
        receiverRDD, startReceiverFunc, Seq(0), (_, _) => Unit, ())
      // We will keep restarting the receiver job until ReceiverTracker is stopped
      future.onComplete {
        case Success(_) =>
          if (!shouldStartReceiver) {
            //todo  启动成功
            onReceiverJobFinish(receiverId)
          } else {
            logInfo(s"Restarting Receiver $receiverId")
            self.send(RestartReceiver(receiver))//todo 启动失败，从新启动Receiver
          }
        case Failure(e) =>
          if (!shouldStartReceiver) {
            onReceiverJobFinish(receiverId)
          } else {
            logError("Receiver has been stopped. Try to restart it.", e)
            logInfo(s"Restarting Receiver $receiverId")
            self.send(RestartReceiver(receiver))//todo 启动失败，从新启动Receiver
          }
      }(submitJobThreadPool)
      logInfo(s"Receiver ${receiver.streamId} started")
    }

```

这个receiver rdd中重要的是startReceiverFunc这个方法，这个方法里面详细描述着receiver启动和它工作的内容，在这里它会实例化一个supervisor；

```scala
 // Function to start the receiver on the worker node
 val startReceiverFunc: Iterator[Receiver[_]] => Unit =
   (iterator: Iterator[Receiver[_]]) => {
     if (!iterator.hasNext) {
       throw new SparkException(
         "Could not start receiver as object not found.")
     }
     if (TaskContext.get().attemptNumber() == 0) {
       val receiver = iterator.next()
       assert(iterator.hasNext == false)
       val supervisor = new ReceiverSupervisorImpl(
         receiver, SparkEnv.get, serializableHadoopConf.value, checkpointDirOption)
       supervisor.start()
       supervisor.awaitTermination()
     } else {
       // It's restarted by TaskScheduler, but we want to reschedule it again. So exit it.
     }
   }
```

然后调用ReceiverSupervisor的start方法    
```scala
  /** Start the supervisor */
  def start() {
    onStart()
    startReceiver()
  }
```
我们先看下ReceiverSupervisorImpl实现的onStart()方法，接着进入start方法，可以看到在start方法中做了两件事：  
1，启动blockIntervalTimer，它是一个定时器，具体执行的是updateCurrentBuffer方法，前面说过SocketInputDStream的例子，通过网络接受数据，并将数据store到currentBuffer中，而updateCurrentBuffer方法的作用就是将currentBuffer中的数据转换成block，然后存储到blocksForPushing数据结构中;  
2，启动blockPushingThread，这个线程具体执行的方法是keepPushingBlocks()，在keepPushingBlocks()方法中将生成的block通过blockManager进行存储;  
```scala
  //todo 在onstart方法里调用start方法
  override protected def onStart() {
    registeredBlockGenerators.foreach { _.start() }
  }
  
  /** Start block generating and pushing threads. */
  def start(): Unit = synchronized {
    if (state == Initialized) {
      state = Active
      //todo 定时器的启动
      blockIntervalTimer.start()
      //todo blockPushingThread线程的启动
      blockPushingThread.start()
      logInfo("Started BlockGenerator")
    } else {
      throw new SparkException(
        s"Cannot start BlockGenerator as its not in the Initialized state [state = $state]")
    }
  }
  
  /** Change the buffer to which single records are added to. */
  private def updateCurrentBuffer(time: Long): Unit = {
    try {
      var newBlock: Block = null
      synchronized {
        if (currentBuffer.nonEmpty) {
          val newBlockBuffer = currentBuffer
          currentBuffer = new ArrayBuffer[Any]
          val blockId = StreamBlockId(receiverId, time - blockIntervalMs)
          listener.onGenerateBlock(blockId)
          //todo 将内存中的数据buffer生成一个block
          newBlock = new Block(blockId, newBlockBuffer)
        }
      }

      if (newBlock != null) {
        //todo 将新生成的block放入到blocksForPushing中
        blocksForPushing.put(newBlock)  // put is blocking when queue is full
      }
    } catch {
      case ie: InterruptedException =>
        logInfo("Block updating timer thread was interrupted")
      case e: Exception =>
        reportError("Error in block updating thread", e)
    }
  }
  
  /** Keep pushing blocks to the BlockManager. */
  private def keepPushingBlocks() {
    logInfo("Started block pushing thread")

    def areBlocksBeingGenerated: Boolean = synchronized {
      state != StoppedGeneratingBlocks
    }

    try {
      // While blocks are being generated, keep polling for to-be-pushed blocks and push them.
      while (areBlocksBeingGenerated) {
        Option(blocksForPushing.poll(10, TimeUnit.MILLISECONDS)) match {
          case Some(block) => pushBlock(block)
          case None =>
        }
      }

      // At this point, state is StoppedGeneratingBlock. So drain the queue of to-be-pushed blocks.
      logInfo("Pushing out the last " + blocksForPushing.size() + " blocks")
      while (!blocksForPushing.isEmpty) {
        val block = blocksForPushing.take()
        logDebug(s"Pushing block $block")
        pushBlock(block)
        logInfo("Blocks left to push " + blocksForPushing.size())
      }
      logInfo("Stopped block pushing thread")
    } catch {
      case ie: InterruptedException =>
        logInfo("Block pushing thread was interrupted")
      case e: Exception =>
        reportError("Error in block pushing thread", e)
    }
  }
```
ok，ReceiverSupervisor的onStart()方法介绍完毕了，回过头来再看下startReceiver()方法，在这个方法内启动了receiver；
```scala
  /** Start receiver */
  def startReceiver(): Unit = synchronized {
    try {
      if (onReceiverStart()) {
        logInfo("Starting receiver")
        receiverState = Started
        //todo receiver启动
        receiver.onStart()
        logInfo("Called receiver onStart")
      } else {
        // The driver refused us
        stop("Registered unsuccessfully because Driver refused to start receiver " + streamId, None)
      }
    } catch {
      case NonFatal(t) =>
        stop("Error starting receiver " + streamId, Some(t))
    }
  }
```
     
我们再来看下receiver.onStart()具体的实现，还以SocketReceiver为列，可以看到receive方法中通过socket一条一条的接收数据，并将接收到的数据做存储，其实存储就是存储在上面所说的currentBuffer中了，这样就衔接上了，reveiver不停的接受数据并存储在内存中，ReceiverSupervisor将内存中的数据转换成block并用blockManager存储在内存或者磁盘中；

```scala
  def onStart() {
    // Start the thread that receives data over a connection
    new Thread("Socket Receiver") {
      setDaemon(true)
      //todo 调用receive方法
      override def run() { receive() }
    }.start()
  }
  
  /** Create a socket connection and receive data until receiver is stopped */
  def receive() {
    var socket: Socket = null
    try {
      logInfo("Connecting to " + host + ":" + port)
      socket = new Socket(host, port)
      logInfo("Connected to " + host + ":" + port)
      val iterator = bytesToObjects(socket.getInputStream())
      while(!isStopped && iterator.hasNext) {
        //todo 通过网络接受数据不断的尽心存储
        store(iterator.next)
      }
      if (!isStopped()) {
        restart("Socket data stream had no more data")
      } else {
        logInfo("Stopped receiving")
      }
    } catch {
      case e: java.net.ConnectException =>
        restart("Error connecting to " + host + ":" + port, e)
      case NonFatal(e) =>
        logWarning("Error receiving data", e)
        restart("Error receiving data", e)
    } finally {
      if (socket != null) {
        socket.close()
        logInfo("Closed socket to " + host + ":" + port)
      }
    }
  }
}
  
```
     
 #### 至此receiver介绍完毕；    
