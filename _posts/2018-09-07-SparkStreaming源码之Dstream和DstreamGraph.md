---
layout:     post
title:      SparkStreaming源码之Dstream和DstreamGraph
subtitle:   SparkStreaming源码之Dstream和DstreamGraph这些魔板类
date:       2018-09-07
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>SparkStreaming源码之Dstream和DstreamGraph篇
> 


## 先谈DstreamGraph，
在DstreamGraph中有两个ArrayBuffer，
```scala
  private val inputStreams = new ArrayBuffer[InputDStream[_]]()
  private val outputStreams = new ArrayBuffer[DStream[_]]()
```

inputStreams的作用就是存放一个流的inputDstream，例如SocketInputDStream,他是在父类InputDStream中执行具体的存放操作
```scala
abstract class InputDStream[T: ClassTag] (ssc_ : StreamingContext)
  extends DStream[T](ssc_) {

  private[streaming] var lastValidTime: Time = null

  //todo 将DStream放入到DstreamGraph的InputStream数组中
  ssc.graph.addInputStream(this)
```

那么InputDStream接收的数据又是如何进行存储的呢?
```scala
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

def store(dataItem: T) {
    supervisor.pushSingle(dataItem)
 }
```

经过代码追踪发现接受的数据实际上是以block的形式存放，BlockGenerator以spark.streaming.blockInterval作为时间单位来生成block块，内部有一个Timer来定时生成block块：我觉得这里的RecurringTimer做的挺好，同一个Timer根据不同的callback方法来执行不同的任务，get到了新技能,点赞！
```scala
private val blockIntervalMs = conf.getTimeAsMs("spark.streaming.blockInterval", "200ms")
require(blockIntervalMs > 0, s"'spark.streaming.blockInterval' should be a positive value")

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
        newBlock = new Block(blockId, newBlockBuffer)
      }
    }
    if (newBlock != null) {
      blocksForPushing.put(newBlock)  // put is blocking when queue is f
    }
  } catch {
    case ie: InterruptedException =>
      logInfo("Block updating timer thread was interrupted")
    case e: Exception =>
      reportError("Error in block updating thread", e)
  }
}
```

上面说了inputStream,接下来看下outputStream,以print操作为例:
```scala
def print(num: Int): Unit = ssc.withScope {
  def foreachFunc: (RDD[T], Time) => Unit = {
    (rdd: RDD[T], time: Time) => {
      val firstNum = rdd.take(num + 1)
      // scalastyle:off println
      println("-------------------------------------------")
      println("Time: " + time)
      println("-------------------------------------------")
      firstNum.take(num).foreach(println)
      if (firstNum.length > num) println("...")
      println()
      // scalastyle:on println
    }
  }
  foreachRDD(context.sparkContext.clean(foreachFunc), displayInnerRDDOps = false)
}

private def foreachRDD(
     foreachFunc: (RDD[T], Time) => Unit,
     displayInnerRDDOps: Boolean): Unit = {
   new ForEachDStream(this,
     context.sparkContext.clean(foreachFunc, false), displayInnerRDDOps).register()
 }
```

在这里将输出操作的Dstream注册进入了DstreamGraph的outputDstream中
```scala
/**
 * Register this streaming as an output stream. This would ensure that RDDs of this
 * DStream will be generated.
 */
private[streaming] def register(): DStream[T] = {
  ssc.graph.addOutputStream(this)
  this
}
```

还有就是Dstream中outputStreamArray中的action是如何触发job的,其实在jobGenertor中通过定时器RecurringTimer来实现的，那就再来看下这个定时器，RecurringTimer是在JobGneretor中进行实例化的
```scala
  private val timer = new RecurringTimer(clock, ssc.graph.batchDuration.milliseconds,
    longTime => eventLoop.post(GenerateJobs(new Time(longTime))), "JobGenerator")
```

来看下RecurringTimer执行的内容
```scala
private[streaming]
class RecurringTimer(clock: Clock, period: Long, callback: (Long) => Unit, name: String)
  extends Logging {

  private val thread = new Thread("RecurringTimer - " + name) {
    setDaemon(true)
    override def run() { loop }
  }
  
  
private def triggerActionForNextInterval(): Unit = {
  clock.waitTillTime(nextTime)
  callback(nextTime)
  prevTime = nextTime
  nextTime += period
  logDebug("Callback for " + name + " called at time " + prevTime)
}  
```

