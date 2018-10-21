---
layout:     post
title:      SparkStreaming源码之JobScheduler
subtitle:   SparkStreaming源码之JobScheduler
date:       2018-09-08
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>SparkStreaming源码之JobScheduler篇
> 

首先看下JobScheduler这个类是在什么时候被实例化的，打开StreamingContext代码可见：
```scala
  private[streaming] val scheduler = new JobScheduler(this)

  private[streaming] val waiter = new ContextWaiter

  private[streaming] val progressListener = new StreamingJobProgressListener(this)
```

再看下job的产生者jobGenerator是如何将生成的job传递给JobScheduler的
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

#### JobScheduler处理提交上来的job，并将job存放在jobSet的数据结构中
```scala
  def submitJobSet(jobSet: JobSet) {
    if (jobSet.jobs.isEmpty) {
      logInfo("No jobs added for time " + jobSet.time)
    } else {
      listenerBus.post(StreamingListenerBatchSubmitted(jobSet.toBatchInfo))
      jobSets.put(jobSet.time, jobSet)
      jobSet.jobs.foreach(job => jobExecutor.execute(new JobHandler(job)))
      logInfo("Added jobs for time " + jobSet.time)
    }
  }
```

SparkStreaming在一个Application中能够同时运行多个job的，其实就是使用多线程来实现
```scala
  //todo jobSet数据结构
  private val jobSets: java.util.Map[Time, JobSet] = new ConcurrentHashMap[Time, JobSet]
  private val numConcurrentJobs = ssc.conf.getInt("spark.streaming.concurrentJobs", 1)
  //todo 使用线程池来运行多个job事件
  private val jobExecutor =
    ThreadUtils.newDaemonFixedThreadPool(numConcurrentJobs, "streaming-job-executor"
```

JobScheduler负责job的调度，在内部是使用一个消息循环体来处理job的各种事件，而这个消息循环体也是在JobSchduler的start方法中实例化
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
    jobGenerator.start()
    logInfo("Started JobScheduler")
  }
```

看下这个消息循环体具体的内容，可见Job的启动，完成，还有错误处理都在这里，具体方法可以点进去看
```scala
  private def processEvent(event: JobSchedulerEvent) {
    try {
      event match {
        case JobStarted(job, startTime) => handleJobStart(job, startTime)//todo 启动job事件处理
        case JobCompleted(job, completedTime) => handleJobCompletion(job, completedTime)//todo job完成事件处理
        case ErrorReported(m, e) => handleError(m, e)//todo 异常事件处理
      }
    } catch {
      case e: Throwable =>
        reportError("Error in job scheduler", e)
    }
  }
```

现在看下一个job的启动，在SubmitJobSet方法，JobExecutor线程池去执行每个JobHandler
```scala
  def submitJobSet(jobSet: JobSet) {
    if (jobSet.jobs.isEmpty) {
      logInfo("No jobs added for time " + jobSet.time)
    } else {
      listenerBus.post(StreamingListenerBatchSubmitted(jobSet.toBatchInfo))
      jobSets.put(jobSet.time, jobSet)
      //todo 在这里处理jobSet里面的每个job
      jobSet.jobs.foreach(job => jobExecutor.execute(new JobHandler(job)))
      logInfo("Added jobs for time " + jobSet.time)
    }
  }
```
看下jobHandler这个线程的run方法
```scala
  private class JobHandler(job: Job) extends Runnable with Logging {
    import JobScheduler._

    def run() {
      try {
        val formattedTime = UIUtils.formatBatchTime(
          job.time.milliseconds, ssc.graph.batchDuration.milliseconds, showYYYYMMSS = false)
        val batchUrl = s"/streaming/batch/?id=${job.time.milliseconds}"
        val batchLinkText = s"[output operation ${job.outputOpId}, batch time ${formattedTime}]"

        ssc.sc.setJobDescription(
          s"""Streaming job from <a href="$batchUrl">$batchLinkText</a>""")
        ssc.sc.setLocalProperty(BATCH_TIME_PROPERTY_KEY, job.time.milliseconds.toString)
        ssc.sc.setLocalProperty(OUTPUT_OP_ID_PROPERTY_KEY, job.outputOpId.toString)

        // We need to assign `eventLoop` to a temp variable. Otherwise, because
        // `JobScheduler.stop(false)` may set `eventLoop` to null when this method is running, then
        // it's possible that when `post` is called, `eventLoop` happens to null.
        var _eventLoop = eventLoop
        if (_eventLoop != null) {
          //todo 这里给自己发消息启动job，其实就是打出一些日志
          _eventLoop.post(JobStarted(job, clock.getTimeMillis()))
          // Disable checks for existing output directories in jobs launched by the streaming
          // scheduler, since we may need to write output to an existing directory during checkpoint
          // recovery; see SPARK-4835 for more details.
          PairRDDFunctions.disableOutputSpecValidation.withValue(true) {
            //todo 这里是关键 一个job内容的运行
            job.run()
          }
          _eventLoop = eventLoop
          if (_eventLoop != null) {
            //todo 这里给自己发消息jo完成，其实也是打出一些日志
            _eventLoop.post(JobCompleted(job, clock.getTimeMillis()))
          }
        } else {
          // JobScheduler has been stopped.
        }
      } finally {
        ssc.sc.setLocalProperty(JobScheduler.BATCH_TIME_PROPERTY_KEY, null)
        ssc.sc.setLocalProperty(JobScheduler.OUTPUT_OP_ID_PROPERTY_KEY, null)
      }
    }
  }
}
```
看下最终的run方法,这个run方法执行的是job的输出代码的方法，例如print操作产生的job
```scala
private[streaming]
class Job(val time: Time, func: () => _) {
  private var _id: String = _
  private var _outputOpId: Int = _
  private var isSet = false
  private var _result: Try[_] = null
  private var _callSite: CallSite = null
  private var _startTime: Option[Long] = None
  private var _endTime: Option[Long] = None

  def run() {
    //todo 这里的func()便是你的action操作方法,或者是你传入的输入方法
    _result = Try(func())
  }
  
  //todo print操作
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

**至此 JobScheduler角色的工作以叙述完毕！**


