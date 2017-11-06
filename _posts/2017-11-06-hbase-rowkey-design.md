---
layout: post
title: RowKey设计准则
comments: true
categories:
  - HBase学习
---
## RowKey设计准则
HBase是一个key-value数据库，逻辑上看它一行可以包含多个列，列数可以不固定，这是HBase的一个优势。有一点需要注意：HBase虽然一个rowkey可以对应很多列（column），但是由于key-value数据库的性质，每存一列，rowkey就同样也要再存一次，HBase也不例外。因此rowkey如果过大，会导致存储量变得很大会导致存储量变得很大。
HBase中的行是按照rowkey的字典顺序排序的HBase中的行是按照rowkey的字典顺序排序的，这种设计优化了scan操作，可以将相关的行以及会被一起读取的行存取在临近位置，便于scan。然而糟糕的rowkey设计是热点的源头，有可能某些具有特定特点的rowkey前缀发生大量集聚，导致读写某个regionServer的region性能下降。
下边是一些常见的避免热点的方法：
###### 加盐，即在rowkey前边加入随机数，使得数据分散在不同的region上
###### 哈希，哈希使得同一行永远用一个前缀加盐，与加盐的区别就是读是有可预测性的。
###### 反转，假如rowkey是手机号，可以将手机号倒序储存作为rowkey，使得经常改变的部分放在前边，避免18*,13*这样固定格式的开头导致的热点。
###### 如果数据越新，越容易在查询的时候被命中，那么rowkey的设计最好使用`[key]+[Long.Max_Value - timestamp]`的形式，这样的话，使用`scan[key]`时拿到的记录就是最新的。