里面的callback方法是关键，我们顺着来看下callback方法执行的内容
```scala
private val timer = new RecurringTimer(clock, ssc.graph.batchDuration.milliseconds,
   longTime => eventLoop.post(GenerateJobs(new Time(longTime))), "JobGenerator")
    
  /** Processes all events */
private def processEvent(event: JobGeneratorEvent) {
  logDebug("Got event " + event)
  event match {
    case GenerateJobs(time) => generateJobs(time)
    case ClearMetadata(time) => clearMetadata(time)
    case DoCheckpoint(time, clearCheckpointDataLater) =>
      doCheckpoint(time, clearCheckpointDataLater)
    case ClearCheckpointData(time) => clearCheckpointData(time)
  }
}   

/** Generate jobs and perform checkpoint for the given `time`.  */
private def generateJobs(time: Time) {
  // Set the SparkEnv in this thread, so that job generation code can access the environment
  // Example: BlockRDDs are created in this thread, and it needs to access BlockManager
  // Update: This is probably redundant after threadlocal stuff in SparkEnv has been removed.
  SparkEnv.set(ssc.env)
  Try {
    jobScheduler.receiverTracker.allocateBlocksToBatch(time) // allocate received blocks to batc
    graph.generateJobs(time) // generate jobs using allocated block
  } match {
    case Success(jobs) =>
      val streamIdToInputInfos = jobScheduler.inputInfoTracker.getInfo(time)
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
      val jobOption = outputStream.generateJob(time)
      jobOption.foreach(_.setCallSite(outputStream.creationSite))
      jobOption
    }
  }
  logDebug("Generated " + jobs.length + " jobs for time " + time)
  jobs
}
    
```

在上面已经将要输出的DStream存放于DStreamGraph的outputStreams数组中，接下来就是具体的执行
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
***


## 再看DStream

第一：inputDStream是如何产生RDD的，还是以SocketInputDStraem为例:
```scala
/**
 * Generates RDDs with blocks received by the receiver of this stream. */
override def compute(validTime: Time): Option[RDD[T]] = {
  val blockRDD = {
    if (validTime < graph.startTime) {
      // If this is called for any time before the start time of the context,
      // then this returns an empty RDD. This may happen when recovering from a
      // driver failure without any write ahead log to recover pre-failure data.
      new BlockRDD[T](ssc.sc, Array.empty)
    } else {
      // Otherwise, ask the tracker for all the blocks that have been allocated to this stream
      // for this batch
      val receiverTracker = ssc.scheduler.receiverTracker
      val blockInfos = receiverTracker.getBlocksOfBatch(validTime).getOrElse(id, Seq.empty)
      // Register the input blocks information into InputInfoTracker
      val inputInfo = StreamInputInfo(id, blockInfos.flatMap(_.numRecords).sum)
      ssc.scheduler.inputInfoTracker.reportInfo(validTime, inputInfo)
      // Create the BlockRDD
      createBlockRDD(validTime, blockInfos)
    }
  }
  Some(blockRDD)
}
```

第二:Transform级别的DStream:例如:FlatMappedDStream,在它的comput方法中,使用parent.getOrCompute来获取父Dstream产生的RDD，然后使用父Dstream产生的RDD来执行map方法（此map方法是基于RDD的map方法）；此处可以发现SparkStreaming是对SparkCore的一层抽象，而SparkStreaming的实际执行还是基于sparkCore实体来执行的；
```scala
  override def compute(validTime: Time): Option[RDD[U]] = {
    parent.getOrCompute(validTime).map(_.flatMap(flatMapFunc))
  }
```

