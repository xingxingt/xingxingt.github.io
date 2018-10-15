---
layout:     post
title:      SparkStreaming源码之JobScheduler
subtitle:   SparkStreaming源码之JobScheduler
date:       2018-09-08
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spark
---

>
>SparkStreaming源码之JobScheduler篇
> 

#### 首先看下JobScheduler这个类是在什么时候被实例化的，打开StreamingContext代码可见：
![](https://ws1.sinaimg.cn/large/006tNc79gy1fvp3olf2mnj31bm0fat9g.jpg)

#### 再看下job的产生者jobGenerator是如何将生成的job传递给JobScheduler的
![](https://ws4.sinaimg.cn/large/006tNc79gy1fvp3ixpgh0j31fm0osmz6.jpg)

#### JobScheduler处理提交上来的job，并将job存放在jobSet的数据结构中
![](https://ws2.sinaimg.cn/large/006tNc79ly1fvp55329opj31cu0dudgo.jpg)

### SparkStreaming在一个Application中能够同时运行多个job的，其实就是使用多线程来实现
![](https://ws1.sinaimg.cn/large/006tNc79ly1fvp86auo1jj31kw0ciwfy.jpg)

### JobScheduler负责job的调度，在内部是使用一个消息循环体来处理job的各种事件，而这个消息循环体也是在JobSchduler的start方法中实例化
![](https://ws3.sinaimg.cn/large/006tNc79ly1fvp89dixiej31kw0nh75u.jpg)

### 看下这个消息循环体具体的内容，可见Job的启动，完成，还有错误处理都在这里，具体方法可以点进去看
![](https://ws2.sinaimg.cn/large/006tNc79ly1fvp8aq0ki6j31kw0fn3zf.jpg)

### 现在看下一个job的启动，在SubmitJobSet方法，JobExecutor线程池去执行每个JobHandler
![](https://ws3.sinaimg.cn/large/006tNc79ly1fvp8lb6ttkj31fu0f43zd.jpg)

### 看下jobHandler这个线程的run方法
![](https://ws2.sinaimg.cn/large/006tNc79ly1fvp8pl0l30j31kw122tch.jpg)

### 看下最终的run方法,这个run方法执行的是job的输出代码的方法，例如print操作产生的job
![](https://ws1.sinaimg.cn/large/006tNc79ly1fvp8r9h59gj310y0hm0tk.jpg)
![](https://ws2.sinaimg.cn/large/006tNc79ly1fvp8zb6bixj31aw0mydgt.jpg)


### 至此 JobScheduler角色的工作以叙述完毕！


