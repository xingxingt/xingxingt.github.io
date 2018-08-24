---
layout:     post
title:      Git clone  Authentication failed
subtitle:   解决Git clone  Authentication failed问题
date:       2018-08-24
author:     XINGXING
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Git
---

#异常信息:
$ git clone XXXXXXXX
Cloning into 'pro_name'...
fatal: Authentication failed for 'XXXXXXXX'

#解决方法:
clone项目时 一直提示Authentication failed
最后发现问题是windows凭证的问题,应该是我修改了我的用户密码所致
win+s 搜索 “凭据” ，打开凭据管理器
删除git 之前保存的凭据，重新clone ，弹出提示框输入新的用户名和密码即可
然后在执行 git clone XXXXXXXX
