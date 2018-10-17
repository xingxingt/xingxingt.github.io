---
layout:     post
title:      Spark源码之Master
subtitle:   Spark源码之Master介绍
date:       2018-10-11
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>Spark源码之Master介绍篇
> 

### Master介绍
Master作为资源管理和分配的组件，所以今天我们重点来看Spark Core中的Master如何实现资源的注册，状态的维护以及调度分配;
    
### Master内部代码概览
1,Master继承了ThreadSafeRpcEndpoint所以它本身也就是一个消息循环体，可以接受其他组件发送的消息并处理;  
2,在Master内部维护了一大堆数据结构，用于存放资源组件的元数据信息,例如注册的worker,application,Driver等；  
3,处理Master所管理的一些组件；  
4,资源调度;   参看如下源码：  

```scala
  //todo 在Master中维护的数据结构
  val workers = new HashSet[WorkerInfo]
  val idToApp = new HashMap[String, ApplicationInfo]
  val waitingApps = new ArrayBuffer[ApplicationInfo]
  val apps = new HashSet[ApplicationInfo]

  private val idToWorker = new HashMap[String, WorkerInfo]
  private val addressToWorker = new HashMap[RpcAddress, WorkerInfo]

  private val endpointToApp = new HashMap[RpcEndpointRef, ApplicationInfo]
  private val addressToApp = new HashMap[RpcAddress, ApplicationInfo]
  private val completedApps = new ArrayBuffer[ApplicationInfo]
  private var nextAppNumber = 0
  // Using ConcurrentHashMap so that master-rebuild-ui-thread can add a UI after asyncRebuildUI
  private val appIdToUI = new ConcurrentHashMap[String, SparkUI]

  private val drivers = new HashSet[DriverInfo]
  private val completedDrivers = new ArrayBuffer[DriverInfo]
  // Drivers currently spooled for scheduling
  private val waitingDrivers = new ArrayBuffer[DriverInfo]
  private var nextDriverNumber = 0
  
  //todo 消息处理
  override def receive: PartialFunction[Any, Unit] = {......}
  override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {......}
  
  //todo 管理的一些组件
  private def newApplicationId(submitDate: Date): String = {......}
  private def newDriverId(submitDate: Date): String = {......}
  private def createDriver(desc: DriverDescription): DriverInfo = {......}
  private def launchDriver(worker: WorkerInfo, driver: DriverInfo) {......}
  ......
  
  //资源调度分配
  private def schedule(): Unit = {
    if (state != RecoveryState.ALIVE) { return }
    // Drivers take strict precedence over executors
    val shuffledWorkers = Random.shuffle(workers) // Randomization helps balance drivers
    for (worker <- shuffledWorkers if worker.state == WorkerState.ALIVE) {
      for (driver <- waitingDrivers) {
        if (worker.memoryFree >= driver.desc.mem && worker.coresFree >= driver.desc.cores) {
          launchDriver(worker, driver)
          waitingDrivers -= driver
        }
      }
    }
    startExecutorsOnWorkers()
  }  
  
```

### Master对其他组件的注册处理
1,Master接受注册的对象主要是Worker，Driver，Application；而Excutor不会注册给master，Excutor是注册给Driver中的SchedulerBackend的；  
2,再者Worker是再启动后主动向Master注册的，所以如果生产环境中加入新的worker到正在运行的spark集群中，此时不需要重新启动spark集群就能够使用新加入的worker，以提升处理能力；  

我们以Worker注册为例，
Master在接收到worker注册到的请求后，首先会判断一下当前的master是否是standby模式，如果是就不处理；
然后会判断当前Master的内存数据结构idToWorker中是否已经存在该worker，如果有的话就不会重复注册；
Master如果决定接受注册的worker，首先会创建WorkerInfo对象来保存注册当前的worker信息；

