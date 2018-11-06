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









