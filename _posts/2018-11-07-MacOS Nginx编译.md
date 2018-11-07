---
layout:     post
title:      MacOS Nginx编译
subtitle:   MacOS Nginx编译
date:       2018-11-07
author:     XINGXING
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Nginx
---

>
>MacOS Nginx编译解析篇
> 

### 下载Nginx
先下载Nginx，开源免费版本下载地址,我这里用的是最新稳固版本1.14.0 ：

    http://nginx.org/en/download.html
    
![](https://ws2.sinaimg.cn/large/006tNbRwly1fwz9zstxayj30qy0a2t92.jpg)

### 编译Nginx 
**步骤一:**  
执行命令:

    ./configure 
可以追加选项，例如指定编译目录:  

    ./configure  --prefix=/Users/axing/opt/

我在这里并没有指定编译的路径而是直接 `./configure` 命令, 一波日志打印后输出了Error信息，提示的信息很友好很详细，提示缺少PCRE模块，你可以使用
`--without-http_rewrite_module`选项忽略,也可以`--with-pcre=`选项添加模块;  
![](https://ws3.sinaimg.cn/large/006tNbRwgy1fwyqtkdgjuj31ck0dg77b.jpg)

我选择的是自己下载PCRE自己编译，PCRE下载地址:

    https://ftp.pcre.org/pub/pcre/
    
刚开始我下载的是`pcre2-10.30`版本，开始编译:

    tar -xvf pcre2-10.30.tar.gz
    ./configure
    make
    sudo make install
    make -k check
    
命令执行后PCRE编译完美出错，如下图所示:  
![](https://ws2.sinaimg.cn/large/006tNbRwly1fwzadwsmr1j30qm0betam.jpg) 

Google了一把说是10版本不支持,好，那我更换8版本，下载了pcre-8.41版本，又重新编译一遍，Nice!问题得到解决；
![](https://ws2.sinaimg.cn/large/006tNbRwly1fwzai2o2t0j30qi0dyaca.jpg)

PCRE搞定后再回到Nginx的编译，执行如下命令,查看日志一切正常:

       ./configure  --with-pcre=/Users/axing/opt/pcre-8.41
       
**步骤二**  
执行`make`和`make install`操作，如果执行`make install`出现权限问题,执行`sudo make install`命令即可;

### 验证
经过上面的操作Nginx编译完毕，接下来我们来验证下编译后的Nginx，进入编译后的目录启动Nginx,默认在`/usr/local/nginx/ `:

    cd  /usr/local/nginx/
    sudo ./sbin/nginx

启动Nginx后访问`127.0.0.1`,看到欢迎界面就大功告成了:
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fwzaszxpgbj31kw0h3q3g.jpg)






    
    
