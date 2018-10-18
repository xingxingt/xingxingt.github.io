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



`DAGScheduler`将TaskSet提交给`TaskScheduler`,那么就先看下`submitTasks()`,打开`TaskScheduler`的实现类`TaskSchedulerImpl`,
在这个方法里面，先生成了一个`TaskManager`对象来封装taskSet,然后判断当前stage中是否只正常运行一个taskSet，以及taskManager是否是僵尸进程；
随后将生成的TaskManager放入到`schedulableBuilder`调度策略中，做完以上工作后开始想backend申请资源`backend.reviveOffers()`;

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

我们进入`CoarseGrainedSchedulerBackend`的`reviveOffers`方法，可以看到在方法里面`driverEndpoint`向自己发送了一个`ReviveOffers`消息，而
这个`driverEndpoint`我们前面也讲过，就是当前应用程序的Driver;
```scala
  override def reviveOffers() {
    driverEndpoint.send(ReviveOffers)
  }
```

进入`driverEndpoint`的`reviveOffers`方法，最终调用的是`makeOffers()`方法,

```scala
   case ReviveOffers =>
     makeOffers()
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


