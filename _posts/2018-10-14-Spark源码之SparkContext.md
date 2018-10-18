---
layout:     post
title:      Spark源码之SparkContext
subtitle:   Spark源码之SparkContext介绍
date:       2018-10-14
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>Spark源码之SparkContext介绍篇
> 

### SparkContext介绍
SparkContext作为spark的主入口类，SparkContext表示一个spark集群的链接,它会用在创建RDD,计数器以及广播变量
在Spark集群；
SparkContext特性:
1. Spark的程序编写时基于SparkContext的，具体包括两方面:
  Spark编程的核心基础--RDD，是由SparkContext来最初创建（第一个RDD一定是由SparkContext来创建的）；
  Spark程序的调度优化优势基于SparkContext；
2. Spark程序的注册是通过SparkContext实例化的时候产生的对象来完成的（其实是SchedulerBackend来注册程序）
3. Spark程序运行的时候通过Cluster Manager获取具体的计算资源，计算资源的获取也是通过SparkContext产生的对象来申请的
  (其实是SchedulerBackend来获取计算资源的）;
4. sparkContext崩溃或者结束的时候整个Spark程序也就结束！
    
### SparkContext构建的三大核心对象
SparkContext构建的顶级三大核心对象：DAGScheduler，TaskScheduler，ScheduleBackend；
DAGScheduler是面向job的stage的高层调度器；
TaskScheduler是一个接口，根据具体的cluster manager的不同会有不同的实现，standalone模式下具体的实现是TaskSchedulerImpl;
SchedulerBackend是一个接口，根据具体的ClusterManager的不同会有不同的实现，Standalone模式下具体的实现是SparkDeploySchedulerBackend；

### 深入三大核心对象
下面我们从源代码层面看下三大核心对象是如何在SparkContext中产生的；
根据下面代码可以看到DAGScheduler，TaskScheduler,ScheduleBackend在SparkContext内部的实例化,DAGScheduler在这里直接实例化出来，而TaskScheduler,ScheduleBackend则在createTaskScheduler()方法中根据集群类型来实例化,因为这跟随后的任务调度有关;

```scala
 // Create and start the scheduler
    val (sched, ts) = SparkContext.createTaskScheduler(this, master)
    _schedulerBackend = sched
    _taskScheduler = ts
    _dagScheduler = new DAGScheduler(this)
    _heartbeatReceiver.ask[Boolean](TaskSchedulerIsSet)

    // start TaskScheduler after taskScheduler sets DAGScheduler reference in DAGScheduler's
    // constructor
    _taskScheduler.start()

```
  
再进入createTaskScheduler()方法,在这个方法内是根据集群以什么方式启动的来实例出相应的TaskScheduler，ScheduleBackend的子类；

```scala
private def createTaskScheduler(
      sc: SparkContext,
      master: String): (SchedulerBackend, TaskScheduler) = {
    import SparkMasterRegex._

    // When running locally, don't try to re-execute tasks on failure.
    val MAX_LOCAL_TASK_FAILURES = 1

    master match {
      case "local" =>
        val scheduler = new TaskSchedulerImpl(sc, MAX_LOCAL_TASK_FAILURES, isLocal = true)
        val backend = new LocalBackend(sc.getConf, scheduler, 1)
        scheduler.initialize(backend)
        (backend, scheduler)

      case LOCAL_N_REGEX(threads) =>
        def localCpuCount: Int = Runtime.getRuntime.availableProcessors()
        // local[*] estimates the number of cores on the machine; local[N] uses exactly N threads.
        val threadCount = if (threads == "*") localCpuCount else threads.toInt
        if (threadCount <= 0) {
          throw new SparkException(s"Asked to run locally with $threadCount threads")
        }
        val scheduler = new TaskSchedulerImpl(sc, MAX_LOCAL_TASK_FAILURES, isLocal = true)
        val backend = new LocalBackend(sc.getConf, scheduler, threadCount)
        scheduler.initialize(backend)
        (backend, scheduler)

      case LOCAL_N_FAILURES_REGEX(threads, maxFailures) =>
      .......
```


我们以Standalone模式为例讲解,参看下图源码，可见TaskScheduler实例的子类为TaskSchedulerImpl，SchedulerBackend的实例化的子类是SparkDeploySchedulerBackend;

```scala
case SPARK_REGEX(sparkUrl) =>
        val scheduler = new TaskSchedulerImpl(sc)
        val masterUrls = sparkUrl.split(",").map("spark://" + _)
        val backend = new SparkDeploySchedulerBackend(scheduler, sc, masterUrls)
        scheduler.initialize(backend)
        (backend, scheduler)
``` 

 我们随后会单独讲解TaskSchedulerImpl和DAGScheduler,现在我们先专注于SparkDeploySchedulerBackend；
 SparkDeploySchedulerBackend核心功能:
 1. 负责与Master连接注册当前程序！
 2. 接收集群中为当前应用程序分配的计算资源，Executor的注册并且管理Executors！
 3. 负责发送Task到具体的Executor执行！
 4. 它的父类CoarseGrainedSchedulerBackend中的DriverEndpoint就是Spark应用程序中的Driver;
 
我们进入SparkDeploySchedulerBackend代码中，首先看它的start()方法；

```scala
 override def start() {
    super.start()
    launcherBackend.connect()

    // The endpoint for executors to talk to us
    val driverUrl = rpcEnv.uriOf(SparkEnv.driverActorSystemName,
      RpcAddress(sc.conf.get("spark.driver.host"), sc.conf.get("spark.driver.port").toInt),
      CoarseGrainedSchedulerBackend.ENDPOINT_NAME)
    val args = Seq(
      "--driver-url", driverUrl,
      "--executor-id", "{{EXECUTOR_ID}}",
      "--hostname", "{{HOSTNAME}}",
      "--cores", "{{CORES}}",
      "--app-id", "{{APP_ID}}",
      "--worker-url", "{{WORKER_URL}}")
    val extraJavaOpts = sc.conf.getOption("spark.executor.extraJavaOptions")
      .map(Utils.splitCommandString).getOrElse(Seq.empty)
    val classPathEntries = sc.conf.getOption("spark.executor.extraClassPath")
      .map(_.split(java.io.File.pathSeparator).toSeq).getOrElse(Nil)
    val libraryPathEntries = sc.conf.getOption("spark.executor.extraLibraryPath")
      .map(_.split(java.io.File.pathSeparator).toSeq).getOrElse(Nil)
    ......
```
    
在SparkDeploySchedulerBackend的Start()方法中，super.start()其实执行的是它的父类CoarseGrainedSchedulerBackend的start方法，如下代码所示,可以看到执行了createDriverEndpoint(properties)方法，我们进入看下，在createDriverEndpoint()方法中实例化了一个DriverEndpoint，而DriverEndpoint其实是一个消息循环体,其实这个DriverEndPoint就是Spark应用程序中的Driver，他内部可以接收处理其他组件发来的消息；

```scala
  override def start() {
    val properties = new ArrayBuffer[(String, String)]
    for ((key, value) <- scheduler.sc.conf.getAll) {
      if (key.startsWith("spark.")) {
        properties += ((key, value))
      }
    }

    // TODO (prashant) send conf instead of properties
    driverEndpoint = rpcEnv.setupEndpoint(ENDPOINT_NAME, createDriverEndpoint(properties))
  }
  
  protected def createDriverEndpoint(properties: Seq[(String, String)]): DriverEndpoint = {
    new DriverEndpoint(rpcEnv, properties)
  }
  
  class DriverEndpoint(override val rpcEnv: RpcEnv, sparkProperties: Seq[(String, String)])
   extends ThreadSafeRpcEndpoint with Logging {

   // If this DriverEndpoint is changed to support multiple threads,
   // then this may need to be changed so that we don't share the serializer
   // instance across threads
   private val ser = SparkEnv.get.closureSerializer.newInstance()

   override protected def log = CoarseGrainedSchedulerBackend.this.log

   protected val addressToExecutorId = new HashMap[RpcAddress, String]

   private val reviveThread =
     ThreadUtils.newDaemonSingleThreadScheduledExecutor("driver-revive-thread")

    ......
```

在SparkDeploySchedulerBackend的Start()方法中执行完super.start()后，进行一系列的参数配置后，就开始了Spark应用程序的处理,如下源码所示:
在这里实例化出APPClient对象,并调用client.start()方法,进入APPClient的start()方法,在这个方法里new了一个ClientEndpoint实例,其实ClientEndpoint他也是一个消息循环体;

```scala
    // Start executors with a few necessary configs for registering with the scheduler
    val sparkJavaOpts = Utils.sparkJavaOpts(conf, SparkConf.isExecutorStartupConf)
    val javaOpts = sparkJavaOpts ++ extraJavaOpts
    val command = Command("org.apache.spark.executor.CoarseGrainedExecutorBackend",
      args, sc.executorEnvs, classPathEntries ++ testingClassPath, libraryPathEntries, javaOpts)
    val appUIAddress = sc.ui.map(_.appUIAddress).getOrElse("")
    val coresPerExecutor = conf.getOption("spark.executor.cores").map(_.toInt)
    val appDesc = new ApplicationDescription(sc.appName, maxCores, sc.executorMemory,
      command, appUIAddress, sc.eventLogDir, sc.eventLogCodec, coresPerExecutor)
    client = new AppClient(sc.env.rpcEnv, masters, appDesc, this, conf)
    client.start()
    
    def start() {
       //AppClient 的start方法
       // Just launch an rpcEndpoint; it will call back into the listener.
       endpoint.set(rpcEnv.setupEndpoint("AppClient", new ClientEndpoint(rpcEnv)))
   }
```

再看ClientEndpoint的onStart()方法的,这里有个重要的地方，之前我也很疑惑就是Application是如何注册给当前的集群中的master,在onStart()中执行registerWithMaster(1)，向master注册Application;
如下代码所示:
这个跟worker向Master注册差不多，向每个Master发送RegisterApplication的消息,Master接收到注册消息并处理;

```scala
    //ClientEndpoint的onStart()方法
    override def onStart(): Unit = {
      try {
        registerWithMaster(1)
      } catch {
        case e: Exception =>
          logWarning("Failed to connect to master", e)
          markDisconnected()
          stop()
      }
    }
    //向master注册
    private def registerWithMaster(nthRetry: Int) {
      registerMasterFutures.set(tryRegisterAllMasters())
      registrationRetryTimer.set(registrationRetryThread.scheduleAtFixedRate(new Runnable {
        override def run(): Unit = {
          Utils.tryOrExit {
            if (registered.get) {
              registerMasterFutures.get.foreach(_.cancel(true))
              registerMasterThreadPool.shutdownNow()
            } else if (nthRetry >= REGISTRATION_RETRIES) {
              markDead("All masters are unresponsive! Giving up.")
            } else {
              registerMasterFutures.get.foreach(_.cancel(true))
              registerWithMaster(nthRetry + 1)
            }
          }
        }
      }, REGISTRATION_TIMEOUT_SECONDS, REGISTRATION_TIMEOUT_SECONDS, TimeUnit.SECONDS))
    }
    //向所有的master注册
    private def tryRegisterAllMasters(): Array[JFuture[_]] = {
      for (masterAddress <- masterRpcAddresses) yield {
        registerMasterThreadPool.submit(new Runnable {
          override def run(): Unit = try {
            if (registered.get) {
              return
            }
            logInfo("Connecting to master " + masterAddress.toSparkURL + "...")
            val masterRef =
              rpcEnv.setupEndpointRef(Master.SYSTEM_NAME, masterAddress, Master.ENDPOINT_NAME)
            masterRef.send(RegisterApplication(appDescription, self))
          } catch {
            case ie: InterruptedException => // Cancelled
            case NonFatal(e) => logWarning(s"Failed to connect to master $masterAddress", e)
          }
        })
      }
    }
    //Master 接收到Application的注册
    case RegisterApplication(description, driver) => {
      // TODO Prevent repeated registrations from some driver
      if (state == RecoveryState.STANDBY) {
        // ignore, don't send response
      } else {
        logInfo("Registering app " + description.name)
        val app = createApplication(description, driver)
        registerApplication(app)
        logInfo("Registered app " + description.name + " with ID " + app.id)
        persistenceEngine.addApplication(app)
        driver.send(RegisteredApplication(app.id, self))
        schedule()
      }
    }
```

回过头来再看SparkDeploySchedulerBackend的Start()方法,具体完成了Driver的启动和Application的注册;
在Application提交时有个重要的地方，如下图所示,在appDesc中有个command,是这样的:
当通过SparkDeploySchedulerBackend注册程序给Master的时候会把上述command提交给master，master发指令给Worker去启动Excutor所在的进程的时候加载main方法所在的入口类，就是command中的CoarseGrainedExcutorBackend，当然你也可以自己实现excutorBackend！CoarseGrainedExecutorBackend中启动Executor（Excutor是先注册再实例化的）,Excutor通过线程池并发执行Task！关于Executor部分我们随后详细阐述；

```scala
    // Start executors with a few necessary configs for registering with the scheduler
    val sparkJavaOpts = Utils.sparkJavaOpts(conf, SparkConf.isExecutorStartupConf)
    val javaOpts = sparkJavaOpts ++ extraJavaOpts
    val command = Command("org.apache.spark.executor.CoarseGrainedExecutorBackend",
      args, sc.executorEnvs, classPathEntries ++ testingClassPath, libraryPathEntries, javaOpts)
    val appUIAddress = sc.ui.map(_.appUIAddress).getOrElse("")
    val coresPerExecutor = conf.getOption("spark.executor.cores").map(_.toInt)
    val appDesc = new ApplicationDescription(sc.appName, maxCores, sc.executorMemory,
      command, appUIAddress, sc.eventLogDir, sc.eventLogCodec, coresPerExecutor)
    client = new AppClient(sc.env.rpcEnv, masters, appDesc, this, conf)
    client.start()
```
  
    
#### Ok,这里主要讲述了SparkContext内部的核心对象，以及Driver的生成和Application的注册;SparkContext的内容还是挺多的,随后慢慢添加；
![](https://ws3.sinaimg.cn/large/006tNbRwgy1fw93gdam07j31kw0kngmb.jpg)
    
    
    
