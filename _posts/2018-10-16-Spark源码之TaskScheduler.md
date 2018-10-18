---
layout:     post
title:      Spark源码之TaskScheduler
subtitle:   Spark源码之TaskScheduler介绍
date:       2018-10-16
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>Spark源码之TaskScheduler介绍篇
> 


前面`DAGScheduler`将stage划分好之后,又将生成的TaskSet提交给`TaskScheduler`,那么本章节就要叙述下`TaskScheduler`如何启动Task的；



DAGScheduler将TaskSet提交给TaskScheduler,那么就先看下`submitTasks()`,打开TaskScheduler的实现类TaskSchedulerImpl,在这个方法里面，先生成了一个TaskManager对象来封装taskSet,然后判断当前stage中是否只正常运行一个taskSet，以及taskManager是否是僵尸进程；随后将生成的TaskManager放入到schedulableBuilder调度策略中，做完以上工作后开始想backend申请资源`backend.reviveOffers()`;

```scala
  override def submitTasks(taskSet: TaskSet) {
    val tasks = taskSet.tasks
    logInfo("Adding task set " + taskSet.id + " with " + tasks.length + " tasks")
    this.synchronized {
      //创建一个TaskManager maxTaskFailures:最大失败重试次数 默认4次
      val manager = createTaskSetManager(taskSet, maxTaskFailures)
      val stage = taskSet.stageId
      val stageTaskSets =
        taskSetsByStageIdAndAttempt.getOrElseUpdate(stage, new HashMap[Int, TaskSetManager])
      stageTaskSets(taskSet.stageAttemptId) = manager
      // 在这里判断当前stage的是否有两个taskSet在运行，因为同一个stage中只能运行一个taskSet
      // 一方面判断当前的TaskSet是否已经在运行了，
      // 另一方面判断当前的taskSetManager是否是僵尸进程
      val conflictingTaskSet = stageTaskSets.exists { case (_, ts) =>
        ts.taskSet != taskSet && !ts.isZombie
      }
      if (conflictingTaskSet) {
        throw new IllegalStateException(s"more than one active taskSet for stage $stage:" +
          s" ${stageTaskSets.toSeq.map{_._2.taskSet.id}.mkString(",")}")
      }
      //将TaskSetManager添加到调度策略中(FIFOSchedulableBuilder/FIARSchedulableBuilder)
      schedulableBuilder.addTaskSetManager(manager, manager.taskSet.properties)

      if (!isLocal && !hasReceivedTask) {
        starvationTimer.scheduleAtFixedRate(new TimerTask() {
          override def run() {
            if (!hasLaunchedTask) {
              logWarning("Initial job has not accepted any resources; " +
                "check your cluster UI to ensure that workers are registered " +
                "and have sufficient resources")
            } else {
              this.cancel()
            }
          }
        }, STARVATION_TIMEOUT_MS, STARVATION_TIMEOUT_MS)
      }
      hasReceivedTask = true
    }
    //在这里开始向backend的发送资源申请的请求，其实是向DriverEndPoint发送的请求
    backend.reviveOffers()
  }
```

我们进入CoarseGrainedSchedulerBackend的reviveOffers方法，可以看到在方法里面driverEndpoint向自己发送了一个ReviveOffers消息，而这个driverEndpoint我们前面也讲过，就是当前应用程序的Driver;

```scala
  override def reviveOffers() {
    driverEndpoint.send(ReviveOffers)
  }
```

进入`driverEndpoint`的`reviveOffers`方法，最终调用的是`makeOffers()`方法,在这个方法里面先过滤出状态为alive的executor，然后将这些activeExecutor封装成WorkerOffer对象，关键点是在最后的`lanchTasks`方法，我们先看下`scheduler.resourceOffers(workOffers)`这个方法的作用；

