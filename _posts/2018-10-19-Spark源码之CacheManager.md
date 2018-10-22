---
layout:     post
title:      Spark源码之CacheManager
subtitle:   Spark源码之CacheManager介绍
date:       2018-10-17
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>Spark源码之CacheManager篇
>

**CacheManager介绍**  
1.CacheManager管理spark的缓存，而缓存可以基于内存的缓存，也可以是基于磁盘的缓存；  
2.CacheManager需要通过BlockManager来操作数据；  
3.当Task运行的时候会调用RDD的comput方法进行计算，而compute方法会调用iterator方法；  

**CacheManager源码解析**  
既然要说CacheManager，那么就从RDD获取数据开始入手,例如MapPartitionsRDD,在RDD的compute方法中通过` firstParent[T].iterator(split, context)`
获取父RDD的数据;
```scala
private[spark] class MapPartitionsRDD[U: ClassTag, T: ClassTag](
    prev: RDD[T],
    f: (TaskContext, Int, Iterator[T]) => Iterator[U],  // (TaskContext, partition index, iterator)
    preservesPartitioning: Boolean = false)
  extends RDD[U](prev) {

  override val partitioner = if (preservesPartitioning) firstParent[T].partitioner else None

  override def getPartitions: Array[Partition] = firstParent[T].partitions

  override def compute(split: Partition, context: TaskContext): Iterator[U] =
    f(context, split.index, firstParent[T].iterator(split, context))
}
```

我们再来看下RDD的iterator是如何获取数据的,在这个方法中可以看到，首先判断该RDD的数据是否进行了缓存，如果有缓存则使用cacheManager获取数据，
如果没有缓存则调用computeOrReadCheckpoint方法获取缓存;
```scala
  /**
   * Internal method to this RDD; will read from cache if applicable, or otherwise compute it.
   * This should ''not'' be called by users directly, but is available for implementors of custom
   * subclasses of RDD.
   */
  final def iterator(split: Partition, context: TaskContext): Iterator[T] = {
    //todo 判断RDD的数据是否进行了缓存
    if (storageLevel != StorageLevel.NONE) {
     //todo 如果缓存了则使用cacheManager获取数据
      SparkEnv.get.cacheManager.getOrCompute(this, split, context, storageLevel)
    } else {
      //todo 否则冲checkpoint中获取数据或者从新计算rdd的数据
      computeOrReadCheckpoint(split, context)
    }
  }
```

我们先看下如何使用cacheManager获取数据，进入cacheManager的getOrCompute方法,该方法的作用就是从缓存中获取RDD的数据,如果缓存中没有数据则重新计算；  
从这个方法中我们可以看到cacheManager确实是通过blockManager来获取数据的，通过blockManager的get方法，从本地或者远程读取数据；  
如果有数据返回则直接返回用InterruptibleIterator封装的数据集；  
如果blockManager的get方法没有获取到数据那么就要从新计算该RDD的数据，如下源码可见，先给指定的partition加锁，
然后使用`rdd.computeOrReadCheckpoint(partition, context)`从新计算RDD的数据，计算之后再将计算的结果数据存入缓存中，之后便返回RDD的数据;


```scala
  /** Gets or computes an RDD partition. Used by RDD.iterator() when an RDD is cached. */
  //todo 从缓存中获取RDD的数据,如果缓存中没有数据则重新计算
  def getOrCompute[T](
      rdd: RDD[T],
      partition: Partition,
      context: TaskContext,
      storageLevel: StorageLevel): Iterator[T] = {

    val key = RDDBlockId(rdd.id, partition.index)
    logDebug(s"Looking for partition $key")
    //todo cacheManager利用blockManager获取数据
    blockManager.get(key) match {
      case Some(blockResult) =>
        // Partition is already materialized, so just return its values
        //todo 如果有数据则直接返回
        val existingMetrics = context.taskMetrics
          .getInputMetricsForReadMethod(blockResult.readMethod)
        existingMetrics.incBytesRead(blockResult.bytes)

        val iter = blockResult.data.asInstanceOf[Iterator[T]]
        new InterruptibleIterator[T](context, iter) {
          override def next(): T = {
            existingMetrics.incRecordsRead(1)
            delegate.next()
          }
        }
      case None =>
        //todo 如果没有数据则从新计算，并将计算后的数据存入缓存中
        // Acquire a lock for loading this partition
        // If another thread already holds the lock, wait for it to finish return its results
        //todo 给指定的分区加锁
        val storedValues = acquireLockForPartition[T](key)
        if (storedValues.isDefined) {
          return new InterruptibleIterator[T](context, storedValues.get)
        }

        // Otherwise, we have to load the partition ourselves
        try {
          logInfo(s"Partition $key not found, computing it")
          //todo  从新计算rdd的数据
          val computedValues = rdd.computeOrReadCheckpoint(partition, context)

          // If the task is running locally, do not persist the result
          //todo 如果任务运行在本地，则直接返回结果数据不进行持久化
          if (context.isRunningLocally) {
            return computedValues
          }

          // Otherwise, cache the values and keep track of any updates in block statuses
          //todo 将结果数据放入到缓存中
          val updatedBlocks = new ArrayBuffer[(BlockId, BlockStatus)]
          val cachedValues = putInBlockManager(key, computedValues, storageLevel, updatedBlocks)
          val metrics = context.taskMetrics
          val lastUpdatedBlocks = metrics.updatedBlocks.getOrElse(Seq[(BlockId, BlockStatus)]())
          metrics.updatedBlocks = Some(lastUpdatedBlocks ++ updatedBlocks.toSeq)
          new InterruptibleIterator(context, cachedValues)

        } finally {
          loading.synchronized {
            loading.remove(key)
            loading.notifyAll()
          }
        }
    }
  
```

