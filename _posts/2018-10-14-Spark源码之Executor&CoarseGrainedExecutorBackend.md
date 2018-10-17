---
layout:     post
title:      Spark源码之Executor&CoarseGrainedExecutorBackend
subtitle:   Spark源码之Executor&CoarseGrainedExecutorBackend介绍
date:       2018-10-14
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>Spark源码之Executor&CoarseGrainedExecutorBackend介绍篇
> 

### CoarseGrainedExecutorBackend和Executor的关系
我们先说下CoarseGrainedExecutorBackend和Executor这两者的关系，CoarseGrainedExecutorBackend比较直观因为我们在启动Spark集群运行任务通过JPS命令,可以看到有一个CoarseGrainedExecutorBackend这样的进程，其实CoarseGrainedExecutorBackend就是一个进程，而Executor则是一个实例对象，并且Executor是运行在CoarseGrainedExecutorBackend进程中的；再者CoarseGrainedExecutorBackend和Executor是一一对应的；
    
### CoarseGrainedExecutorBackend内幕
既然Executor是运行在CoarseGrainedExecutorBackend进程中，那就先说下这个CoarseGrainedExecutorBackend;
首先我们应该知道CoarseGrainedExecutorBackend是什么时候被实例出来的,我们在【spark源码之SparkContext】中介绍过AppLication的注册和Driver的产生，在APPClient实例化时候传入了一个command，而这个command就是CoarseGrainedExecutorBackend这个类，如下源码所示，Application在注册时把这个command也提交给了Master，master发指令给Worker去启动Excutor所在的进程的时候加载main方法所在的入口类，就是command中的CoarseGrainedExcutorBackend;

```scala
    //在SparkDeploySchedulerBackend中实例化出Application
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
```scala
  //ExecutorRunner
  private def fetchAndRunExecutor() {
    try {
      // Launch the process
      val builder = CommandUtils.buildProcessBuilder(appDesc.command, new SecurityManager(conf),
        memory, sparkHome.getAbsolutePath, substituteVariables)
      val command = builder.command()
      val formattedCommand = command.asScala.mkString("\"", "\" \"", "\"")
      logInfo(s"Launch command: $formattedCommand")

      builder.directory(executorDir)
      builder.environment.put("SPARK_EXECUTOR_DIRS", appLocalDirs.mkString(File.pathSeparator))
      // In case we are running this from within the Spark Shell, avoid creating a "scala"
      // parent process for the executor command
      builder.environment.put("SPARK_LAUNCH_WITH_SCALA", "0")

      // Add webUI log urls
      val baseUrl =
        s"http://$publicAddress:$webUiPort/logPage/?appId=$appId&executorId=$execId&logType="
      builder.environment.put("SPARK_LOG_URL_STDERR", s"${baseUrl}stderr")
      builder.environment.put("SPARK_LOG_URL_STDOUT", s"${baseUrl}stdout")

      process = builder.start()
```
Master在启动一个Excutor所在的进程的时候加载了CoarseGrainedExecutorBackend的main方法，我们进入main方法，先进行进行了一系列的参数初始化之后进入了run()方法，在run()方法中进行环境参数配置后启动RPC通信，并且实例化出CoarseGrainedExecutorBackend；

```scala
   def main(args: Array[String]) {
   
        ......
        
        case ("--app-id") :: value :: tail =>
          appId = value
          argv = tail
        case ("--worker-url") :: value :: tail =>
          // Worker url is used in spark standalone mode to enforce fate-sharing with worker
          workerUrl = Some(value)
          argv = tail
        case ("--user-class-path") :: value :: tail =>
          userClassPath += new URL(value)
          argv = tail
        case Nil =>
        case tail =>
          // scalastyle:off println
          System.err.println(s"Unrecognized options: ${tail.mkString(" ")}")
          // scalastyle:on println
          printUsageAndExit()
      }
    }

    if (driverUrl == null || executorId == null || hostname == null || cores <= 0 ||
      appId == null) {
      printUsageAndExit()
    }

    run(driverUrl, executorId, hostname, cores, appId, workerUrl, userClassPath)
  }
```
```scala
      //main中调用的run方法
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
      workerUrl.foreach { url =>
        env.rpcEnv.setupEndpoint("WorkerWatcher", new WorkerWatcher(env.rpcEnv, url))
      }
      env.rpcEnv.awaitTermination()
      SparkHadoopUtil.get.stopExecutorDelegationTokenRenewer()
