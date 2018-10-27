---
layout: post
title: 一次零拷贝引发的悬案
comments: true
categories:
  - JAVA学习
---

## 一次零拷贝引发的悬案
根据 Wiki 对 Zero-copy 的定义:
> "Zero-copy" describes computer operations in which the CPU does not perform the task of copying data from one memory area to another. This is frequently used to save CPU cycles and memory bandwidth when transmitting a file over a network.
即所谓的 `Zero-copy`, 就是在操作数据时，不需要将数据 buffer 从一个内存区域拷贝到另一个内存区域。因为少了一次内存的拷贝，因此 CPU 的效率就得到的提升。
在 OS 层面上的 `Zero-copy` 通常指避免在 `用户态(User-space)` 与 `内核态(Kernel-space)` 之间来回拷贝数据。例如 Linux 提供的 `mmap` 系统调用，它可以将一段用户空间内存映射到内核空间, 当映射成功后，用户对这段内存区域的修改可以直接反映到内核空间；同样地，内核空间对这段区域的修改也直接反映用户空间。正因为有这样的映射关系, 我们就不需要在 `用户态(User-space)` 与 `内核态(Kernel-space)` 之间拷贝数据，提高了数据传输的效率。

### JAVA中的DirectByteBuffer
`DirectByteBuffer` 是 Java NIO 用于实现堆外内存的一个很重要的类，而 `Netty` 用 `DirectByteBuffer` 作为`PooledDirectByteBuf` 和 `UnpooledDirectByteBuf` 的内部数据容器（区别于 `HeapByteBuf` 直接用 `byte[]` 作为数据容器）。
在 `DirectByteBuffer` 的父类 `Buffer` 中有个 `address` 属性：
```
// Used only by direct buffers
// NOTE: hoisted here for speed in JNI GetDirectBufferAddress
long address;
```
`address` 表示分配的堆外内存的地址，JNI对这个堆外内存的操作都是通过这个 `address` 实现的。
如果我们直接使用堆外内存，即直接在堆外分配一块内存来存储数据，这样就可以避免堆内存和堆外内存之间的数据拷贝，进行I/O操作时直接将堆外内存地址传给JNI的I/O函数就好了，这也是我们NIO中最常用的一种零拷贝技术。