刚才在CacheManager的getOrCompute方法中调用了`rdd.computeOrReadCheckpoint(partition, context)`来获取RDD的数据，我们现在来看下是怎么回事；  
如下源代码所示，在computeOrReadCheckpoint方法中，会先判断spark应用程序是否启用了checkPoint机制，如果启用了则直接去获取数据,如果没有启用checkPoint
机制则需要重新计算,其实这里的重新计算就是调用父类的iterator方法获取数据;

```scala
  /**
   * Compute an RDD partition or read it from a checkpoint if the RDD is checkpointing.
   */
  private[spark] def computeOrReadCheckpoint(split: Partition, context: TaskContext): Iterator[T] =
  {
    //todo 判断数据是否checkPoint
    if (isCheckpointedAndMaterialized) {
      firstParent[T].iterator(split, context)
    } else {
      //todo 如果没有checkPoint则从新进行RDD的comput
      compute(split, context)
    }
  }
  
  override def compute(partition: Partition, context: TaskContext): Iterator[T] = {
    partition.asInstanceOf[CoalescedRDDPartition].parents.iterator.flatMap { parentPartition =>
      firstParent[T].iterator(parentPartition, context)
    }
  }  
```

再说下CacheManager是如何向缓存中写入数据的:  
如下源码所示,在putInBlockManager方法中，先获取RDD的store级别，如果不使用memory存储，则直接使用blockManager.putIterator进行数据存储,
这个方法最终调用的是 blockManager.doPut()存储数据，在[Spark源码之BlockManager](https://github.com/xingxingt/xingxingt.github.io/blob/master/_posts/2018-10-18-Spark%E6%BA%90%E7%A0%81%E4%B9%8BBlockManager.md)这篇中已经讲述过了，
如果使用memory存储数据，则先用blockManager.memoryStore.unrollSafely()获取一块连续内存空间,如果有足够的空间，则直接存储,如果没有足够的空间则
判断是否可以存储在磁盘上，如果可以存储在磁盘上则修改它的store级别为diskOnlyLevel，然后再调用一次putInBlockManager方法将数据存入Disk中;否则直接返回
数据不进行存储;

```scala
  private def putInBlockManager[T](
      key: BlockId,
      values: Iterator[T],
      level: StorageLevel,
      updatedBlocks: ArrayBuffer[(BlockId, BlockStatus)],
      effectiveStorageLevel: Option[StorageLevel] = None): Iterator[T] = {

    //todo 获取store级别
    val putLevel = effectiveStorageLevel.getOrElse(level)
    if (!putLevel.useMemory) {
      /*
       * This RDD is not to be cached in memory, so we can just pass the computed values as an
       * iterator directly to the BlockManager rather than first fully unrolling it in memory.
       */
      //todo 不使用memory的方式存储
      updatedBlocks ++=
        blockManager.putIterator(key, values, level, tellMaster = true, effectiveStorageLevel)
      blockManager.get(key) match {
        case Some(v) => v.data.asInstanceOf[Iterator[T]]
        case None =>
          logInfo(s"Failure to store $key")
          throw new BlockException(key, s"Block manager failed to return cached value for $key!")
      }
    } else {
      /*
       * This RDD is to be cached in memory. In this case we cannot pass the computed values
       * to the BlockManager as an iterator and expect to read it back later. This is because
       * we may end up dropping a partition from memory store before getting it back.
       *
       * In addition, we must be careful to not unroll the entire partition in memory at once.
       * Otherwise, we may cause an OOM exception if the JVM does not have enough space for this
       * single partition. Instead, we unroll the values cautiously, potentially aborting and
       * dropping the partition to disk if applicable.
       */
      //todo 采用memory的形式存储
      blockManager.memoryStore.unrollSafely(key, values, updatedBlocks) match {
        case Left(arr) =>
          // We have successfully unrolled the entire partition, so cache it in memory
          updatedBlocks ++=
            blockManager.putArray(key, arr, level, tellMaster = true, effectiveStorageLevel)
          arr.iterator.asInstanceOf[Iterator[T]]
        case Right(it) =>
          // There is not enough space to cache this partition in memory
          val returnValues = it.asInstanceOf[Iterator[T]]
          if (putLevel.useDisk) {
            logWarning(s"Persisting partition $key to disk instead.")
            val diskOnlyLevel = StorageLevel(useDisk = true, useMemory = false,
              useOffHeap = false, deserialized = false, putLevel.replication)
            putInBlockManager[T](key, returnValues, level, updatedBlocks, Some(diskOnlyLevel))
          } else {
            returnValues
          }
      }
    }
  }
```