```scala
  override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
    case RegisterWorker(
        id, workerHost, workerPort, workerRef, cores, memory, workerUiPort, publicAddress) => {
      logInfo("Registering worker %s:%d with %d cores, %s RAM".format(
        workerHost, workerPort, cores, Utils.megabytesToString(memory)))
      if (state == RecoveryState.STANDBY) {
        context.reply(MasterInStandby)
      } else if (idToWorker.contains(id)) {
        context.reply(RegisterWorkerFailed("Duplicate worker ID"))
      } else {
        val worker = new WorkerInfo(id, workerHost, workerPort, cores, memory,
          workerRef, workerUiPort, publicAddress)
        if (registerWorker(worker)) {
          persistenceEngine.addWorker(worker)
          context.reply(RegisteredWorker(self, masterWebUiUrl))
          schedule()
        } else {
          val workerAddress = worker.endpoint.address
          logWarning("Worker registration failed. Attempted to re-register worker at same " +
            "address: " + workerAddress)
          context.reply(RegisterWorkerFailed("Attempted to re-register worker at same address: "
            + workerAddress))
        }
      }
    }
```

然后在执行registerWorker(worker)方法执行具体的注册流程，如果worker的状态是DEAD的话就直接过滤掉，对于UNKOWN状态的内容调用removeWorker进行清理（也包括清理该worker下的Excutors和Drivers）,其实这里就是检查该worker的状态然后将woker信息存放在Master维护的数据结构中；参看如下代码:

```scala
  private def registerWorker(worker: WorkerInfo): Boolean = {
    // There may be one or more refs to dead workers on this same node (w/ different ID's),
    // remove them.
    workers.filter { w =>
      (w.host == worker.host && w.port == worker.port) && (w.state == WorkerState.DEAD)
    }.foreach { w =>
      workers -= w
    }

    val workerAddress = worker.endpoint.address
    if (addressToWorker.contains(workerAddress)) {
      val oldWorker = addressToWorker(workerAddress)
      if (oldWorker.state == WorkerState.UNKNOWN) {
        // A worker registering from UNKNOWN implies that the worker was restarted during recovery.
        // The old worker must thus be dead, so we will remove it and accept the new worker.
        removeWorker(oldWorker)
      } else {
        logInfo("Attempted to re-register worker at same address: " + workerAddress)
        return false
      }
    }

    workers += worker
    idToWorker(worker.id) = worker
    addressToWorker(workerAddress) = worker
    true
  }
```
    
1,如果registerWoker成功，则将worker信息进行持久化，可以将worker信息持久化在zookeeper中,或者系统文件(fileSystem),用户自定义的文件persistenceEngine中，以便于处理灾难后的数据恢复；   
2,worker信息持久化后遍回复worker完成注册的消息；  
3,接着进行schudule，给worker分配资源，schudule我们单独分析；  
参看下面两段代码：

```scala
  override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
    case RegisterWorker(
        id, workerHost, workerPort, workerRef, cores, memory, workerUiPort, publicAddress) => {
      logInfo("Registering worker %s:%d with %d cores, %s RAM".format(
        workerHost, workerPort, cores, Utils.megabytesToString(memory)))
      if (state == RecoveryState.STANDBY) {
        context.reply(MasterInStandby)
      } else if (idToWorker.contains(id)) {
        context.reply(RegisterWorkerFailed("Duplicate worker ID"))
      } else {
        val worker = new WorkerInfo(id, workerHost, workerPort, cores, memory,
          workerRef, workerUiPort, publicAddress)
        if (registerWorker(worker)) {
          //todo 将worker信息持久化
          persistenceEngine.addWorker(worker)
          context.reply(RegisteredWorker(self, masterWebUiUrl))
          schedule()
        } else {
          val workerAddress = worker.endpoint.address
          logWarning("Worker registration failed. Attempted to re-register worker at same " +
            "address: " + workerAddress)
          context.reply(RegisterWorkerFailed("Attempted to re-register worker at same address: "
            + workerAddress))
        }
      }
    }
```
```scala
  override def onStart(): Unit = {
  
    ......
    //todo 选择持久化信息系统
    val serializer = new JavaSerializer(conf)
    val (persistenceEngine_, leaderElectionAgent_) = RECOVERY_MODE match {
      case "ZOOKEEPER" =>
        logInfo("Persisting recovery state to ZooKeeper")
        val zkFactory =
          new ZooKeeperRecoveryModeFactory(conf, serializer)
        (zkFactory.createPersistenceEngine(), zkFactory.createLeaderElectionAgent(this))
      case "FILESYSTEM" =>
        val fsFactory =
          new FileSystemRecoveryModeFactory(conf, serializer)
        (fsFactory.createPersistenceEngine(), fsFactory.createLeaderElectionAgent(this))
      case "CUSTOM" =>
        val clazz = Utils.classForName(conf.get("spark.deploy.recoveryMode.factory"))
        val factory = clazz.getConstructor(classOf[SparkConf], classOf[Serializer])
          .newInstance(conf, serializer)
          .asInstanceOf[StandaloneRecoveryModeFactory]
        (factory.createPersistenceEngine(), factory.createLeaderElectionAgent(this))
      case _ =>
        (new BlackHolePersistenceEngine(), new MonarchyLeaderAgent(this))
    }
    persistenceEngine = persistenceEngine_
    leaderElectionAgent = leaderElectionAgent_
  }

```