第三:再看Action级别的DStream: 例如:print(), 在foreachFunc方法中就是基于RDD进行操作的；
```scala
/**
 * Print the first num elements of each RDD generated in this DStream. This is an output
 * operator, so this DStream will be registered as an output stream and there materialized
 */
def print(num: Int): Unit = ssc.withScope {
  def foreachFunc: (RDD[T], Time) => Unit = {
    (rdd: RDD[T], time: Time) => {
      val firstNum = rdd.take(num + 1)
      // scalastyle:off println
      println("-------------------------------------------")
      println("Time: " + time)
      println("-------------------------------------------")
      firstNum.take(num).foreach(println)
      if (firstNum.length > num) println("...")
      println()
      // scalastyle:on println
    }
  }
  foreachRDD(context.sparkContext.clean(foreachFunc), displayInnerRDDOps = false)
}
```

而foreachDStream中的compute方法为空,是因为foreachDStream是job中最后的Action操作，而generateJob内执行的执行发放foreachFunc中执行的还是RDD的输出操作；
```scala
private[streaming]
class ForEachDStream[T: ClassTag] (
    parent: DStream[T],
    foreachFunc: (RDD[T], Time) => Unit,
    displayInnerRDDOps: Boolean
  ) extends DStream[Unit](parent.ssc) {

  override def dependencies: List[DStream[_]] = List(parent)

  override def slideDuration: Duration = parent.slideDuration

  override def compute(validTime: Time): Option[RDD[Unit]] = None

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
}
```

在JobGenerator中,定时器RecurringTimer不停的执行triggerActionForNextInterval的callback方法
```scala
  private def triggerActionForNextInterval(): Unit = {
    clock.waitTillTime(nextTime)
    callback(nextTime)
    prevTime = nextTime
    nextTime += period
    logDebug("Callback for " + name + " called at time " + prevTime)
  }
```
callback方法具体执行的就是DStreamGraph中的generateJobs方法，
```scala
  def generateJobs(time: Time): Seq[Job] = {
    logDebug("Generating jobs for time " + time)
    val jobs = this.synchronized {
      outputStreams.flatMap { outputStream =>
        val jobOption = outputStream.generateJob(time)
        jobOption.foreach(_.setCallSite(outputStream.creationSite))
        jobOption
      }
    }
    logDebug("Generated " + jobs.length + " jobs for time " + time)
    jobs
  }
```

DStreamGraph中的generateJobs方法执行的是DStream的generateJob方法，在此方法中最终执行的是SparkCore的runJob方法;
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
而Dstream的generateJob方法中调用DStream的gerorcompute,在此方法中根据时间在generatedRDDs中存储对应Time的RDD数组,其他每个DStream都有一个这样的数据结构来根据Time来存储对应的RDD；
```scala
  private[streaming] final def getOrCompute(time: Time): Option[RDD[T]] = {
    // If RDD was already generated, then retrieve it from HashMap,
    // or else compute the RDD
    generatedRDDs.get(time).orElse {
      // Compute the RDD if time is valid (e.g. correct time in a sliding window)
      // of RDD generation, else generate nothing.
      if (isTimeValid(time)) {

        val rddOption = createRDDWithLocalProperties(time, displayInnerRDDOps = false) {
          // Disable checks for existing output directories in jobs launched by the streaming
          // scheduler, since we may need to write output to an existing directory during checkpoint
          // recovery; see SPARK-4835 for more details. We need to have this call here because
          // compute() might cause Spark jobs to be launched.
          PairRDDFunctions.disableOutputSpecValidation.withValue(true) {
            compute(time)
          }
        }

        rddOption.foreach { case newRDD =>
          // Register the generated RDD for caching and checkpointing
          if (storageLevel != StorageLevel.NONE) {
            newRDD.persist(storageLevel)
            logDebug(s"Persisting RDD ${newRDD.id} for time $time to $storageLevel")
          }
          if (checkpointDuration != null && (time - zeroTime).isMultipleOf(checkpointDuration)) {
            newRDD.checkpoint()
            logInfo(s"Marking RDD ${newRDD.id} for time $time for checkpointing")
          }
          generatedRDDs.put(time, newRDD)
        }
        rddOption
      } else {
        None
      }
    }
  }
```
接下来就是将基于RDD产生的Job提交给cluster进行执行……………

### 总结:
其实DStream只是基于RDD的一个抽象的模板,而DstreamGreaph就是生成DAG的模板，最终每个Dstream都会生成一个以time为key，RDD[T]为value的数据结     构用来存储基于模板生成的RDD，SparkStreaming最终做执行操作的还是SparkCore的RDD；


















