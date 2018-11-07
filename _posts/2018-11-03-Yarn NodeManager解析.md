---
layout:     post
title:      Yarn NodeManager解析
subtitle:   Yarn NodeManager解析
date:       2018-10-31
author:     XINGXING
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Yarn
---

>
>Yarn NodeManager解析篇
> 

### NodeManager简介
NodeManager是Yarn中单节点的代理，它管理Hadoop集群中单个计算节点,他需要与应用程序的ApplicationMaster和集群管理器RM交互，从ApplicationMaster上接收到相关Container的命令(启动，停止Container)并执行,向RM汇报各个Container的运行状态和节点健康状态，并领取相关的Container的命令执行;  
功能包括与RM保持通信，管理Container的生命周期，监控每个Container的资源使用情况，追踪节点健康状况，管理日志以及不同应用程序用到的附属服务;  

### NodeManager基本职能
NodeManager通过两个RPC协议与RM和各个ApplicationMaster进行通信:   
**1.ResourceTrackerProtocol协议**  
NodeManager通过该RPC协议向RM注册，汇报节点的健康状态以及Container的运行状态，并领取RM下发的命令例如重新初始化Container，清理Container等；在这个协议总NodeManager主动向RM发请求，RM相应NodeManager的请求；该RPC协议的代码如下:  
```protobuf
option java_package = "org.apache.hadoop.yarn.proto";
option java_outer_classname = "ResourceTracker";
option java_generic_services = true;
option java_generate_equals_and_hash = true;
package hadoop.yarn;

import "yarn_server_common_service_protos.proto";

service ResourceTrackerService {
  //NodeManager启动时通过该RPC函数向RM注册,注册信息包括:NM的http端口,NM所在host的RPC端口,该NM可分配的总资源
  rpc registerNodeManager(RegisterNodeManagerRequestProto) returns (RegisterNodeManagerResponseProto);
  //此RPC函数比较重要，NM启动后定期的通过此函数向RM汇报Container的运行信息和节点健康状况，并领取RM的命令，例如：kill掉一些Container
  rpc nodeHeartbeat(NodeHeartbeatRequestProto) returns (NodeHeartbeatResponseProto);
}
```

**2.ContainerManagementProtocol协议**   
应用程序的ApplicationMaster通过该协议与NodeManager通信发起针对Container的命令操作，例如：启动，杀死Container，获取Container的运行状态等;在该协议中ApplicationMaster主动向NodeManager发送请求，NodeManager接收到请求做出相应,该RPC协议的代码如下:  
```protobuf
option java_package = "org.apache.hadoop.yarn.proto";
option java_outer_classname = "ContainerManagementProtocol";
option java_generic_services = true;
option java_generate_equals_and_hash = true;
package hadoop.yarn;

import "yarn_service_protos.proto";

service ContainerManagementProtocolService {
  //AM通过该函数要求NM启动一个Container
  rpc startContainers(StartContainersRequestProto) returns (StartContainersResponseProto);
  //AM通过该函数要求NM杀死一个Container
  rpc stopContainers(StopContainersRequestProto) returns (StopContainersResponseProto);
  //AM通过该函数获取Container的运行状态
  rpc getContainerStatuses(GetContainerStatusesRequestProto) returns (GetContainerStatusesResponseProto);
}

```