```


CoarseGrainedExecutorBackend实例化出来后我们再看它的onStart()方法，在CoarseGrainedExecutorBackend启动后就立即向Driver注册,如下图所示;

```scala
  override def onStart() {
    logInfo("Connecting to driver: " + driverUrl)
    rpcEnv.asyncSetupEndpointRefByURI(driverUrl).flatMap { ref =>
      // This is a very fast action so we can use "ThreadUtils.sameThread"
      driver = Some(ref)
      ref.ask[RegisterExecutorResponse](
        RegisterExecutor(executorId, self, hostPort, cores, extractLogUrls))
    }(ThreadUtils.sameThread).onComplete {
      // This is a very fast action so we can use "ThreadUtils.sameThread"
      case Success(msg) => Utils.tryLogNonFatalError {
        Option(self).foreach(_.send(msg)) // msg must be RegisterExecutorResponse
      }
      case Failure(e) => {
        logError(s"Cannot register with driver: $driverUrl", e)
        System.exit(1)
      }
    }(ThreadUtils.sameThread)
  }

```


打开Driver的源码,找到RegisterExecutor部分，如下代码所示：
在Driver里面有一个数据结构executorDataMap，用于存储注册的ExecutorBackend信息，先判断该ExecutorBackend是否在该数据结构中存在，如果不存在则继续往下执行，然后将ExecutorBackend的信息存于各种数据结构中，接下来就是调用通知CoarseGrainedExecutorBackend注册ExecutorBackend成功，再调用makeOffers()方法。makeOffers()方法主要是给ExecutorBackend分配资源，并且启动Task任务;Task的内容会在TaskScheduler部分叙述,我们在这里暂且不过多讲述;

```scala
 override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {

      case RegisterExecutor(executorId, executorRef, hostPort, cores, logUrls) =>
        if (executorDataMap.contains(executorId)) {
          context.reply(RegisterExecutorFailed("Duplicate executor ID: " + executorId))
        } else {
          // If the executor's rpc env is not listening for incoming connections, `hostPort`
          // will be null, and the client connection should be used to contact the executor.
          val executorAddress = if (executorRef.address != null) {
              executorRef.address
            } else {
              context.senderAddress
            }
          logInfo(s"Registered executor $executorRef ($executorAddress) with ID $executorId")
          addressToExecutorId(executorAddress) = executorId
          totalCoreCount.addAndGet(cores)
          totalRegisteredExecutors.addAndGet(1)
          val data = new ExecutorData(executorRef, executorRef.address, executorAddress.host,
            cores, cores, logUrls)
          // This must be synchronized because variables mutated
          // in this block are read when requesting executors
          CoarseGrainedSchedulerBackend.this.synchronized {
            executorDataMap.put(executorId, data)
            if (numPendingExecutors > 0) {
              numPendingExecutors -= 1
              logDebug(s"Decremented number of pending executors ($numPendingExecutors left)")
            }
          }
          // Note: some tests expect the reply to come after we put the executor in the map
          context.reply(RegisteredExecutor(executorAddress.host))
          listenerBus.post(
            SparkListenerExecutorAdded(System.currentTimeMillis(), executorId, data))
          makeOffers()
        }
```
```scala
    // Make fake resource offers on all executors
    private def makeOffers() {
      // Filter out executors under killing
      val activeExecutors = executorDataMap.filterKeys(executorIsAlive)
      val workOffers = activeExecutors.map { case (id, executorData) =>
        new WorkerOffer(id, executorData.executorHost, executorData.freeCores)
      }.toSeq
      launchTasks(scheduler.resourceOffers(workOffers))
    }
```

如下图所示，CoarseGrainedExecutorBackend在接到RegisteredExecutor消息后立即实例化了一个executor对象;

```scala
  override def receive: PartialFunction[Any, Unit] = {
    case RegisteredExecutor(hostname) =>
      logInfo("Successfully registered with driver")
      executor = new Executor(executorId, hostname, env, userClassPath, isLocal = false)
```  

#### 需要注意的是,我们现在主要说的是spark的StandAlone模式;CoarseGrainedExecutorBackend进程的产生和Executor对象的实例化都阐述完毕，最后放出这篇的分析图：
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fwa5cpgiv5j31kw0eyte9.jpg)
