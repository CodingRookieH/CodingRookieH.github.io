---
layout: post
title: AQS中共享锁源码解析
comments: true
categories:
  - JAVA学习
---

## AQS中共享锁源码解析
AQS中的非共享锁,也就是我们所说的排他锁,相信大家都不陌生,比如`ReentrantLock`中的公平锁以及非公平锁。最近小编心血来潮看了看AQS中的共享锁，于是写点东西分享下，欢迎指正。
AQS中共享锁的实现，其实很多，比如大家耳熟能详的`CountDownLatch`,`ReentrantReadWriteLock`，读写锁大部分也都是用了共享锁的思想。还是按照以前看互斥锁的思路来看。

```
　public final void acquireShared(int arg) {
        //尝试获取锁，如果没有获取到，那么就进行下边的逻辑，这里尝试获取锁的方式有很多，
        //比如CountDownLatch中就是判断AQS中的state是否为0，也就是初始化时需要CountDown
        //的次数
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```
下边来看具体逻辑：
```
    private void doAcquireShared(int arg) {
        //将该线程加入CLH队列中，这里Node的类型小编趟过坑，共享类型和排他类型，都会放入这个
        //类型，但是这个类型只是在Node中的nextWaiter中，不会在CLH队列中，只是一个标识，标
        //识是共享模式还是排他模式
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
        	//这个字段是判断在自旋过程中是否被中断，也就是@1的位置
            boolean interrupted = false;
            for (;;) {
            	//找到当前Node的前驱节点
                final Node p = node.predecessor();
                //如果前驱就是head，那么就尝试获取锁
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    //获取成功，就开始改变头节点，唤醒后继节点了
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                //如果不是头节点，那么就是喜闻乐见的流程了，这里shouldParkAfterFailedAcquire
                //大家应该不会陌生，跟排他锁中流程类似，如果当前节点的前一个节点waitStatus为
                //SIGNAL，那么我可以放心的将线程挂起了。但是如果不是，那么会一直寻找到有效的
                //前驱节点（在前驱节点是CANCELLED的情况下），或者尝试将前驱节点变成SIGNAL，
                //最后挂起线程，等待被唤醒。被唤醒后，再次在for循环中去check前驱是不是头
                //节点，再尝试获取锁。
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;                     //@1
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

```
看完上述源码，相信大家已经有了一个大概流程上的认识了，下边继续看`setHeadAndPropagate（）`方法又做了什么：
```
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        //改变头节点
        setHead(node);
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            //找到头节点的下一个节点,如果还是共享节点，那么好，开始你的表演
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```
```
    private void doReleaseShared() {
        //进入循环
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                //如果头节点的状态为SIGNAL，那么就唤醒头节点
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases

                    //唤醒后继节点，就是传说中的 LockSupport.unpark()
                    unparkSuccessor(h);
                }
                //这种情况，相信很多人不理解，我当时也是一脸懵逼，但是这里我说下我的拙见吧
                //如果不对希望大家能够指正我，我把邮箱留在下方大家可以探讨，因为disqus有
                //的人可能被墙了。
                //Node.PROPAGATE这个状态，很疑惑是干嘛的。我的理解是，在多线程情况下，
                //head的waitStatus是有可能为0的，比如CountDownLatch多处await（）的
                //时候，直接获取到锁了，那么这种情况下后继节点是不需要唤醒的
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            //如果头节点发生了改变，那么继续循环
            if (h == head)                   // loop if head changed
                break;
        }
    }
```
说到这里，尝试获取锁的过程就说完了，其实共享锁和排他锁的区别可以清楚的看到，就是锁的状态是可以共享的，获取到锁的节点会依次唤醒后继节点（这里显而易见，是按照CLH队列中添加的顺序依次向后唤醒），因此，依次会有多个线程获取到锁并进行后续操作。这就是共享的实质。

说完获取锁，那么再看释放锁：
```
    public final boolean releaseShared(int arg) {
        //尝试释放锁
        if (tryReleaseShared(arg)) {
            //释放成功，那么就唤醒后继节点
            doReleaseShared();
            return true;
        }
        return false;
    }

```
看到这里大家可能会有疑问，为什么release的时候又调用了之前见过的`doReleaseShared()`方法了呢。拿CountDownLatch为例，`countDown()`方法会将AQS中的state减去１，这时候所有调用`await()`方法的线程都会阻塞，直到state减到０。那么减到０后怎么办呢，那么就在这里，调用`doReleaseShared()`唤醒被阻塞的线程（也就是调用`await()`的线程，继续执行，讲到这里，CountDownLatch内部实现的机制相信大家也就知道了。

本文为作者原创，转载请注明出处 。**邮箱：568718043@qq.com**