### NodeManager内部架构  
NodeManager的内部架构如下图所示:  
![](https://ws2.sinaimg.cn/large/006tNbRwly1fwygnztgncj31g811egpb.jpg)  
1. NodeStatusUpdater: 这个组件是NodeManager与RM通信的唯一通道，包括NM注册之后向RM的注册汇报资源，以及周期性的汇报节点信息和Container的运行状态，同时RM会返回给待清理的Container列表，待清理的应用程序，诊断信息等;  
2. ContainerManager: 该组件是NodeManager最核心的组件之一，它内部有很多子组件组成，如上图所示；
-RPC Server: 该协议实现了ContainerManagementProtocol协议，实现AM与NM之前的通信通道，通过该协议接受来自各个AM的启动或者杀死Container的命令; 
-ResourceLocalizationService: 负责Container所需资源的本地化，按照资源描述将资源从HDFS上下载所需资源，并将这些资源均摊给各个磁盘以防数据热点;  
-ContainerLauncher: 维护一个线程池以并行的方式完成Container的操作，比如来自AM的启动Container，来自AM或者RM的kill Container；  
-AuxServices: NM允许用户添加附属服务;  
-ContainersMonitor: 监控Container资源的使用状况，周期性的检查Container资源使用情况，一旦有Container超过它允许使用资源的上限则kill掉；  
-Loghandler: 可插拔的组件，控制Container日志保存的方式；  
-ContainerEventDispatcher: Container的事件调度器，负责将ContainerEvent类型的事件调度给对应Container的状态机ContainerImpl;  
-ApplicationEventDispatcher: Application的事件调度器,负责将ApplicationEvent类型的事件调度给Application的状态机ApplicationImpl;  
3. ContainerExecutor: 它可与底层的操作系统交互，安全存放Container的文件和目录，进而安全的启动和清理Container；
4. NodeHealthCheckerService: 周期性的运行一个向磁盘写文件的脚本,检测节点的健康状态，并汇报给RM；  
5. DeletionService: NodeManager删除文件组件；  
6. Security: 安全模块;  
7. WebServer： web页面展示该节点上应用程序的状态;  



### NodeManager的事件与事件处理器 
NodeManager主要组件也是通过事件进行交互的，这使得组件能够异步并发完成各种功能；如下图所示:  
![](https://ws3.sinaimg.cn/large/006tNbRwly1fwzip0pg3gj31gm188td9.jpg)

### 节点健康状态监测
节点健康状态监测是NodeManager自带的健康状态诊断机制，通过该机制，NodeManager可以时刻的掌握资深的健康状况，并及时的汇报给RM，RM根据节点的健康状况调整分配的任务数目;如果NodeManager监测到自身的状态为“unhealthy”,则通知RM不再为该节点分配任务，直到状态转为"healthy"再向该节点分配任务;

**自定义监测脚本**   
NodeManager有两种判断节点健康状态的方法：  
第一种: 管理员自定义shell脚本，NodeManager上有一个周期性的执行脚本，如果脚本监测到节点为不健康状态，则脚本只要标准输出“ERROR”开头的字符串即可，NodeHealthScriptRunner周期性的监测发现脚本输出“ERROR”开头的字符串则认为该节点为不健康的节点，并通过心跳通知RM，RM将会把该节点加入黑名单，知道该节点的状态转为健康状态，RM再将其移除黑名单，并为该节点分配任务；  
NodeHealthScriptRunner服务主要工作是周期性的执行节点健康状况的监测脚本，即上面所说的自定义脚本,但需要在yarn-site.xml中配置,如下:  
![](https://ws2.sinaimg.cn/large/006tNbRwly1fwyihkm1gzj31fm0hmjvr.jpg)
自定义脚本案例如下:  
![](https://ws4.sinaimg.cn/large/006tNbRwly1fwyiicsulnj31go0f4q4w.jpg)

第二种: 判断磁盘的好坏，NodeManager上有一个周期性的执行脚本监测磁盘的好坏，默认的情况下该功能是开启的，该功能由LocalDirsHandlerService服务实现的，它周期性的任务检测NodeManager本地磁盘的好坏，并通过心跳将健康状态汇报给RM；NodeManager有两个目录一个是本地可用的目录列表用于存放程序或者任务的中间结果，另一个是日志目录，而LocalDirsHandlerService服务就是检测这些目录的好坏，一旦发现正常磁盘的比例低于设定的值(yarn.nodemanager.disk-health-checker.min-healthy-disks默认0.25)则将节点置为不健康状态;

### 分布式缓存
**分布式缓存介绍**  
在Yarn中，分布式缓存是一中分布式文件分发和缓存机制，主要作用是将用户应用程序需要的外部文件资源自动透明的下载并缓存到各个节点上，从而省去了用户手动部署的麻烦，**注意，这里的分布式缓存并不是将文件缓存到集群各个节点的内存中,而是将文件缓存到各个节点的本地磁盘上;**  
Yarn的分布式缓存工作流程如下:  
![](https://ws1.sinaimg.cn/large/006tNbRwly1fwzigj76l1j31j00uy0vv.jpg)
1. 客户端将应用程序所需的文件资源提交到HDFS上;
2. 客户端将应用程序提交给RM；
3. RM与某个NM通信，启动应用程序的ApplicationMaster，NodeManager收到命令后，先从HDFS上下载文件资源，然后启动ApplicationMaster；
4. ApplicationMaster与RM通信请求获取计算资源；
5. ApplicationMaster收到RM分配的资源后与NM通信启动任务；
6. 如果该应用程序第一次再该节点上启动任务，则NN先从HDFS上下载文件资源缓存在本地，然后再启动任务；
7. NM后续在收到启动任务的请求时，先判断所需的文件资源是否已经下载到本地，如果没有下载到本地则等待文件缓存完毕后再启动任务;

**资源可见性和分类**  
分布式缓存机制是由各个NM实现的，主要功能是将应用程序所需的文件资源缓存到本地，以便后续任务的使用，资源缓存是用时触发的，也就是第一个用到该资源的任务触发，后续任务无需再进行缓存，直接使用即可；  
根据可见性,NM将资源分为三类：  
1. Public:节点上所有的用户都可以共享该资源，只要有一个用户的应用程序将着这些资源缓存到本地，其他所有用户的所有应用程序都可以使用；
2. Private: 节点上同一用户的所有应用程序共享该资源,只要该用户其中一个应用程序将资源缓存到本地，该用户的所有应用程序都可以使用；
3. Application: 节点上同一应用程序的所有Container共享该资源;
![](https://ws4.sinaimg.cn/large/006tNbRwly1fwzj0izs48j31d20mawjs.jpg)

根据资源类型，NM向资源分为三类：  
1. archive: 归档文件,支持.jar,.zip,.tar.gz,.tgz,.tar5中归档文件；
2. file : 普通文件，NM只是将这类文件下载到本地目录，不做任何处理；
3. pattern: 以上两种文件的混合体；

注意点:  
1. YARN是通过比较resource、type、timestamp和pattern四个字段是否相同来判断两个资源请求是否相同的。如果一个已经被缓存到各个节点上的文件被用户修改了，则下次使用时会自动触发一次缓存更新，以重新从HDFS上下载文件。
2. 分布式缓存完成的主要功能是文件下载，涉及大量的磁盘读写，因此整个过程采用了异步并发模型加快文件下载速度，以避免同步模型带来的性能开销。为了防止缓存过多磁盘被撑爆，NM会采用LRU算法定期清理这些文件;


### 目录结构  

NodeManager上的目录可分为两种:  
1. **数据目录**: 存放执行Container所需的数据(比如可执行程序或JAR包、配置文件等)和运行过程中产生的临时数据,由yarn.nodemanager.local-dirs指定;
2. **日志目录**: 存放Container运行输出日志，由yarn.nodemanager.log-dirs指定;

NM在每个磁盘上为该作业创建相同的目录结构，且采用轮询的调度方式将目录（磁盘）分配给不同的Container的不同模块以避免干扰。  
考虑到一个应用程序的不同Container之间存在依赖，为了避免提前清除已经完成的Container输出的中间数据破坏应用程序的设计逻辑，YARN统一规定，只有当应用程序运行结束后，才统一清楚Container产生的中间数据。

### 状态机管理

NM维护了三类状态机，分别是:Application、Container和LocalizedResource,它们均直接或者间接参与维护一个应用程序的生命周期;  
当NodeManager收到来自某个应用程序第一次Container启动命令时，会创建一个Application状态机跟踪该应用程序在该结点上的生命周期，而每个Container的运行过程同样由一个状态机维护。此外Application所需的资源(比如文本文件、JAR包、归档文件等）需要从HDFS上下载，每个资源的下载过程均由一个状态机LocalizedResouce维护和跟踪。  

状态机如下：  
1. **Application状态机**  
NM上Application维护的信息是RM端Application信息的子集，这有利于对一个节点上的同一个Application的所有Container进行统一管理（比如记录每一个Application运行在该节点上的Container列表，杀死一个Application的所有Container等）。它实际的实现类是ApplicationImpl，该类维护了一个Application状态机，记录了Application可能存在的各个状态以及导致状态间转换的事件。需要注意的是NM上的Application生命周期与ResourceManager上Application的生命周期是一致的。

2. **Container状态机**   
Container是NM中用于维护Container生命周期的数据结构，它的是现实ContainerImpl，该类维护了一个Container的状态机，记录了Container可能存在的各种状态以及导致状态转换的事件;

3. **LocalizedResource状态机**   
LocalizedResource是NodeManager中维护一种“资源”(资源文件、JAR包、归档文件等外部文件资源)生命周期的数据结构，它维护了一个状态，记录了"资源"可能存在的各种状态以及导致状态间转换的事件。

















