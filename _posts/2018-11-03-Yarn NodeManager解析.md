
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
**1.ResourceTrackerProtocol**
NodeManager通过该RPC协议向RM注册，汇报节点的健康状态以及Container的运行状态，并领取RM下发的命令例如重新初始化Container，清理Container等；在这个协议总NodeManager主动向RM发请求，RM相应NodeManager的请求；该RPC的代码如下:  
```proto
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