### Netty中的 CompositeByteBuf 实现零拷贝
而需要注意的是, Netty 中有一种 `Zero-copy` 与上面我们所提到到 OS 层面上的 `Zero-copy` 不太一样，这种 `Zero-coyp` 完全是在用户态(Java 层面)的, 它的 `Zero-copy` 的更多的是偏向于 `优化内存操作` 这样的概念。
假设我们有一份协议数据，它由头部和消息体组成，而头部和消息体是分别存放在两个 ByteBuf 中的，即：
```
ByteBuf header = ...
ByteBuf body = ...
```
我们在代码处理中, 通常希望将 header 和 body 合并为一个 ByteBuf, 方便处理, 那么通常的做法是:
```
ByteBuf allBuf = Unpooled.buffer(header.readableBytes() + body.readableBytes());
allBuf.writeBytes(header);
allBuf.writeBytes(body);
```
可以看到, 我们将 header 和 body 都拷贝到了新的 allBuf 中了, 这无形中增加了两次额外的数据拷贝操作了。
那么有没有更加高效优雅的方式实现相同的目的呢? 我们来看一下 `CompositeByteBuf` 是如何实现这样的需求的吧。
```
ByteBuf header = ...
ByteBuf body = ...

CompositeByteBuf compositeByteBuf = Unpooled.compositeBuffer();
compositeByteBuf.addComponents(true, header, body);
```
上面代码中, 我们定义了一个 `CompositeByteBuf` 对象，然后调用
```
public CompositeByteBuf addComponents(boolean increaseWriterIndex, ByteBuf... buffers) {
...
}
```
方法将 `header` 与 `body` 合并为一个逻辑上的 ByteBuf，即：
![placeholder](https://raw.githubusercontent.com/CodingRookieH/blog-image/master/2018-10-27-zero-copy/1.png)
不过需要注意的是, 虽然看起来`CompositeByteBuf`是由两个`ByteBuf`组合而成的，不过在`CompositeByteBuf`内部, 这两个`ByteBuf`都是单独存在的，`CompositeByteBuf`只是逻辑上是一个整体。

### 文件零拷贝技术：FileChannel.transferTo()
让我们看一段传输文件的一般写法吧。
```
File.read(file, buf, len);
Socket.send(socket, buf, len);
```
尽管上面的代码看起来很简单，但在内部实际包含了4次用户态-内核态上下文切换，和4次数据拷贝。
![placeholder](https://raw.githubusercontent.com/CodingRookieH/blog-image/master/2018-10-27-zero-copy/3.gif)

![placeholder](https://raw.githubusercontent.com/CodingRookieH/blog-image/master/2018-10-27-zero-copy/2.gif)
其中步骤有：
1. `read()` 调用导致了一次用户态到内核态的上下文切换，在内部，一个 `sys_read()` （或等价函数）被执行来从文件中读取数据。第一次拷贝是由 `DMA` 引擎将数据从磁盘文件存储到内核地址空间缓冲区。
2. 被请求长度的数据从内核的读缓冲区拷贝到用户缓冲区，并且 `read()` 调用返回。这个返回导致又一次从内核态到用户态的上下文切换。现在数据是存储在用户地址空间缓冲区。
3. `send()` 调用引起了一次从用户态到内核态的上下文切换。第三次拷贝又一次将数据放进内核地址空间缓冲区，尽管这一次是放进另一个不同的缓冲区，和目标socket联系在一起。
4. `send()` 系统调用返回，产生了第四次上下文切换。第四次拷贝由 `DMA` 引擎独立异步地将数据从内核缓冲区传递给协议引擎。
看到这里可能有些读者会问，`read()` 函数为什么不直接将数据拷贝到用户地址空间的缓冲区，而要经内核地址空间的缓冲区转一次手，这不是白白多了一次拷贝操作吗？
对IO函数有了解的童鞋肯定知道，在IO函数的背后有一个缓冲区 `buffer` ，我们平常的读和写操作并不是直接和底层硬件设备打交道，而是通过一块叫缓冲区的内存区域缓存数据来间接读写。我们知道，和CPU、高速缓存、内存比，磁盘、网卡这些设备属于慢速设备，交换一次数据要花很多时间，同时会消耗总线传输带宽，所以我们要尽量降低和这些设备打交道的频率，而使用缓冲区中转数据就是为了这个目的。
引用参考文献中的话：
> Using the intermediate buffer on the read side allows the kernel buffer to act as a "readahead cache" when the application hasn't asked for as much data as the kernel buffer holds. This significantly improves performance when the requested data amount is less than the kernel buffer size. The intermediate buffer on the write side allows the write to complete asynchronously.

大意是说，在读一侧的中间缓冲区可以作为预读缓存显著提高当请求数据大小小于内核缓冲区大小时的读性能，在写一侧的中间缓冲区可以允许写操作异步完成。
不过，当读请求数据的大小大于内核缓冲区时这个策略本身会变成一个性能瓶颈，数据在到达应用程序前会在磁盘、内核缓冲区、用户缓冲区之间反复多次拷贝。
让我们重新思考下上面的过程，会发现第二次和第三次的拷贝其实是不必要的，我们为什么不直接从读缓冲区将数据传输到socket缓冲区呢？实际上这就是 `transferTo()` 所做的。
```
public void transferTo(long position, long count, WritableByteChannel target);
```
`transferTo()` 方法将数据从一个文件channel传输到一个可写channel。在内部它依赖于操作系统对 `Zero-copy` 的支持，在UNIX/Linux系统上， `transferTo()` 实际会调用 `sendfile()` 这个系统函数，将数据从一个文件描述符传输到另一个。
```
#include <sys/socket.h>
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```
![placeholder](https://raw.githubusercontent.com/CodingRookieH/blog-image/master/2018-10-27-zero-copy/4.gif)

![placeholder](https://raw.githubusercontent.com/CodingRookieH/blog-image/master/2018-10-27-zero-copy/5.gif)
可以看到我们将上下文切换已经从4次减少到2次，同时把数据拷贝从4次减少到3次（只有1次 `CPU` 参与，另外2次 `DMA` 引擎完成），那么我们可不可以把这唯一一次CPU参与的数据拷贝也省掉呢？
如果网卡支持 `gather operations` 内核就可以进一步减少数据拷贝。在 Linux kernels 2.4 及更新的版本，socket 描述符已经为适应这个需求做了变化。现在这个方法不仅减少了上下文切换，而且消除了CPU参与的数据拷贝。API接口是一样的，但是实质已经发生了变化：
![placeholder](https://raw.githubusercontent.com/CodingRookieH/blog-image/master/2018-10-27-zero-copy/6.gif)
1. `transferTo()` 方法引起 `DMA` 引擎将文件内容拷贝到内核缓冲区。
2. 没有数据从内核缓冲区拷贝到socket缓冲区，只有携带位置和长度信息的描述符被追加到socket缓冲区上， `DMA` 引擎直接将内核缓冲区的数据传递到协议引擎，全程无需CPU拷贝数据。
到这里大家对 `transferTo()` 实现 `Zero-copy` 的原理应该很清楚了吧。

### 总结
其实普通RPC中间件会使用第一种零拷贝多一些，因为大多数JAVA中间件基于`Netty`，而`Netty`又基于NIO，使用堆外内存的方式减少拷贝次数也无可厚非。
对于Kafka这种基于文件系统的中间件（Kafka的消息最终会落到磁盘文件中），使用的是第三种零拷贝多一些，毕竟是基于文件数据的传输。

参考 **[Efficient data transfer through zero copy](https://www.ibm.com/developerworks/library/j-zerocopy/)**