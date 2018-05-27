---
layout: post
title: 系统异常的定位
comments: true
categories:
  - JAVA学习
---

## MAT工具定位系统异常
对于系统异常时，一般采取重启tomcat的方式来快速回复服务，我们应该考虑保留一台有问题的机器处于服务下线状态，方便导出当时的系统快照，分析异常原因。下线并不是指关闭tomcat，而是类似于修改ng或者修改分布式系统中标识机器可用的那个东西，以达到没有请求能够访问到本台机器，但是机器的tomcat还是运行状态

### 获取PID
这个很简单了，在之前的博客中也有写过，`ps -ef | grep tomcat`，查询到对应web服务的进场号。（这里方便举例，使用web服务)

### 对于频繁的FULL GC
- Dump下JVM的Heap，利用上边说的命令获取到的pid,`sudo jmap -F -dump:format=b,file=dump.bin ${PID}`,这样的话dump.bin就是我们需要的文件。
- 将导出的dump.bin用MAT打开(http://www.eclipse.org/mat/), 通过MAT来分析FULL GC的问题。
- ![placeholder](http://img.blog.csdn.net/20160223200720426 "MAT内存问题")
- 对于MAT的安装问题，具体可以参考 **[雪水的博客](http://blog.csdn.net/wanghuiqi2008/article/details/50724676)**
- 如果在分析的时候报internal error 可能是jvm的参数设置过小，这里可以通过修改 **`MemoryAnalyzer.ini`** 中的 **`-Xmx1024m`** 参数，默认是1024m，如果你本机内存比较大，可以设置的大一点，但是最好不要超过本机的内存大小。

### 其他的系统异常
- **负载CPU使用率过高**，可以使用**`ps p 3912 -L -o pcpu,pid,tid,time,tname,cmd`** 将有问题的线程号 `TID` (十进制) 转换成十六进制，如3046对应0xb46。然后 **`sudo jstack -F 3912 |sudo tee stack.log`** 打印出stack的快照。最后在stack.log文件中查找0xb46对应的线程信息。
- **GC监控**，实时，使用**`jstat -gcutil pid 间隔毫秒数  获取次数`**命令。