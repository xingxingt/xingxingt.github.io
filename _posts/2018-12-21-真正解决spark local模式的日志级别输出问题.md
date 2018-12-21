---
layout:     post
title:      真正解决spark local模式的日志级别输出问题
subtitle:   真正解决spark local模式的日志级别输出问题,亲测，靠谱
date:       2018-12-21
author:     XINGXING
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Spark
---

>
>真正解决spark local模式的日志级别输出问题
> 

在IDEA中开发Spark程序，程序一执行密密麻麻的Info日志一大堆，这让人很恶心，如下图：
![](https://ws3.sinaimg.cn/large/006tNbRwgy1fyedm312lej32d00mi77j.jpg)


很早以前就解决过一次，不过谷歌百度都没有真正的解决这个问题，试过无数遍的将log4j.properties文件放在工程的resources目录下都没用，绝望之际打开spark工程源代码，直接搜索log4j-defaults.properties文件位置，如下图:
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fyedrx20tmj31qn0u041q.jpg)

spark工程源码中的log4j-defaults.properties是放在resources目录下的org.apapche.spark下,看到这个目录结构我在想我可以尝试下在我工程的resources目录下也建立org.apapche.spark目录，并将log4j-defaults.properties放在这个目录下，如下图所示：
![](https://ws3.sinaimg.cn/large/006tNbRwgy1fyedy7ujapj30wk0lkwfa.jpg)

然后修改log4j-defaults.properties：
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fyedz55ewij31cy0tc40i.jpg)

再次运行spark程序,完美解决:
![](https://ws1.sinaimg.cn/large/006tNbRwgy1fyee0e23xfj32fk0o03zw.jpg)
