---
layout:     post
title:      spark Shuffle
subtitle:   
date:       2018-08-28
author:     XINGXING
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Spark
---

>
>spark Shuffle篇
> 

### 为什么需要shuffle:
    需要shuffle的关键性原因是某种具有共同特征的数据需要汇聚到一个计算节点上进行计算。

### shuffle可能面临的问题有哪些:
    1,数据量特别大
    2,数据如何分类，即如何partition，hash，sort，钨丝计划
    3,负载均衡；（数据倾斜
    4,网络传输效率，需要在压缩和解压缩之间做出权衡，序列化和反序列化也要考虑
    说明：具体的Task进行计算的时候尽一切最大可能使得数据具备Process Locality的特性；退而求其次可以增加数据分片，减少每个Task处理的数据量。

### Hash Shuffle:
    1,key不能是array；
    2,Hash Shuffle不需要排序，此时从理论上讲就节省了Hadoop MapReduce中进行Shuffle,需要排序时候的时间浪费,因为实际生产环境中有大量的数据不需要排序的shuffle类型；
    3,每个shuffleMapTask（除了最后一个stage中的任务是resultTask之外前面的所有都是shuffleMapTask）会根据key的哈希值计算出当前key需要写入的partition，然后把决定后的结果写入单独的文件，此时导致每个Task产生R（值下一个stage的并行度）个文件，
      如果当前的stage中有M个shuffleMapTask，则会产生M*R个文件！

#### 思考:       
    Q:不需要排序的hash shuffle是否一定比需要排序的sorted shuffle速度快？
    A:不一定！如果数据规模比较小的情况下，hash shuffle会比sorted shuffle速度快很多，但是如果数据量大的情况下，此时sorted shuffle一般都会比hash Shuffle快很多！
    
#### 注意：
    shuffle操作绝大多数情况下是需要通过网络传输的，如果map和reducer在同一台机器上，此时只需要读取本地磁盘即可！

### hash shuffle的缺陷:
    第一：shuffle前会产生海量的小文件在磁盘上，此时会产生大量耗时低效的IO操作；
    第二：内存不够用，由于内存中需要保存大量海量文件操作句柄和临时缓存信息，如果数据处理规模比较庞大的话，内存不可承受。出现oom现象。
![](https://ws3.sinaimg.cn/large/0069RVTdgy1fuyfwlkp15j31kw0npdks.jpg)

### hash shuffle的改善:
    为了改善以上问题（同时打开过多文件导致Writer Handler（每个大约50k）内存使用过大以及产生过度文件导致大量的随机读写带来的效率低下的磁盘IO操作），
    spark后来推出了consalidate机制，来把小文件合并。此时shuffle时的文件数量为core*R（reduce的个数），对于shuffleMapTask的数量明显多于同时
    可用的并行cores的数量的情况下，shuffle产生的文件会大幅度减少，提高性能，会 极大的降低OOM的可能。

### sort-based shuffle的诞生:
    1，Shuffle一般包含两个任务阶段，
       第一部分：产生shuffle数据的阶段（map阶段 补充：需要实现shuffleManager中的getWriter方法来写数据，数据    可以通过blockmanager写到memory，disk，tachyon等，
       例如像非常快的shuffle，此时可以考虑吧数据写到内存中，但是内存不稳定，建议采用Memory_and_disk（如果内存放不下就会存入磁盘中）方式）；
       第二部分:使用shuffle数据的阶段（reduce阶段 补充：需要实现shuffleManager中的getReader方法，reader会向driver去获取上一个shuffle所产生的数据）；
    2，spark的job会分为很多个stage：
       如果只有一个stage，则这个job就相当于只有一个mapper阶段，当然不会产生shuffle，适合于简单的ETL；
       如果不止一个stage，则最后一个stage就是最终的Reducer，最左侧的第一个stage就仅仅是整个job的mapper，
       中间任意一个stage是其父stage的reducer并且还是其子stage的mapper；
    3，Spark Shuffle 在最开始的时候只支持 Hash-based shuffle；默认mapper阶段会为Reducer阶段的每一个task单独
       创建一个文件来保存该Task中要使用的数据,但是在一些情况下（例如数据量非常大的情况）会造成大量文件（M*R个文件，其中M代表Mapper中所有的并行任务数量，
       R代表Reducer中所有的并行任务数据量）的随机磁盘IO操作且会产生大量的memory消耗，容易造成OOM，因为第一：不能处理大规模的数据，第二：spark不能够运行在大规模的分布式集群上，
       所以后来加入了shuffle consolidate机制来讲shuffle时候产生的文件数量减少到C*R个，但是此时如果reducer端的并行数据分片过多的话则C*R可能已经过大，依旧难逃文件打开过多的厄运！
       spark在引入sort-based shuffle（1.1版本之前）以前比较适用于中小规模的大数据处理！
    4，为了让spark在更大规模的集群上更高性能处理更大规模的数据，于是就引入sort-based shuffle，从此以后（1.1版本之后）spark可以胜任任意规模（PB以及PB以上）的大数据处理！
       尤其是随着钨丝计算的引入和优化，把spark更快速的更大规模的集群处理更海量的数据的能力推向了一个新的巅峰！

### spark1.6版本之后至少三种类型shuffle:
![](https://ws2.sinaimg.cn/large/0069RVTdgy1fuyg7hchqvj30yq05ht9d.jpg)

### Sort-based shuffle特性:
    1,Sort-based shuffle会产生2M（M代表Mapper阶段中并行的partition的总数量，其实就是Mapper端shuffleMapTask的总数量）个shuffle临时文件！
    2,sort-based shuffle不会为每个reducer中的task生成一个单独的文件，相反sort-based shuffle会把mapper中每个shuffleMapTask所有的输出数据Data只写到一个文件中。
      因为每个shuffleMapTask中的数据会被分类，所以sort-based shuffle使用了index文件存储具体shuffleMapTask输出数据在同一个Data文件中是如何分类的信息！所以说基于sort-based
      的shuffle会在Mapper中的每个ShuffleMapTask中产生两个文件，Data文件和index文件，其中data文件用于存储当前task的shuffle输出，而index文件中则存储Data文件中的数据通过
      partitioner的分类信息，此时下一个阶段的stage中的task就是根据index文件获取自己所要抓取的上一个stage中的shuffleMapTask所产生的数据;

###  Shuffle变化历史，shuffle产生的临时文件的数量变化趋势：
    Basic hash shuffle：          M*R
    consalidate方式的hash shuffle：C*R
    Sort-based shuffle:           2M

    


	
