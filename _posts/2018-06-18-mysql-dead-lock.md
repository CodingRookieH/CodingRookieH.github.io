---
layout: post
title: ON DUPLICATE KEY UPDATE引起的死锁问题
comments: true
categories:
  - JAVA学习
---

## ON DUPLICATE KEY UPDATE引起的死锁问题
MYSQL中为了保证隔离级别的有效性，以及插入数据的正确性，加入了一些锁机制，比如间隙锁（GAP),排他锁，共享锁，插入意向锁等等，锁的粒度也有很多。今天谈到的一个问题是有关于数据库死锁的问题，大家在看之前最好能先看看MYSQL中的一些锁机制，方便了解.

### 前言
在进行一次业务开发的时候，我们发现在使用`REAPEATED-READ`事务隔离级别的时候，压测过长中产生了大量的死锁，建表语句为:

```Java
CREATE TABLE `group_info_snapshot_100` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '自增主键',
  `group_id` bigint(20) NOT NULL COMMENT '群id',
  `user_id` bigint(20) NOT NULL COMMENT 'user_id',
  `content` text NOT NULL COMMENT 'xxx',
  `create_time` bigint(20) NOT NULL COMMENT 'xxx',
  `update_time` bigint(20) NOT NULL COMMENT 'xxx',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uidx_group_id_user_id` (`group_id`,`user_id`)
) ENGINE=InnoDB AUTO_INCREMENT=20 DEFAULT CHARSET=utf8mb4 COMMENT='xxx'
```

死锁过程为:

| 事务1                                                    | 事务2                                                    | 事务3                                                        |
| -------------------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| Begin;                                                   | Begin;                                                   | Begin;                                                       |
| INSERT INTO xxxx <br />ON DUPLICATE KEY UPDATE<br />xxxx |                                                          |                                                              |
| Query OK, 1 row affected (38.69 sec)                     | INSERT INTO xxxx <br />ON DUPLICATE KEY UPDATE<br />xxxx | INSERT INTO xxxx <br />ON DUPLICATE KEY UPDATE<br />xxxx     |
|                                                          | Blocking…...                                             | Blocking…...                                                 |
| COMMIT;                                                  |                                                          |                                                              |
|                                                          | Query OK, 1 row affected (38.69 sec)                     | ERROR 1213 (40001): Deadlock <br />found when trying to get lock; try <br />restarting transaction |

非常不幸，事务3发现了死锁。
有点需要声明，Insert数据的时候，没有任何唯一主键的冲突，没有任何唯一主键的冲突，没有任何唯一主键的冲突，重要的事情说三遍，免得大家忽略一些事实。
### 解析
有的人会问了，不就是插入一条数据么，为什么会产生死锁？不知道你有没有注意到`ON DUPLICATE KEY UPDATE`，是的，这个东西就是本次死锁的元凶。但是为什么呢？首先，我们将整个死锁日志打印一下`show engine innodb status`，我们找到了死锁日志：

```java
LATEST DETECTED DEADLOCK
------------------------
2018-06-18 13:53:19 0x7f8f7ca03700
*** (1) TRANSACTION:
TRANSACTION 5124532, ACTIVE 40 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s), undo log entries 1
MySQL thread id 4028, OS thread handle 140250145822464, query id 305595 localhost 127.0.0.1 root update
INSERT INTO group_info_snapshot_100(`group_id`,`user_id`,`content`,`create_time`,`update_time`) VALUES(216172,xxxx,'{"basicInfo":{"groupId":"216132","admin":{"appId":2,"uid":"xxxx"},"groupStatus":"VALID","invitePermission":"EVERYONE","joinNeedPermission":"NONE","createTime":"1528954214128","updateTime":"1528954214128","defaultGroupName":"test17962、test93315、test39185、test30105","groupType":"PRIVATE"},"groupMembersBriefInfo":{"memberCount":22,"topMembers":[{"appId":2,"uid":"xxxx"},{"appId":2,"uid":"xxxx"},{"appId":2,"uid":"xxxx"},{"appId":2,"uid":"xxxx"}],"lastUpdateTime":"xxxx"}}',1528954217857,1528954217857)  ON DUPLICATE KEY UPDATE `content`='{"basicInfo":{"groupId":"216132","admin":{"appId":2,"uid":"xxxx"},"groupStatus":"VALID","invitePermission":"EVERYONE","joinNeedPermission":"NONE","createTime":"1528954214128","updateTime":"1528954214128","defaultGrou
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 46423 page no 4 n bits 72 index uidx_group_id_user_id of table `im`.`group_info_snapshot_100` trx id 5124532 lock_mode X insert intention waiting
*** (2) TRANSACTION:
TRANSACTION 5124500, ACTIVE 48 sec inserting, thread declared inside InnoDB 1
mysql tables in use 1, locked 1
4 lock struct(s), heap size 1136, 3 row lock(s), undo log entries 1
MySQL thread id 4027, OS thread handle 140254247925504, query id 305570 localhost 127.0.0.1 root update
INSERT INTO group_info_snapshot_100(`group_id`,`user_id`,`content`,`create_time`,`update_time`) VALUES(216171,xxxx,'{\"basicInfo\":{\"groupId\":\"216133\",\"admin\":{\"appId\":2,\"uid\":\"xxxx\"},\"groupStatus\":\"VALID\",\"invitePermission\":\"EVERYONE\",\"joinNeedPermission\":\"NONE\",\"createTime\":\"1528954214404\",\"updateTime\":\"1528954214404\",\"defaultGroupName\":\"test2194、test94277、test58560、test12835\",\"groupType\":\"PRIVATE\"},\"groupMembersBriefInfo\":{\"memberCount\":11,\"topMembers\":[{\"appId\":2,\"uid\":\"xxxx\"},{\"appId\":2,\"uid\":\"xxxx\"},{\"appId\":2,\"uid\":\"xxxx\"},{\"appId\":2,\"uid\":\"xxxx\"}],\"lastUpdateTime\":\"1528954217340\"}}',1528954218740,1528954218740)  ON DUPLICATE KEY UPDATE `content`='{\"basicInfo\":{\"groupId\":\"216133\",\"admin\":{\"appId\":2,\"uid\":\"xxxx\"},\"groupStatus\":\"VALID\",\"invitePermission\":\"EVERYONE\",\
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 46423 page no 4 n bits 72 index uidx_group_id_user_id of table `im`.`group_info_snapshot_100` trx id 5124500 lock_mode X
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 46423 page no 4 n bits 72 index uidx_group_id_user_id of table `im`.`group_info_snapshot_100` trx id 5124500 lock_mode X insert intention waiting
*** WE ROLL BACK TRANSACTION (1)
```

OK，找到死锁日志，仔细读一下，看到应该是`lock_mode X insert intention waiting`此处发生了锁等待并导致了死锁，but why ? 有的人肯定WTF，什么玩意都不知道的锁啊，这是啥锁，科普一下，这个是插入意向锁，是Mysql为了解决幻读问题(当然只是一部分幻读问题)使用的锁，如果插入前，插入的位置或者插入行为所涉及的位置(可能是Gap也可能不是)，插入意向锁就会被阻塞。
言归正传，我们能否将block的时候的信息DUMP下来呢？答案是可以的。

![placeholder](https://github.com/CodingRookieH/blog-image/blob/master/2018-06-18-mysql-dead-lock-step1.png?raw=true)

图中，5124462为事务1，5124500为事务2，5124532为事务3，block的时候，可以看到对于RK（直接lock在索引上），而锁等待的只有5124500（事务2）和5124532（事务3），从`blocking_trx_id`字段也能够看的出来。除此之外，我们还看到5124532（事务3）还block在5124500（事务2）上，这个其实就类似JAVA中的CLH队列了。此时锁队列中5124500（事务2）和5124532（事务3）都阻塞在5124462（事务1）的RK上，此时空间中存在的锁有：

- 锁1:`5124500（事务2）LOCK(排他锁)，阻塞在事务1上`
- 锁2:`5124532（事务3）LOCK(排他锁)，阻塞在事务1上`
- 锁3:`5124532（事务3）LOCK(排他锁),阻塞在事务2上`

那到这里，事务1释放锁（提交或者回滚）事务会发生什么呢，没错，5124500（事务2）会获取到锁，并且开始尝试获取插入意向锁，而此时队列中还有事务3的插入意向锁，空间中的锁有：

- 队列头节点:`5124500（事务2）RUNNING`
- 锁3:`5124532（事务3）LOCK(排他锁)，阻塞在事务2上`。
- 锁4:`5124500（事务2）LOCK(插入意向锁),阻塞在RK上`（表中是看不到的）
- 锁5:`5124532（事务3）LOCK(插入意向锁),阻塞在RK上`（表中是看不到的）

此时，我们可以清晰的看到锁3，锁4，锁5构成了清晰的死锁。
OK，那么唯一键都没有冲突的时候，都出现了这个问题，冲突的时候呢？
问的好，做了实验后，发现还是会出现问题，只不过排他锁变成了共享锁，换汤不换药，都是一个问题。
### 解决方式
终于到大家最关心的环节了，怎么解决？
1. 降低数据库隔离级别，前边都提到了，插入意向锁是为了解决幻读问题，那么允许出现幻读是不是就没有问题了呢？确实是没问题了，现实中我们也是这么解决的。
2. 还有2呢？有，那就是避免使用这种语句吧，你可以插入的时候catch住Duplicate Index Exception之类的，然后尝试去更新，这也许是一种方式。
最后，无论大家用哪种方式，都鼓励大家尝试，出现问题分析问题，才能够得到成长。

**参考DOC**
https://dev.mysql.com/doc/refman/5.7/en/innodb-information-schema-examples.html
https://dev.mysql.com/doc/refman/5.7/en/innodb-lock-waits-table.html
https://bugs.mysql.com/bug.php?id=52020

本文为作者原创，转载请注明出处 。**邮箱：568718043@qq.com**

