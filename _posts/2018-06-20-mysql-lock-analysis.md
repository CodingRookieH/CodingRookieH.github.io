---
layout: post
title: 解析MYSQL中的锁
comments: true
categories:
  - JAVA学习
---

## 解析MYSQL中的锁
在上一篇博客中，博主遇到了一个死锁的案例。但是真的要弄清楚MYSQL中的锁，确实是一件很复杂的事情。这里我只针对InnoDB引擎，介绍一些MYSQL中的锁，主要为了给各位提供一个解决死锁的思路。（当然，如果不了解MYSQL索引原理的同学，还是先了解一些MYSQL索引的原理）。

### 一、MYSQL执行一条语句的过程是怎样的呢?
关于 MySQL 的索引是一个很大的话题，譬如，增删改查时 B+ 树的调整算法是怎样实现的，如何通过索引加快 SQL 的执行速度，如何优化索引，等等等等。我们这里为了加强对锁的理解，只需要了解索引的数据结构即可。当执行下面的 SQL 时（id 为 students 表的主键），我们要知道，InnoDb 存储引擎会在 id = 49 这个主键索引上加一把 X 锁。

```mysql> update students set score = 100 where id = 49;```

当执行下面的 SQL 时（name 为 students 表的二级索引），InnoDb 存储引擎会在 name = 'Tom' 这个索引上加一把 X 锁，同时会通过 name = 'Tom' 这个二级索引定位到 id = 49 这个主键索引，并在 id = 49 这个主键索引上加一把 X 锁。

```mysql> update students set score = 100 where name = 'Tom';```