### Master对资源组件状态变化的处理：

如下源码中对Driver状态的处理，先检查Driver的状态，如果Driver出现error,finished,killed,failed状态，则将此Driver移除掉，在处理executor的状态变化时也是先检查executor的状态，然后在进行移除或者资源重分配的操作；详细看下图源码：

```scala
    //对Driver状态的变化处理
    case DriverStateChanged(driverId, state, exception) => {
      state match {
        case DriverState.ERROR | DriverState.FINISHED | DriverState.KILLED | DriverState.FAILED =>
          removeDriver(driverId, state, exception)
        case _ =>
          throw new Exception(s"Received unexpected state update for driver $driverId: $state")
      }
    }
    
    //对Executor状态变化的处理
    case ExecutorStateChanged(appId, execId, state, message, exitStatus) => {
      val execOption = idToApp.get(appId).flatMap(app => app.executors.get(execId))
      execOption match {
        case Some(exec) => {
          val appInfo = idToApp(appId)
          val oldState = exec.state
          exec.state = state

          if (state == ExecutorState.RUNNING) {
            assert(oldState == ExecutorState.LAUNCHING,
              s"executor $execId state transfer from $oldState to RUNNING is illegal")
            appInfo.resetRetryCount()
          }

          exec.application.driver.send(ExecutorUpdated(execId, state, message, exitStatus))

          if (ExecutorState.isFinished(state)) {
            // Remove this executor from the worker and app
            logInfo(s"Removing executor ${exec.fullId} because it is $state")
            // If an application has already finished, preserve its
            // state to display its information properly on the UI
            if (!appInfo.isFinished) {
              appInfo.removeExecutor(exec)
            }
            exec.worker.removeExecutor(exec)

            val normalExit = exitStatus == Some(0)
            // Only retry certain number of times so we don't go into an infinite loop.
            if (!normalExit) {
              if (appInfo.incrementRetryCount() < ApplicationState.MAX_NUM_RETRY) {
                schedule()
              } else {
                val execs = appInfo.executors.values
                if (!execs.exists(_.state == ExecutorState.RUNNING)) {
                  logError(s"Application ${appInfo.desc.name} with ID ${appInfo.id} failed " +
                    s"${appInfo.retryCount} times; removing it")
                  removeApplication(appInfo, ApplicationState.FAILED)
                }
              }
            }
          }
        }
        case None =>
          logWarning(s"Got status update for unknown executor $appId/$execId")
      }
    }    
```


###  资源调度分配
Master中的资源分配尤为重要，所以我们着重查探Master是如何进行资源的调度分配的；
Master中的Schedule()方法就是用于资源分配，Schedule()将可用的资源分配给等待被分配资源的Applications,这个方法随时都要被调用，比如说Application的加入或者可用资源的变化；
再看Schedule具体执行的内容:
1. 先判断master的状态，如果不是alive状态，就什么都不做；
2. 将注册进来的所有worker进行shuffle,随机打乱，以便做到负载均衡；
3. 然后在打乱的worker中过滤掉状态不是alive的worker；
4. 将waitingDrivers中的worker一个一个的在worker上启动；
5. 启动Driver后才能启动executor；

