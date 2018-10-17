---
layout:     post
title:      Spark源码之Worker
subtitle:   Spark源码之Worker介绍
date:       2018-10-12
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>Spark源码之Worker介绍篇
> 

### Worker介绍
    Worker作为工作节点,一般Driver以及Executor都会在这Worker上分布;
    
### Worker代码概览   
1. Worker继承了ThreadSafeRpcEndpoint,所以本身就是一个消息循环体,可以直接跟其他组件进行通信；
2. 内部封装一堆数据结构，用于记录存储Driver,Executor，Application等信息；
3. Worker内部对自身的资源维护；
4. 与其他组件通信的通信结构；
5. 与Master之间的协助等
见下面源码:

```scala
//todo 继承ThreadSafeRpcEndpoint消息循环体
private[deploy] class Worker(
    override val rpcEnv: RpcEnv,
    webUiPort: Int,
    cores: Int,
    memory: Int,
    masterRpcAddresses: Array[RpcAddress],
    systemName: String,
    endpointName: String,
    workDirPath: String = null,
    val conf: SparkConf,
    val securityMgr: SecurityManager)
  extends ThreadSafeRpcEndpoint with Logging {


  //TODO 用户存储数据结构
  var workDir: File = null
  val finishedExecutors = new LinkedHashMap[String, ExecutorRunner]
  val drivers = new HashMap[String, DriverRunner]
  val executors = new HashMap[String, ExecutorRunner]
  val finishedDrivers = new LinkedHashMap[String, DriverRunner]
  val appDirectories = new HashMap[String, Seq[String]]
  val finishedApps = new HashSet[String]


  //TODO 内部的资源维护
  var coresUsed = 0
  var memoryUsed = 0

  def coresFree: Int = cores - coresUsed
  def memoryFree: Int = memory - memoryUsed
  
  //通信结构
  override def receive: PartialFunction[Any, Unit] = synchronized(......)
  override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {......}
  
  //与Master之间协作
  private def changeMaster(masterRef: RpcEndpointRef, uiUrl: String) {......}
  private def tryRegisterAllMasters(): Array[JFuture[_]] = {......}
  private def reregisterWithMaster(): Unit = {......}
  ......
  
```


### Worker内部分析
还会从Woker的初始化和启动开始，如下代码所示:

```scala
  //在Worker的伴生对象中
  def startRpcEnvAndEndpoint(
      host: String,
      port: Int,
      webUiPort: Int,
      cores: Int,
      memory: Int,
      masterUrls: Array[String],
      workDir: String,
      workerNumber: Option[Int] = None,
      conf: SparkConf = new SparkConf): RpcEnv = {

    // The LocalSparkCluster runs multiple local sparkWorkerX RPC Environments
    val systemName = SYSTEM_NAME + workerNumber.map(_.toString).getOrElse("")
    val securityMgr = new SecurityManager(conf)
    val rpcEnv = RpcEnv.create(systemName, host, port, conf, securityMgr)
    val masterAddresses = masterUrls.map(RpcAddress.fromSparkURL(_))
    rpcEnv.setupEndpoint(ENDPOINT_NAME, new Worker(rpcEnv, webUiPort, cores, memory,
      masterAddresses, systemName, ENDPOINT_NAME, workDir, conf, securityMgr))
    rpcEnv
  }
```

再看Worker的启动onStart()方法,因为Worker是主动向Master注册的，所以在WorKer启动的方法内就直接向Master注册；
 
```scala
  override def onStart() {
    assert(!registered)
    logInfo("Starting Spark worker %s:%d with %d cores, %s RAM".format(
      host, port, cores, Utils.megabytesToString(memory)))
    logInfo(s"Running Spark version ${org.apache.spark.SPARK_VERSION}")
    logInfo("Spark home: " + sparkHome)
    createWorkDir()
    shuffleService.startIfEnabled()
    webUi = new WorkerWebUI(this, workDir, webUiPort)
    webUi.bind()
    //TODO 向master注册
    registerWithMaster()

    metricsSystem.registerSource(workerSource)
    metricsSystem.start()
    // Attach the worker metrics servlet handler to the web ui after the metrics system is started.
    metricsSystem.getServletHandlers.foreach(webUi.attachHandler)
  }
```

进入registerWithMaster() 方法，

```scala
  private def registerWithMaster() {
    // onDisconnected may be triggered multiple times, so don't attempt registration
    // if there are outstanding registration attempts scheduled.
    registrationRetryTimer match {
      case None =>
        registered = false
        registerMasterFutures = tryRegisterAllMasters()
        connectionAttemptCount = 0
        registrationRetryTimer = Some(forwordMessageScheduler.scheduleAtFixedRate(
          new Runnable {
            override def run(): Unit = Utils.tryLogNonFatalError {
              Option(self).foreach(_.send(ReregisterWithMaster))
            }
          },
          INITIAL_REGISTRATION_RETRY_INTERVAL_SECONDS,
          INITIAL_REGISTRATION_RETRY_INTERVAL_SECONDS,
          TimeUnit.SECONDS))
      case Some(_) =>
        logInfo("Not spawning another attempt to register with the master, since there is an" +
          " attempt scheduled already.")
    }
  }
```
继续往下走,进入tryRegisterAllMasters()，为什么会有tryRegisterAllMasters()方法呢？因为如何Master是HA的情况下就会出现多个Master，所以Worker要将它的信息注册给每个Master,在下面的代码中可见，在Worker里用了一个线程池registerMasterThreadPool生成一条线程去与Master通信；