![placeholder](http://www.aneasystone.com/usr/uploads/2017/10/1423030488.png)

像上面这样的 SQL 比较简单，只操作单条记录，如果要同时更新多条记录，加锁的过程又是什么样的呢？譬如下面的 SQL（假设 score 字段为二级索引）：

```mysql> update students set level = 3 where score >= 60;```

下图展示了当用户执行这条 SQL 时，MySQL Server 和 InnoDb 之间的执行流程：

![placeholder](http://www.aneasystone.com/usr/uploads/2017/10/201797556.png)

从图中可以看到当 UPDATE 语句被发给 MySQL 后，MySQL Server 会根据 WHERE 条件读取第一条满足条件的记录，然后 InnoDB 引擎会将第一条记录返回并加锁（current read），待 MySQL Server 收到这条加锁的记录之后，会再发起一个 UPDATE 请求，更新这条记录。一条记录操作完成，再读取下一条记录，直至没有满足条件的记录为止。因此，MySQL 在操作多条记录时 InnoDB 与 MySQL Server 的交互是一条一条进行的，加锁也是一条一条依次进行的，先对一条满足条件的记录加锁，返回给 MySQL Server，做一些 DML 操作，然后在读取下一条加锁，直至读取完毕。理解这一点，对我们后面分析复杂 SQL 语句的加锁过程将很有帮助。

### 二、锁的种类
根据锁的粒度可以把锁细分为表锁和行锁，行锁根据场景的不同又可以进一步细分，在 MySQL 的源码里，定义了四种类型的行锁，如下：
``` 
#define LOCK_TABLE  16  /* table lock */
#define LOCK_REC    32  /* record lock */

/* Precise modes */
#define LOCK_GAP    512 
#define LOCK_REC_NOT_GAP 1024  
#define LOCK_INSERT_INTENTION 2048 
```
- LOCK_ORDINARY：也称为 **Next-Key Lock**，锁一条记录及其之前的间隙，这是 RR 隔离级别用的最多的锁，从名字也能看出来；
- LOCK_GAP：间隙锁，锁两个记录之间的 GAP，防止记录插入；
- LOCK_REC_NOT_GAP：只锁记录；
- LOCK_INSERT_INTENSION：插入意向 GAP 锁，插入记录时使用，是 LOCK_GAP 的一种特例。

这四种行锁将是理解并解决数据库死锁的关键，我们下面将深入研究这四种锁的特点。但是在介绍这四种锁之前，让我们再来看下 MySQL 下锁的模式。

### 三、锁的类型
MySQL 将锁分成两类：锁类型（lock_type）和锁模式（lock_mode）。锁类型就是上文中介绍的表锁和行锁两种类型，当然行锁还可以细分成记录锁和间隙锁等更细的类型，锁类型描述的锁的粒度，也可以说是把锁具体加在什么地方；而锁模式描述的是到底加的是什么锁，譬如读锁或写锁。锁模式通常是和锁类型结合使用的，锁模式在 MySQL 的源码中定义如下：
``` 
/* Basic lock modes */
enum lock_mode {
    LOCK_IS = 0, /* intention shared */
    LOCK_IX,    /* intention exclusive */
    LOCK_S,     /* shared */
    LOCK_X,     /* exclusive */
    LOCK_AUTO_INC,  /* locks the auto-inc counter of a table in an exclusive mode*/
    ...
};
```
- LOCK_IS：读意向锁；
- LOCK_IX：写意向锁；
- LOCK_S：读锁；
- LOCK_X：写锁；
- LOCK_AUTO_INC：自增锁；

将锁分为读锁和写锁主要是为了提高读的并发，如果不区分读写锁，那么数据库将没办法并发读，并发性将大大降低。而 IS（读意向）、IX（写意向）只会应用在表锁上，方便表锁和行锁之间的冲突检测。LOCK_AUTO_INC 是一种特殊的表锁。

#### 3.1 读写锁
读锁和写锁都是最基本的锁模式，它们的概念也比较容易理解。读锁，又称共享锁（Share locks，简称 S 锁），加了读锁的记录，所有的事务都可以读取，但是不能修改，并且可同时有多个事务对记录加读锁。写锁，又称排他锁（Exclusive locks，简称 X 锁），或独占锁，对记录加了排他锁之后，只有拥有该锁的事务可以读取和修改，其他事务都不可以读取和修改，并且同一时间只能有一个事务加写锁。（注意：这里说的读都是当前读，快照读是无需加锁的，记录上无论有没有锁，都可以快照读）。

#### 3.2 读写意向锁
表锁锁定了整张表，而行锁是锁定表中的某条记录，它们俩锁定的范围有交集，因此表锁和行锁之间是有冲突的。譬如某个表有 10000 条记录，其中有一条记录加了 X 锁，如果这个时候系统需要对该表加表锁，为了判断是否能加这个表锁，系统需要遍历表中的所有 10000 条记录，看看是不是某条记录被加锁，如果有锁，则不允许加表锁，显然这是很低效的一种方法，为了方便检测表锁和行锁的冲突，从而引入了意向锁。
意向锁为表级锁，也可分为读意向锁（IS 锁）和写意向锁（IX 锁）。当事务试图读或写某一条记录时，会先在表上加上意向锁，然后才在要操作的记录上加上读锁或写锁。这样判断表中是否有记录加锁就很简单了，只要看下表上是否有意向锁就行了。意向锁之间是不会产生冲突的，也不和 AUTO_INC 表锁冲突，它只会阻塞表级读锁或表级写锁，另外，意向锁也不会和行锁冲突，行锁只会和行锁冲突。
下面是各个表锁之间的兼容矩阵：

![placeholder](http://www.aneasystone.com/usr/uploads/2017/10/1431433403.png)

从表中我们可以清楚的看到：
- X 锁和其他所有锁都有冲突。
- IX 锁之间没有冲突（这个也是上篇博客中导致死锁的根本原因之一）。
- 其他的信息大家对号入座就好。

### 四、锁类型+锁种类产生的化学反应
前面在讲行锁时有提到，在 MySQL 的源码中定义了四种类型的行锁，我们这一节将学习这四种锁。在我刚接触数据库锁的概念时，我理解的行锁就是将锁锁在行上，这一行记录不能被其他人修改，这种理解其实很肤浅，因为行锁也有可能并不是锁在行上而是行与行之间的间隙上，事实上，我理解的这种锁是最简单的行锁模式：记录锁。

#### 4.1 记录锁
[**记录锁**](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-record-locks) 是最简单的行锁，并没有什么好说的。譬如下面的 SQL 语句（id 为主键）：

```mysql> UPDATE accounts SET level = 100 WHERE id = 5;```

这条 SQL 语句就会在 id = 5 这条记录上加上记录锁，防止其他事务对 id = 5 这条记录进行修改或删除。记录锁永远都是加在索引上的，就算一个表没有建索引，数据库也会隐式的创建一个索引。如果 WHERE 条件中指定的列是个二级索引，那么记录锁不仅会加在这个二级索引上，还会加在这个二级索引所对应的聚簇索引上（参考上面的加锁流程一节）。
注意，如果 SQL 语句无法使用索引时会走主索引实现全表扫描，这个时候 MySQL 会给整张表的所有数据行加记录锁。如果一个 WHERE 条件无法通过索引快速过滤，存储引擎层面就会将所有记录加锁后返回，再由 MySQL Server 层进行过滤。不过在实际使用过程中，MySQL 做了一些改进，在 MySQL Server 层进行过滤的时候，如果发现不满足，会调用 unlock_row 方法，把不满足条件的记录释放锁（显然这违背了二段锁协议）。这样做，保证了最后只会持有满足条件记录上的锁，但是每条记录的加锁操作还是不能省略的。可见在没有索引时，不仅会消耗大量的锁资源，增加数据库的开销，而且极大的降低了数据库的并发性能，所以说，更新操作一定要记得走索引。

#### 4.2 间隙锁
还是看上面的那个例子，如果 id = 5 这条记录不存在，这个 SQL 语句还会加锁吗？答案是可能有，这取决于数据库的隔离级别。
还记得MYSQL中有一个很难完全解决的问题，叫做 **幻读**，指的是在同一个事务中同一条 SQL 语句连续两次读取出来的结果集不一样。在 read committed 隔离级别很明显存在幻读问题，在 repeatable read 级别下，标准的 SQL 规范中也是存在幻读问题的，但是在 MySQL 的实现中，使用了间隙锁的技术避免了幻读。
[**间隙锁**](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-gap-locks) 是一种加在两个索引之间的锁，或者加在第一个索引之前，或最后一个索引之后的间隙。有时候又称为范围锁（Range Locks），这个范围可以跨一个索引记录，多个索引记录，甚至是空的。使用间隙锁可以防止其他事务在这个范围内插入或修改记录，保证两次读取这个范围内的记录不会变，从而不会出现幻读现象。很显然，间隙锁会增加数据库的开销，虽然解决了幻读问题，但是数据库的并发性一样受到了影响，所以在选择数据库的隔离级别时，要注意权衡性能和并发性，根据实际情况考虑是否需要使用间隙锁，大多数情况下使用 read committed 隔离级别就足够了，对很多应用程序来说，幻读也不是什么大问题。
回到这个例子，这个 SQL 语句在 RC 隔离级别不会加任何锁，在 RR 隔离级别会在 id = 5 前后两个索引之间加上间隙锁。
值得注意的是，间隙锁和间隙锁之间是互不冲突的，间隙锁唯一的作用就是为了防止其他事务的插入，所以加间隙 S 锁和加间隙 X 锁没有任何区别。

#### 4.3 Next-Key Locks

[**Next-key 锁**](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-next-key-locks) 是记录锁和间隙锁的组合，它指的是加在某条记录以及这条记录前面间隙上的锁。假设一个索引包含10、11、13 和 20 这几个值，可能的 Next-key 锁如下：

- (-∞, 10]
- (10, 11]
- (11, 13]
- (13, 20]
- (20, +∞)

通常我们都用这种左开右闭区间来表示 Next-key 锁，其中，圆括号表示不包含该记录，方括号表示包含该记录。前面四个都是 Next-key 锁，最后一个为间隙锁。和间隙锁一样，**在 RC 隔离级别下没有 Next-key 锁，只有 RR 隔离级别才有。**继续拿上面的 SQL 例子来说，如果 id 不是主键，而是二级索引，且不是唯一索引，那么这个 SQL 在 RR 隔离级别下会加什么锁呢？答案就是 Next-key 锁，如下：

- (a, 5]
- (5, b)

其中，a 和 b 是 id = 5 前后两个索引，我们假设 a = 1、b = 10，那么此时如果插入一条 id = 3 的记录将会阻塞住。之所以要把 id = 5 前后的间隙都锁住，仍然是为了解决幻读问题，因为 id 是非唯一索引，所以 id = 5 可能会有多条记录，为了防止再插入一条 id = 5 的记录，必须将下面标记 ^ 的位置都锁住，因为这些位置都可能再插入一条 id = 5 的记录：

1 ^ 5 ^ 5 ^ 5 ^ 10 11 13 15

可以看出来，Next-key 锁确实可以避免幻读，但是带来的副作用是连插入 id = 3 这样的记录也被阻塞了，这根本就不会引起幻读问题的。
关于 Next-key 锁，有一个比较有意思的问题，比如下面这个 orders 表（id 为主键，order_id 为二级非唯一索引）：
``` 
+-----+----------+
|  id | order_id |
+-----+----------+
|   1 |        1 |
|   3 |        2 |
|   5 |        5 |
|   7 |        5 |
|  10 |        9 |
+-----+----------+
```
事务 A 执行下面的 SQL：
``` 
mysql> begin;
mysql> select * from orders where order_id = 5 for update;
+-----+----------+
|  id | order_id |
+-----+----------+
|   5 |        5 |
|   7 |        5 |
+-----+----------+
2 rows in set (0.00 sec)
```

这个时候不仅 order_id = 5 这条记录会加上 X 记录锁，而且这条记录前后的间隙也会加上锁，加锁位置如下：

(2,5]  ,   (5,9]   ,   (9,+∞)

可以看到 (2, 9) 这个区间都被锁住了，这个时候如果插入 order_id = 4 或者 order_id = 8 这样的记录肯定会被阻塞，这没什么问题，那么现在问题来了，如果插入一条记录 order_id = 2 或者 order_id = 9 会被阻塞吗？答案是可能阻塞，也可能不阻塞，这取决于插入记录主键的值，感兴趣的读者可以参考[这篇博客](http://blog.sina.com.cn/s/blog_a1e9c7910102vnrj.html)。

#### 4.4 插入意向锁
[**插入意向锁**](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-insert-intention-locks) 是一种特殊的间隙锁（所以有的地方把它简写成 II GAP），这个锁表示插入的意向，只有在 INSERT 的时候才会有这个锁。注意，这个锁虽然也叫意向锁，但是和上面介绍的表级意向锁是两个完全不同的概念，不要搞混淆了。插入意向锁和插入意向锁之间互不冲突，所以可以在同一个间隙中有多个事务同时插入不同索引的记录。譬如在上面的例子中，id = 1 和 id = 5 之间如果有两个事务要同时分别插入 id = 2 和 id = 3 是没问题的，虽然两个事务都会在 id = 1 和 id = 5 之间加上插入意向锁，但是不会冲突。
插入意向锁只会和间隙锁或 Next-key 锁冲突，正如上面所说，间隙锁唯一的作用就是防止其他事务插入记录造成幻读，那么间隙锁是如何防止幻读的呢？正是由于在执行 INSERT 语句时需要加插入意向锁，而插入意向锁和间隙锁冲突，从而阻止了插入操作的执行。

#### 4.5 行锁的兼容矩阵
下面我们对这四种行锁做一个总结，它们之间的兼容矩阵如下图所示：

![placeholder](http://www.aneasystone.com/usr/uploads/2017/11/3404508090.png)

其中，**第一行表示已有的锁，第一列表示要加的锁**。这个矩阵看起来很复杂，因为它是不对称的，如果要死记硬背可能会晕掉。其实仔细看可以发现，不对称的只有插入意向锁，所以我们先对插入意向锁做个总结，如下：
- 插入意向锁不影响其他事务加其他任何锁。也就是说，一个事务已经获取了插入意向锁，对其他事务是没有任何影响的；
- 插入意向锁与间隙锁和 Next-key 锁冲突。也就是说，一个事务想要获取插入意向锁，如果有其他事务已经加了间隙锁或 Next-key 锁，则会阻塞。

### 五、在 MySQL 中观察行锁
为了更好的理解不同的行锁，下面我们在 MySQL 中对不同的锁实际操作一把。有两种方式可以在 MySQL 中观察行锁，第一种是通过下面的 SQL 语句：

```mysql> select * from information_schema.innodb_locks;```

这个命令会打印出 InnoDb 的所有锁信息，包括锁 ID、事务 ID、以及每个锁的类型和模式等其他信息。第二种是使用下面的 SQL 语句：

```mysql> show engine innodb status\G```

这个命令并不是专门用来查看锁信息的，而是用于输出当前 InnoDb 引擎的状态信息，包括：BACKGROUND THREAD、SEMAPHORES、TRANSACTIONS、FILE I/O、INSERT BUFFER AND ADAPTIVE HASH INDEX、LOG、BUFFER POOL AND MEMORY、ROW OPERATIONS 等等。其中 TRANSACTIONS 部分会打印当前 MySQL 所有的事务，如果某个事务有加锁，还会显示加锁的详细信息。如果发生死锁，也可以通过这个命令来定位死锁发生的原因。不过在这之前需要先打开 Innodb 的锁监控：

``` 
mysql> set global innodb_status_output = ON;
mysql> set global innodb_status_output_locks = ON;
```

打开锁监控之后，使用 `show engine innodb status` 命令，会输出大量的信息，我们在其中可以找到 **TRANSACTIONS** 部分，同时产生死锁时，也能更方便观察问题。
要注意的是，只有在两个事务出现锁竞争时才能在这个表中看到锁信息，譬如你执行一条 UPDATE 语句，它会对某条记录加 X 锁，这个时候 `information_schema.innodb_locks` 表里是没有任何记录的。
另外，只看这个表只能得到当前持有锁的事务，至于是哪个事务被阻塞，可以通过 `information_schema.innodb_lock_waits` 表来查看。相关语句为：
``` 
SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS;
SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCK_WAITS;
```

### 六、参考

 **[http://www.aneasystone.com/archives/2017/12/solving-dead-locks-three.html](http://www.aneasystone.com/archives/2017/12/solving-dead-locks-three.html)**
 **[http://keithlan.github.io/2017/06/21/innodb_locks_algorithms/](http://keithlan.github.io/2017/06/21/innodb_locks_algorithms/)** 
 **[http://keithlan.github.io/2017/06/05/innodb_locks_1/](http://keithlan.github.io/2017/06/05/innodb_locks_1/)**

