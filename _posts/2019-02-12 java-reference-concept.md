---
layout: post
title: JAVA中的引用
comments: true
categories:
  - JAVA学习
---

## JAVA中的引用
学习过JAVA的同学，对引用一定不陌生，但是这篇博客主要是重温一下非常规的引用(强引用)，因为后续准备研读一下 *gRPC* 使用堆外内存是如何管理内存进行 *GC* 的，因此整理一下基础知识。

### 强引用 ( Strong Reference )
强引用是使用最普遍的引用。如果一个对象具有强引用，那垃圾回收器绝不会回收它。当内存空间不足，Java虚拟机宁愿抛出`OutOfMemoryError`错误，使程序异常终止，也不会靠随意回收具有强引用的对象来解决内存不足的问题。 PS：强引用其实也就是我们平时A a = new A()这个意思。

### 软引用 ( Soft Reference )
是用来描述一些还有用但并非必须的对象。对于软引用关联着的对象，在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围之中进行第二次回收。如果这次回收还没有足够的内存，才会抛出内存溢出异常。
 对于软引用关联着的对象，如果内存充足，则垃圾回收器不会回收该对象，如果内存不够了，就会回收这些对象的内存。在 JDK 1.2 之后，提供了 `SoftReference `类来实现软引用。软引用可用来实现内存敏感的高速缓存。软引用可以和一个引用队列`ReferenceQueue`联合使用，如果软引用所引用的对象被垃圾回收器回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。
 **注意：Java 垃圾回收器准备对SoftReference所指向的对象进行回收时，调用对象的 finalize() 方法之前，SoftReference对象自身会被加入到这个 ReferenceQueue 对象中，此时可以通过 ReferenceQueue 的 poll() 方法取到它们。**

### 弱引用 ( Weak Reference )
用来描述非必须的对象，但是它的强度比软引用更弱一些，被弱引用关联的对象只能生存到下一次垃圾收集发生之前。当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。一旦一个弱引用对象被垃圾回收器回收，便会加入到一个注册引用队列中。
 **注意：Java 垃圾回收器准备对WeakReference所指向的对象进行回收时，调用对象的 finalize() 方法之前，WeakReference对象自身会被加入到这个 ReferenceQueue 对象中，此时可以通过 ReferenceQueue 的 poll() 方法取到它们。**

### 虚引用 ( Phantom Reference )
`PhantomReference `是所有“弱引用”中最弱的引用类型。不同于软引用和弱引用，虚引用无法通过 `get()` 方法来取得目标对象的强引用从而使用目标对象，观察源码可以发现 `get()`被重写为永远返回 `null`。
 那虚引用到底有什么作用？**其实虚引用主要被用来 跟踪对象被垃圾回收的状态，通过查看引用队列中是否包含对象所对应的虚引用来判断它是否 即将被垃圾回收，从而采取行动。它并不被期待用来取得目标对象的引用，而目标对象被回收前，它的引用会被放入一个 `ReferenceQueue` 对象中，从而达到跟踪对象垃圾回收的作用。**
 当phantomReference被放入队列时，说明referent的finalize()方法已经调用，并且垃圾收集器准备回收它的内存了。
 **注意：PhantomReference 只有当 Java 垃圾回收器对其所指向的对象真正进行回收时，会将其加入到这个 ReferenceQueue 对象中，这样就可以追综对象的销毁情况。这里referent对象的finalize()方法已经调用过了。**

### Final Reference
当前类是否是`Finalizer`类，注意这里`Finalizer`是由JVM来标志的( 后面简称 *f* 类 )，并不是指`java.lang.ref.Finalizer`类。但是 *f* 类是会被 JVM 注册到`java.lang.ref.Finalizer`类中的。
- 当前类或父类中含有一个参数为空，返回值为void的名为`finalize()`的方法。
- 并且该`finalize()`方法必须非空
GC 回收问题：
1. 对象因为`Finalizer`的引用而变成了一个临时的强引用，即使没有其他的强引用，还是无法立即被回收；
2. 对象至少经历两次GC才能被回收，因为只有在`FinalizerThread`执行完了f对象的`finalize()`方法的情况下才有可能被下次GC回收，而有可能期间已经经历过多次GC了，但是一直还没执行对象的`finalize()`方法；
3. CPU资源比较稀缺的情况下`FinalizerThread`线程有可能因为优先级比较低而延迟执行对象的`finalize()`方法；
4. 因为对象的`finalize()`方法迟迟没有执行，有可能会导致大部分f对象进入到old分代，此时容易引发old分代的GC，甚至Full GC，GC暂停时间明显变长，甚至导致OOM；
5. 对象的`finalize()`方法被调用后，这个对象其实还并没有被回收，虽然可能在不久的将来会被回收。