```scala
  private def tryRegisterAllMasters(): Array[JFuture[_]] = {
    masterRpcAddresses.map { masterAddress =>
      registerMasterThreadPool.submit(new Runnable {
        override def run(): Unit = {
          try {
            logInfo("Connecting to master " + masterAddress + "...")
            val masterEndpoint =
              rpcEnv.setupEndpointRef(Master.SYSTEM_NAME, masterAddress, Master.ENDPOINT_NAME)
            registerWithMaster(masterEndpoint)
          } catch {
            case ie: InterruptedException => // Cancelled
            case NonFatal(e) => logWarning(s"Failed to connect to master $masterAddress", e)
          }
        }
      })
    }
  }
```

继续进入registerWithMaster()方法，在这个方法内你会看到具体想Master发送的注册请求,以及对请求的响应状态的处理;
Worker向Master注册，在[Spark源码之Master中已经详细叙述过]，这里就不再累赘；

```scala
  private def registerWithMaster(masterEndpoint: RpcEndpointRef): Unit = {
    masterEndpoint.ask[RegisterWorkerResponse](RegisterWorker(
      workerId, host, port, self, cores, memory, webUi.boundPort, publicAddress))
      .onComplete {
        // This is a very fast action so we can use "ThreadUtils.sameThread"
        case Success(msg) =>
          Utils.tryLogNonFatalError {
            handleRegisterResponse(msg)
          }
        case Failure(e) =>
          logError(s"Cannot register with master: ${masterEndpoint.address}", e)
          System.exit(1)
      }(ThreadUtils.sameThread)
  }
```
进入handleRegisterResponse()方法，看下具体是如何处理这些Worker向Master注册后的事件；
这个方法里处理的事件：
1. Worker向Master注册成功,在worker注册成功后会调用changeMaster(masterRef, masterWebUiUrl)将当前的Master
   信息保存在Woker内部的数据结构中；
2. Worker向Master注册失败;
3. Master出现StandBy的情况;

```scala
  private def handleRegisterResponse(msg: RegisterWorkerResponse): Unit = synchronized {
    msg match {
      case RegisteredWorker(masterRef, masterWebUiUrl) =>
        logInfo("Successfully registered with master " + masterRef.address.toSparkURL)
        registered = true
        changeMaster(masterRef, masterWebUiUrl)
        forwordMessageScheduler.scheduleAtFixedRate(new Runnable {
          override def run(): Unit = Utils.tryLogNonFatalError {
            self.send(SendHeartbeat)
          }
        }, 0, HEARTBEAT_MILLIS, TimeUnit.MILLISECONDS)
        if (CLEANUP_ENABLED) {
          logInfo(
            s"Worker cleanup enabled; old application directories will be deleted in: $workDir")
          forwordMessageScheduler.scheduleAtFixedRate(new Runnable {
            override def run(): Unit = Utils.tryLogNonFatalError {
              self.send(WorkDirCleanup)
            }
          }, CLEANUP_INTERVAL_MILLIS, CLEANUP_INTERVAL_MILLIS, TimeUnit.MILLISECONDS)
        }

      case RegisterWorkerFailed(message) =>
        if (!registered) {
          logError("Worker registration failed: " + message)
          System.exit(1)
        }

      case MasterInStandby =>
        // Ignore. Master not yet ready.
    }
  }
```
```scala
  private def changeMaster(masterRef: RpcEndpointRef, uiUrl: String) {
    // activeMasterUrl it's a valid Spark url since we receive it from master.
    activeMasterUrl = masterRef.address.toSparkURL
    activeMasterWebUiUrl = uiUrl
    master = Some(masterRef)
    connected = true
    // Cancel any outstanding re-registration attempts because we found a new master
    cancelLastRegistrationRetry()
  }
```

### Woker的其他工作
在Worker还维护着Driver和Executor的变化，如下代码所示：
 
 ```scala
   private[worker] def handleDriverStateChanged(driverStateChanged: DriverStateChanged): Unit = {......}
   private[worker] def handleExecutorStateChanged(executorStateChanged: ExecutorStateChanged): Unit = {......}
   ......
   
 ```
 
####  Worker的源码内容叙述完毕!Worker中的Driver和Executor分析流程部分可参看下图:
![](https://ws3.sinaimg.cn/large/006tNbRwly1fw5ft1y35mj31im0oyn2v.jpg)
