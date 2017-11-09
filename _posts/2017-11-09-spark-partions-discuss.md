---
layout: post
title: Spark中关于数据分区
comments: true
categories:
  - Spark学习
---


## Spark中关于数据分区
RDD概念：Resilient Distributed Datasets  弹性分布式数据集，是一个容错的、并行的数据结构，可以让用户显式地将数据存储到磁盘和内存中，并能控制数据的分区。同时，RDD还提供了一组丰富的操作来操作这些数据。RDD是只读的记录分区的集合，只能通过在其他RDD执行确定的转换操作（transformation操作）而创建。RDD可看作一个spark的对象，它本身存在于内存中，如对文件计算是一个RDD等。

一个RDD可以包含多个分区，每个分区就是一个dataset片段。RDD可以相互依赖。

*窄依赖* : 父RDD的每个分区仅被至多一个子RDD分区使用。
*宽依赖* : 父RDD的分区可以被多个子RDD分区使用。

==注意宽依赖和窄依赖都是在父RDD的概念上理解的。==
上个老图：
![placeholder](http://img.blog.csdn.net/20160913233559680 "宽窄数据分区示意图")

宽依赖，主要有groupByKey、reduceByKey、sortByKey等操作会产生宽依赖，会产生shuffle
窄依赖，map、filter、union等操作会产生窄依赖

**注意**：join操作有两种情况：如果两个RDD在进行join操作时，一个RDD的partition仅仅和另一个RDD中已知个数的Partition进行join，那么这种类型的join操作就是窄依赖，例如图1中左半部分的join操作(join with inputsco-partitioned)；其它情况的join操作就是宽依赖,例如图1中右半部分的join操作(join with inputsnot co-partitioned)，由于是需要父RDD的所有partition进行join的转换，这就涉及到了shuffle，因此这种类型的join操作也是宽依赖。

下边是一个方便理解的图(方框表示RDD，实心矩形表示分区)：
![placeholder](http://img.blog.csdn.net/20170803212134584?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWxidW1fZ3lk/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center "宽窄分区示意图")

相比于宽依赖，窄依赖有利的地方是：

区分这两种依赖很有用。首先，窄依赖允许在一个集群节点上以流水线的方式（pipeline）计算所有父分区。例如，逐个元素地执行map、然后filter操作；而宽依赖则需要首先计算好所有父分区数据，然后在节点之间进行Shuffle，这与MapReduce类似。第二，窄依赖能够更有效地进行失效节点的恢复，即只需重新计算丢失RDD分区的父分区，而且不同节点之间可以并行计算；而对于一个宽依赖关系的Lineage图，单个节点失效可能导致这个RDD的所有祖先丢失部分分区，因而需要整体重新计算。

参考 **[RDD：基于内存的集群计算容错抽象](http://shiyanjun.cn/archives/744.html)**