```scala
  /**
   * Schedule the currently available resources among waiting apps. This method will be called
   * every time a new app joins or resource availability changes.
   */
  private def schedule(): Unit = {
    if (state != RecoveryState.ALIVE) { return }
    // Drivers take strict precedence over executors
    val shuffledWorkers = Random.shuffle(workers) // Randomization helps balance drivers
    for (worker <- shuffledWorkers if worker.state == WorkerState.ALIVE) {
      for (driver <- waitingDrivers) {
        if (worker.memoryFree >= driver.desc.mem && worker.coresFree >= driver.desc.cores) {
          launchDriver(worker, driver)
          waitingDrivers -= driver
        }
      }
    }
    startExecutorsOnWorkers()
  }
```
    
接着看下Driver是如何启动的，在launchDriver()中向worker发送一个消息`worker.endpoint.send(LaunchDriver(driver.id, driver.desc))`让woker启动一个driver线程;Driver启动完成将该Driver的状态改为Running状态;

```scala
  private def launchDriver(worker: WorkerInfo, driver: DriverInfo) {
    logInfo("Launching driver " + driver.id + " on worker " + worker.id)
    worker.addDriver(driver)
    driver.worker = Some(worker)
    worker.endpoint.send(LaunchDriver(driver.id, driver.desc))
    driver.state = DriverState.RUNNING
  }
```
```scala
    //worker接收到的LaunchDriver消息
    case LaunchDriver(driverId, driverDesc) => {
      logInfo(s"Asked to launch driver $driverId")
      val driver = new DriverRunner(
        conf,
        driverId,
        workDir,
        sparkHome,
        driverDesc.copy(command = Worker.maybeUpdateSSLSettings(driverDesc.command, conf)),
        self,
        workerUri,
        securityMgr)
      drivers(driverId) = driver
      driver.start()

      coresUsed += driverDesc.cores
      memoryUsed += driverDesc.mem
    }
```


Ok,Driver启动完成后就可以启动Executor了，因为Executor是注册给Driver的，所以要先把Driver启动完毕；
接着我们进入startExecutorsOnWorkers()方法中，此方法的作用就是调度和启动Executor在worker上；  
1,遍历等待分配资源的Application，并且过滤掉所有一些不需要继续分配资源的Application；  
2,过滤掉状态不为alive和资源不足的worker，并根据资源大小对workers进行排序；  
3,调用scheduleExecutorsOnWorkers()方法，指定executor是在哪些workers上启动，并返回一个为每个worker指定cores的数组;  
 
```scala
  /**
   * Schedule and launch executors on workers
   */
  private def startExecutorsOnWorkers(): Unit = {
    // Right now this is a very simple FIFO scheduler. We keep trying to fit in the first app
    // in the queue, then the second app, etc.
    for (app <- waitingApps if app.coresLeft > 0) {
      val coresPerExecutor: Option[Int] = app.desc.coresPerExecutor
      // Filter out workers that don't have enough resources to launch an executor
      val usableWorkers = workers.toArray.filter(_.state == WorkerState.ALIVE)
        .filter(worker => worker.memoryFree >= app.desc.memoryPerExecutorMB &&
          worker.coresFree >= coresPerExecutor.getOrElse(1))
        .sortBy(_.coresFree).reverse
      val assignedCores = scheduleExecutorsOnWorkers(app, usableWorkers, spreadOutApps)

      // Now that we've decided how many cores to allocate on each worker, let's allocate them
      for (pos <- 0 until usableWorkers.length if assignedCores(pos) > 0) {
        allocateWorkerResourceToExecutors(
          app, assignedCores(pos), coresPerExecutor, usableWorkers(pos))
      }
    }
  }
```


