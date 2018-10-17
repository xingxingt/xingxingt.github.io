---
layout:     post
title:      Spark源码之DAGScheduler
subtitle:   Spark源码之DAGScheduler介绍
date:       2018-10-15
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>Spark源码之DAGScheduler介绍篇
> 

    Spark Application中的RDD经过一系列的Transformation操作后由Action算子导致了SparkContext.runjob的执行,
    之后执行DAGScheduler.runJob(),最终导致了DAGScheduler中的submitJob的执行,在DAGScheduler中完成sparkJob
    的DAG划分,并将生成的TaskSet交给taskScheduler处理，如下图所示:

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fwaee7mjwfj314i0k8q81.jpg)

### 深入DAGScheduler源码

    我们从RDD的Action操作产生的SparkContext.runjob说起,在SparkContext.runjob()中最终调用了
    dagScheduler.runJob()方法；如下图所示:
    
```
/**
   * Return an array that contains all of the elements in this RDD.
   */
  def collect(): Array[T] = withScope {
    val results = sc.runJob(this, (iter: Iterator[T]) => iter.toArray)
    Array.concat(results: _*)
  }
```
```
  /**
   * Run a function on a given set of partitions in an RDD and pass the results to the given
   * handler function. This is the main entry point for all actions in Spark.
   */
  def runJob[T, U: ClassTag](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      resultHandler: (Int, U) => Unit): Unit = {
    if (stopped.get()) {
      throw new IllegalStateException("SparkContext has been shutdown")
    }
    val callSite = getCallSite
    val cleanedFunc = clean(func)
    logInfo("Starting job: " + callSite.shortForm)
    if (conf.getBoolean("spark.logLineage", false)) {
      logInfo("RDD's recursive dependencies:\n" + rdd.toDebugString)
    }
    dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)
    progressBar.foreach(_.finishAll())
    rdd.doCheckpoint()
  }
```

    接着看DAGScheduler.runjob()方法,在方法里面调用了submitJob()方法，并且返回一个JobWaiter监听submitJob的
    结果，并对结果做出相应的处理;

```
  def runJob[T, U](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      callSite: CallSite,
      resultHandler: (Int, U) => Unit,
      properties: Properties): Unit = {
    val start = System.nanoTime
    val waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)
    waiter.awaitResult() match {
      case JobSucceeded =>
        logInfo("Job %d finished: %s, took %f s".format
          (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
      case JobFailed(exception: Exception) =>
        logInfo("Job %d failed: %s, took %f s".format
          (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
        // SPARK-8644: Include user stack trace in exceptions coming from DAGScheduler.
        val callerStackTrace = Thread.currentThread().getStackTrace.tail
        exception.setStackTrace(exception.getStackTrace ++ callerStackTrace)
        throw exception
    }
  }
```


    进入submitJob方法，如下图所示，先生成一个jobId，紧接着使用eventProcessLoop发送一个JobSubmitted的消息，那我
    们就要看下这个eventProcessLoop是什么了；
    
```
  def submitJob[T, U](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      callSite: CallSite,
      resultHandler: (Int, U) => Unit,
      properties: Properties): JobWaiter[U] = {
    // Check to make sure we are not launching a task on a partition that does not exist.
    val maxPartitions = rdd.partitions.length
    partitions.find(p => p >= maxPartitions || p < 0).foreach { p =>
      throw new IllegalArgumentException(
        "Attempting to access a non-existent partition: " + p + ". " +
          "Total number of partitions: " + maxPartitions)
    }

    val jobId = nextJobId.getAndIncrement()
    if (partitions.size == 0) {
      // Return immediately if the job is running 0 tasks
      return new JobWaiter[U](this, jobId, 0, resultHandler)
    }

    assert(partitions.size > 0)
    val func2 = func.asInstanceOf[(TaskContext, Iterator[_]) => _]
    val waiter = new JobWaiter(this, jobId, partitions.size, resultHandler)
    eventProcessLoop.post(JobSubmitted(
      jobId, rdd, func2, partitions.toArray, callSite, waiter,
      SerializationUtils.clone(properties)))
    waiter
  }
```

    查看源码发现eventProcessLoop是一个消息循环体，而且他还继承了EventLoop，再看下EventLoop的代码，发现EventLoop
    是一个时间处理器，在内部使用BlockingQueue去存储接受到的消息事件，用一个守护线程去执行onReceive,而onReceive方法
    在DAGSchedulerEventProcessLoop中已经被重写，而在onReceive方法中调用doOnReceive方法做具体的事件处理;