```scala
   case ReviveOffers =>
     makeOffers()


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

我们再回到TaskSchedulerImpl,查看resourceOffers方法，在方法内部先将可用的executors添加到数据结构中，然后在将可用的executors进行shuffle以便做到负载均衡，为每个executor创建一个task数组用于存放TaskDescription，最后遍历调度策略中的TaskSet,使用就近原则为task分配executor，在这里需要腔调一点的是在`DAGScheduler.submitMissingTasks()`方法中我们是获取了每个task的对应数据的位置，而在本方法中的`taskSet.myLocalityLevels) `是为了获取Task对应数据位置的级别,如下代码所示:

```scala
  def resourceOffers(offers: Seq[WorkerOffer]): Seq[Seq[TaskDescription]] = synchronized {
    // Mark each slave as alive and remember its hostname
    // Also track if new executor is added
    //todo 标记是否有新的executor
    var newExecAvail = false
    //todo 遍历每个executor
    for (o <- offers) {
      //todo 向数据结构中添加executor信息
      executorIdToHost(o.executorId) = o.host
      executorIdToTaskCount.getOrElseUpdate(o.executorId, 0)
      if (!executorsByHost.contains(o.host)) {
        executorsByHost(o.host) = new HashSet[String]()
        executorAdded(o.executorId, o.host)
        newExecAvail = true
      }
      for (rack <- getRackForHost(o.host)) {
        hostsByRack.getOrElseUpdate(rack, new HashSet[String]()) += o.host
      }
    }

    // Randomly shuffle offers to avoid always placing tasks on the same set of workers.
    //todo 将可用的executor进行shuffle打乱,以便做到负载均衡
    val shuffledOffers = Random.shuffle(offers)
    // Build a list of tasks to assign to each worker.
    //todo 为每个executor构建一个task数组
    val tasks = shuffledOffers.map(o => new ArrayBuffer[TaskDescription](o.cores))
    //todo 所有executor可用的core资源
    val availableCpus = shuffledOffers.map(o => o.cores).toArray
    //todo 从调度池中获取TaskSetManagers
    val sortedTaskSets = rootPool.getSortedTaskSetQueue
    for (taskSet <- sortedTaskSets) {
      logDebug("parentName: %s, name: %s, runningTasks: %s".format(
        taskSet.parent.name, taskSet.name, taskSet.runningTasks))
      if (newExecAvail) {
        taskSet.executorAdded()
      }
    }

    // Take each TaskSet in our scheduling order, and then offer it each node in increasing order
    // of locality levels so that it gets a chance to launch local tasks on all of them.
    // NOTE: the preferredLocality order: PROCESS_LOCAL, NODE_LOCAL, NO_PREF, RACK_LOCAL, ANY
    //todo 我们在DAGScheduler.submitMissingTasks()中已经获取了每个Task中数据所在的位置，
    //todo 这是的taskSet.myLocalityLevels只是根据Task数据所在的host来获取它的的数据本地
    //todo 性级别(PROCESS_LOCAL, NODE_LOCAL, NO_PREF, RACK_LOCAL, ANY)
    var launchedTask = false
    //todo  对每一个taskSet，按照就近顺序分配最近的executor来执行task
    for (taskSet <- sortedTaskSets; maxLocality <- taskSet.myLocalityLevels) {
      do {
        //todo 将前面随机打散的WorkOffers计算资源按照就近原则分配给taskSet，用于执行其中的task
        launchedTask = resourceOfferSingleTaskSet(
            taskSet, maxLocality, shuffledOffers, availableCpus, tasks)
      } while (launchedTask)
    }

    if (tasks.size > 0) {
      hasLaunchedTask = true
    }
    return tasks
  }

```

接着进入`resourceOfferSingleTaskSet`方法,遍历所有的executor的索引地址,以便作为tasks的索引，将每个task分配给相应的executor，并填充tasks；
而这个tasks数据结构是调用resourceOfferSingleTaskSet的方法里传进来的，它存储着每个executor内的tasks信息，详细信息见下图源码:

```scala
  private def resourceOfferSingleTaskSet(
      taskSet: TaskSetManager,
      maxLocality: TaskLocality,
      shuffledOffers: Seq[WorkerOffer],
      availableCpus: Array[Int],
      tasks: Seq[ArrayBuffer[TaskDescription]]) : Boolean = {
    var launchedTask = false
    //todo 遍历所有的executor的索引
    for (i <- 0 until shuffledOffers.size) {
      val execId = shuffledOffers(i).executorId
      val host = shuffledOffers(i).host
      //todo 判断该executor可以用的资源是否>=CPUS_PER_TASK(默认为1)
      if (availableCpus(i) >= CPUS_PER_TASK) {
        try {
          //todo 为taskSet中的task分配executor，并将信息存储在tasks中,注意这个tasks是从
          //todo 上面的方法传进来的
          for (task <- taskSet.resourceOffer(execId, host, maxLocality)) {
            //todo 将每个task信息写入下面的数据结构中
            tasks(i) += task
            val tid = task.taskId
            taskIdToTaskSetManager(tid) = taskSet
            taskIdToExecutorId(tid) = execId
            executorIdToTaskCount(execId) += 1
            executorsByHost(host) += execId
            availableCpus(i) -= CPUS_PER_TASK
            assert(availableCpus(i) >= 0)
            launchedTask = true
          }
        } catch {
          case e: TaskNotSerializableException =>
            logError(s"Resource offer failed, task set ${taskSet.name} was not serializable")
            // Do not offer resources for this task, but don't throw an error to allow other
            // task sets to be submitted.
            return launchedTask
        }
      }
    }
    return launchedTask
  }
```