具体的资源分配还是要看scheduleExecutorsOnWorkers(),进入该方法；  
为应用程序分配Executor有两种方式：  
第一种方式是尽可能在集群的所有节点的worker上分配Executor，这种方式往往会带来潜在的更好，有利于数据本地性，  
第二种是尽量在一个节点的worker上分配Executor，这样的效率可想而知非常低下；  
coresPerExecutor：每个Executor上分配的core;  
minCoresPerExecutor:每个Executor最少分配的core数;  
oneExecutorPerWorker:一个worker是否只启动一个Executor；  
memoryPerExecutor:一个Executor分配到momery大小；  
numUsable：可用的worker；      
assignedCores：指的是在某个worker上已经被分配了多少个cores；  
assignedExecutors指的是已经被分配了多少个Executor；  
coresToAssign：最少为Application分配的core数；  

```scala
  private def scheduleExecutorsOnWorkers(
      app: ApplicationInfo,
      usableWorkers: Array[WorkerInfo],
      spreadOutApps: Boolean): Array[Int] = {
    val coresPerExecutor = app.desc.coresPerExecutor
    val minCoresPerExecutor = coresPerExecutor.getOrElse(1)
    val oneExecutorPerWorker = coresPerExecutor.isEmpty
    val memoryPerExecutor = app.desc.memoryPerExecutorMB
    val numUsable = usableWorkers.length
    val assignedCores = new Array[Int](numUsable) // Number of cores to give to each worker
    val assignedExecutors = new Array[Int](numUsable) // Number of new executors on each worker
    var coresToAssign = math.min(app.coresLeft, usableWorkers.map(_.coresFree).sum)
```
 

canLaunchExecutor()此函数判断该worker上是否可以启动一个Executor；  
首先要判断该worker上是否有充足的资源，usableWorkers(pos)代表一个worker；  
接着判断在这个方法内如果允许在一个worker上启动多个Executor，那么他将总是启动新的Executor，否则，如果有之前启动的Executor，就在这个Executor上不断的增加cores；如下代码所示：

```scala
    /** Return whether the specified worker can launch an executor for this app. */
    def canLaunchExecutor(pos: Int): Boolean = {
      val keepScheduling = coresToAssign >= minCoresPerExecutor
      val enoughCores = usableWorkers(pos).coresFree - assignedCores(pos) >= minCoresPerExecutor

      // If we allow multiple executors per worker, then we can always launch new executors.
      // Otherwise, if there is already an executor on this worker, just give it more cores.
      val launchingNewExecutor = !oneExecutorPerWorker || assignedExecutors(pos) == 0
      if (launchingNewExecutor) {
        val assignedMemory = assignedExecutors(pos) * memoryPerExecutor
        val enoughMemory = usableWorkers(pos).memoryFree - assignedMemory >= memoryPerExecutor
        val underLimit = assignedExecutors.sum + app.executors.size < app.executorLimit
        keepScheduling && enoughCores && enoughMemory && underLimit
      } else {
        // We're adding cores to an existing executor, so no need
        // to check memory and executor limits
        keepScheduling && enoughCores
      }
    }
```

接着看具体的执行方法，  
利用canLaunchExecutor()过滤出numUsable中可用的worker；然后遍历每个worker，为每个worker上的Executor分配core；  
这里有个参数spreadOutApps，如果在默认的情况下spreadOutApps=true它会每次给我们的Executor分配一个core；  
如果在默认的情况下（spreadOutApps=true）它会每次给我们的Executor分配一个core，但是如果spreadOutApps=false它也是每次给我们的Executor分配一个core；  
具体的算法：  
如果是spreadOutApps=false则会不断循环使用当前worker上的这个Executor的所有freeCores；实际上的工作原理：假如有四个worker，如果是spreadOutApps=true，它会在每个worker上启动一个Executor然后先循环一轮，给每个woker上的Executor分配一个core，然后再次循环再给每个executor 分配一个core，依次循环分配;