```
  private[scheduler] val eventProcessLoop = new DAGSchedulerEventProcessLoop(this)
```
```
private[scheduler] class DAGSchedulerEventProcessLoop(dagScheduler: DAGScheduler)
  extends EventLoop[DAGSchedulerEvent]("dag-scheduler-event-loop") with Logging {

  private[this] val timer = dagScheduler.metricsSource.messageProcessingTimer

  /**
   * The main event loop of the DAG scheduler.
   */
  override def onReceive(event: DAGSchedulerEvent): Unit = {
    val timerContext = timer.time()
    try {
      doOnReceive(event)
    } finally {
      timerContext.stop()
    }
  }

  private def doOnReceive(event: DAGSchedulerEvent): Unit = event match {
    case JobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties) =>
      dagScheduler.handleJobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties)

    case MapStageSubmitted(jobId, dependency, callSite, listener, properties) =>
      dagScheduler.handleMapStageSubmitted(jobId, dependency, callSite, listener, properties)
```
```
private[spark] abstract class EventLoop[E](name: String) extends Logging {

  private val eventQueue: BlockingQueue[E] = new LinkedBlockingDeque[E]()

  private val stopped = new AtomicBoolean(false)

  private val eventThread = new Thread(name) {
    setDaemon(true)

    override def run(): Unit = {
      try {
        while (!stopped.get) {
          val event = eventQueue.take()
          try {
            onReceive(event)
          } catch {
            case NonFatal(e) => {
              try {
                onError(e)
              } catch {
                case NonFatal(e) => logError("Unexpected error in " + name, e)
              }
            }
          }
        }
      } catch {
        case ie: InterruptedException => // exit even if eventQueue is not empty
        case NonFatal(e) => logError("Unexpected error in " + name, e)
      }
    }
```

    ok，我们已经知道了在DAGScheduler中的消息事件是如何处理的，那么我们还是言归正传，继续看在SubmitJob的方法中使用
    eventProcessLoop发送一个JobSubmitted消息给自己，也就是在doOnReceive方法中找到JobSubmitted事件，在此方法中
    又继续调用了dagScheduler.handleJobSubmitted方法；如下图源代码所示:

```
  private def doOnReceive(event: DAGSchedulerEvent): Unit = event match {
    case JobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties) =>
      dagScheduler.handleJobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties)
```

    那我们就进入handleJobSubmitted方法，我们先看下此方法中的finalStage = newResultStage(....)代码,在这里要说一下
    在一个DAG中最后一个Stage叫做resultStage,而前面的所有stage都叫做shuffleMapStage;而newResultStage(....)方法
    就是根据提供的jobId生成一个ResultStage,如下图源码所示:

![](https://ws1.sinaimg.cn/large/006tNbRwly1fwaziw935hj31fg0wy40d.jpg)

    那我们就要看下ResultStage是如何生成的，我们可以看到,在newResultStage方法中先通过getParentStagesAndId方法获取
    ResultStage的所有父stage，然后在new出一个ResultStage实例来;
    紧接着我们把代码追踪到getParentStages方法中,这个方法可以根据提供的RDD创建一个父stage的列表，我们再来剖析下这个方法; 
    在这个方法中先实例出两个数据结构parents,visited和waitingForVisit,parents是用来存放所有父类stage的数据集，而
    visited使用来存储已被访问的RDD，而waitingForVisit则是等待被访问的RDD数据集;
    在下面代码中,先将传入的RDD放入到waitingForVisit数据集中，然后循环waitingForVisit中所有的RDD，每次循环调用visit
    方法。在visit方法中它利用RDD的dependencies从后向前建立依赖关系，在遍历RDD的dependencies时如果是shufDep就生成
    一个getShuffleMapStage放入到parents数据集中，如果不是就将该dependencie对应的RDD放入到waitingForVisit中，等待
    下一次遍历，最终该方法返回一个父stage的数据集parents给newResultStage方法；
    而且在newResultStage中new出ResultStage,并将stage的数据集parents存放于该ResultStage中;

![](https://ws3.sinaimg.cn/large/006tNbRwly1fwazr8g0dej31fa0g80tg.jpg)
![](https://ws1.sinaimg.cn/large/006tNbRwly1fwb00ckoawj31f006aq37.jpg)
![](https://ws1.sinaimg.cn/large/006tNbRwly1fwb018bh0qj31dk10kwg9.jpg)
     
    经过一番折腾后我们再回到handleJobSubmitted方法,现在我们已经获取到了该job的ResultStage，和该ResultStage的父
    stages然后生成一个ActiveJob在DAGScheduler中,以及打印一些stage的信息， 这里有调用getMissingParentStages()
    方法，这个我们在接下来的submitStage方法中讲述，源代码如下图所示:
    
![](https://ws3.sinaimg.cn/large/006tNbRwly1fwb19lz1c1j31im0pqq4q.jpg)

    接下来进入submitStage方法中，在这个方法中，会先调用getMissingParentStages()方法，将检查是否有缺失的stage,如果
    有则使用递归的方式将该stage提交，并将该stage加入到waitingStages中，也可以再看下getMissingParentStages()方法，
    该方法和getParentStages()方法一样,只不过该方法会判断Stage中的rdds是否在cache中存在，cacheLocs 维护着RDD的
    partitions的location信息,该信息是TaskLocation的实例。如果从cacheLocs中获取到partition的location信息直接
    返回，若获取不到：如果RDD的存储级别为空返回nil；
    
![](https://ws2.sinaimg.cn/large/006tNbRwly1fwb1jbm6xvj31iq0u075s.jpg)
![](https://ws4.sinaimg.cn/large/006tNbRwly1fwb2gwuwmfj319i14ywgh.jpg)
    
    
    
