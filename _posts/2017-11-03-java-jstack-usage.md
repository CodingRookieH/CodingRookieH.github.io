---
layout: post
title: JStack的使用
categories:
  - JAVA学习
---

## JStack的使用
JStack是利用Thread Dump，能够让使用者看到线程的运行状态。他的使用是要比看线上日志可靠的多的，因为线上日志可能并没有处理某些异常的堆栈,因此利用JStack定位问题是很准确、快速的。
#### 线程的状态
JDK 1.8中我们是能够看到，线程的状态有**NEW**，**RUNNABLE**，**BLOCKED**，**WAITING**，**TIMED_WAITING**，**TERMINATED** 6种状态，其中 **NEW** 和 **TERMINATED** 没什么好说的，其他四种状态主要是
- **RUNNABLE** 线程运行中或I/O等待
- **BLOCKED** 线程在等待monitor锁(synchronized关键字)
- **TIMED_WAITING** 线程在等待唤醒，但设置了时限
- **WAITING** 线程在无限等待唤醒

线程各个状态的整个流程如图所示
![placeholder](http://images.cnitblog.com/blog/325852/201302/20012759-f5110611bb224169a3eee61e2ffa77e0.png "线程状态切换图")

#### JStack的使用
在linux系统下，我们是可以使用一些命令查询到Tomcat或者其他JAVA应用的进程号（PID）的，博主一般喜欢`ps -ef | grep --color tomcat`，再通过应用名查询出对应的PID，当然也可以用Top命令查询出PID，这个比较自由。查询出PID后，我们需要查看对应进程的JStack，此时直接打`jstack $PID`，可能会报`$PID: well-known file is not secure`,博主查了查，其实这个是一个校验，大家可以查看下:*/tmp/hsperfdata_$USER/$PID*这个文件存不存在，如果存在，那么在执行jstack时需要将用户切换到$USER，如果有权限问题，那么最暴力的方法:
`sudo -u $USER jstack $PID` ,
最后:

![placeholder](http://upload-images.jianshu.io/upload_images/2184951-311ab1b4ea7dde3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "线程状态切换图")

可以看到图中线程1处于 **RUNNABLE** 状态，并且已经获取了 `0x000000076bf62208` 这个地址上的锁，而线程2 处于 **BLOCKED** 状态，正在等待 `0x000000076bf62208` 这个地址上的锁。

感谢 **[占小狼的简书](http://www.jianshu.com/p/6690f7e92f27) 以及 [圣骑士Wind](http://www.cnblogs.com/mengdd/archive/2013/02/20/2917966.html)** 的博客