```scala
   var freeWorkers = (0 until numUsable).filter(canLaunchExecutor)
    while (freeWorkers.nonEmpty) {
      //todo 遍历每个worker
      freeWorkers.foreach { pos =>
        var keepScheduling = true
        while (keepScheduling && canLaunchExecutor(pos)) {
          coresToAssign -= minCoresPerExecutor
          //todo 记录该worker上启动Executor分配的core数
          assignedCores(pos) += minCoresPerExecutor


          // If we are launching one executor per worker, then every iteration assigns 1 core
          // to the executor. Otherwise, every iteration assigns cores to a new executor.
          //todo 如果在每个worker上启动一个Executor，那么就给每个executor分配一个core
          if (oneExecutorPerWorker) {
            assignedExecutors(pos) = 1
          } else {
            //todo 否则就给当前的executor不断的加一个core
            assignedExecutors(pos) += 1
          }

          // Spreading out an application means spreading out its executors across as
          // many workers as possible. If we are not spreading out, then we should keep
          // scheduling executors on this worker until we use all of its resources.
          // Otherwise, just move on to the next worker.
          if (spreadOutApps) {
            keepScheduling = false
          }
        }
      }
```

从下面的代码其实我们可以看出：  
如果是每个worker下面只能够为当前的应用程序分配一个Executor的话，每次为这个executor只分配一个core！  
在应用程序提交时指定每个Executor分配多个cores，其实在实际分配的时候没有什么用，因为在为每个Executor分配core的时候是一个一个的分配的，但是指定的cores在前面做条件过滤时有用，只用在满足应用程序指定的资源条件情况下才能进行分配；

```scala
    // If we are launching one executor per worker, then every iteration assigns 1 core
    // to the executor. Otherwise, every iteration assigns cores to a new executor.
    //todo 如果在每个worker上启动一个Executor，那么就给每个executor分配一个core
    if (oneExecutorPerWorker) {
      assignedExecutors(pos) = 1
    } else {
      //todo 否则就给当前的executor不断的加一个core
      assignedExecutors(pos) += 1
    }
```

上面scheduleExecutorsOnWorkers()方法中已经讲述了Executor在worker上分配的原则，并且会返回一个每个worker上分配资源的数组assignedCores，接下来就可以根据这个资源分配的数组去为Executor分配资源，并且启动Executor；

 ```scala
   private def allocateWorkerResourceToExecutors(
      app: ApplicationInfo,
      assignedCores: Int,
      coresPerExecutor: Option[Int],
      worker: WorkerInfo): Unit = {
    // If the number of cores per executor is specified, we divide the cores assigned
    // to this worker evenly among the executors with no remainder.
    // Otherwise, we launch a single executor that grabs all the assignedCores on this worker.
    val numExecutors = coresPerExecutor.map { assignedCores / _ }.getOrElse(1)
    val coresToAssign = coresPerExecutor.getOrElse(assignedCores)
    for (i <- 1 to numExecutors) {
      val exec = app.addExecutor(worker, coresToAssign)
      launchExecutor(worker, exec)
      app.state = ApplicationState.RUNNING
    }
  }
 ```

接下来看下launchExecutor()方法，  
1,`worker.endpoint.send(LaunchExecutor(masterUrl.....`向worker发送一个LaunchExecutor消息，在worker中启动一个ExecutorRunner线程；  
2,`exec.application.driver.send(ExecutorAdded(exec.... `向driver中添加一个Executor；  

```scala
  private def launchExecutor(worker: WorkerInfo, exec: ExecutorDesc): Unit = {
    logInfo("Launching executor " + exec.fullId + " on worker " + worker.id)
    worker.addExecutor(exec)
    worker.endpoint.send(LaunchExecutor(masterUrl,
      exec.application.id, exec.id, exec.application.desc, exec.cores, exec.memory))
    exec.application.driver.send(
      ExecutorAdded(exec.id, worker.id, worker.hostPort, exec.cores, exec.memory))
  }

```

#### 至此Master中的重要内容已经叙述完毕，最后祭个图:

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fw4mdxdd7jj31kw0k9mye.jpg)
