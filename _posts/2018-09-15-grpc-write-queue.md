## 承上启下的WriteQueue

经过前边两章，大家应该了解了gRPC中的网络相关的知识，但是真的要通讯起来，网络包、通讯流程、数据结构又是如何的呢？直接使用HTTP2进行通讯不就完事了？远没有那么简单，本文会从gRPC中的`WriteQueue`出发，介绍一下gRPC中请求的封装结构，以及Frame的概念。下文所有的Frame，也叫帧，各位看官注意一下。

### HTTP2中的Frame格式

所有的帧都以一个9字节的报头开始, 后接变长的载荷:

```
+-----------------------------------------------+
 |                 Length (24)                   |
 +---------------+---------------+---------------+
 |   Type (8)    |   Flags (8)   |
 +-+-------------+---------------+-------------------------------+
 |R|                 Stream Identifier (31)                      |
 +=+=============================================================+
 |                   Frame Payload (0...)                      ...
 +---------------------------------------------------------------+
```

报头部分的字段定义如下:

- `Length`: 载荷的长度, 无符号24位整型. **对于发送值大于2^14 (长度大于16384字节)的载荷, 只有在接收方设置SETTINGS_MAX_FRAME_SIZE为更大的值时才被允许**。

  注：帧的报头9字节不算在length里。

- `Type`： 8位的值表示帧类型, 决定了帧的格式和语义。协议实现上必须忽略任何未知类型的帧。

- `Flags`： 为Type保留的bool标识, 大小是8位。对确定的帧类型赋予特定的语义。否则发送时必须忽略(设置为0x0)。

- `R`: 1位的保留字段，尚未定义语义。发送和接收必须忽略(0x0)。

- `Stream Identifier`：31位无符号整型的流标示符。其中0x0作为保留值， 表示与连接相关的frames作为一个整体而不是一个单独的流。

具体的`Frame`定义格式大家可以参考**[RFC7540](https://httpwg.org/specs/rfc7540.html#FrameTypes)**

### 什么是WriteQueue

为什么需要从WriteQueue开始讲起，因为所有的gRPC的数据都是通过他写入`Channel`的，`Channel`不太了解的同学还是看一下`Netty`的源码。

既然叫做`Queue`，储存了什么呢？

```
  private final Channel channel;
  private final Queue<QueuedCommand> queue;
  private final AtomicBoolean scheduled = new AtomicBoolean();
```

看来是一个个的`QueuedCommand`，现在已有的类型有：

- `SendGrpcFrameCommand`真正的gRPC请求数据，你可以理解成类似Data的东西。
- `CreateStreamCommand`在remote端生成Stream的请求，由`NettyClientStream`调用生成。
- `GracefulCloseCommand`正常关闭流。
- `ForcefulCloseCommand`强行关闭流。
- `CancelServerStreamCommand`取消Server端的流。
- `CancelClientStreamCommand`取消Client端的流。
- `SendPingCommand`心跳。
- `SendResponseHeadersCommand`创建回复Header。

比较重要的就是`GrpcFrameCommand`，我们可以看到在他的`run()`方法中，开始了`Channel`的交互：

```
  @Override
  public final void run(Channel channel) {
    channel.write(this, promise);
  }
```

那什么时候会调用呢？

我们来看`WriteQueue`的`scheduleFlush()`方法：

```
  /**
   * Schedule a flush on the channel.
   */
  void scheduleFlush() {
    if (scheduled.compareAndSet(false, true)) {
      // Add the queue to the tail of the event loop so writes will be executed immediately
      // inside the event loop. Note DO NOT do channel.write outside the event loop as
      // it will not wake up immediately without a flush.
      channel.eventLoop().execute(later);
    }
  }

```

看来是串行的在往`eventLoop()`中提交刷新，为什么说是串行的，因为大家会根据源码看看`later`，其中最终会调用到`WriteQueue`的`flush()`方法，所以，流程就变成。

1. `WriteQueue`中的`enqueue()`方法加入队列，并决定是否要往`Channel中写`。
2. 如果要写，那么就会调用`scheduleFlush()`。
3. 回调到`SendGrpcFrameCommand`开始写`channel`。
4. 写完`Channel`刷新缓冲区。

### gRPC中的Frame

讲完`WriteQueue`，相信大家应该知道了，看来我们rpc通信的数据都是放在`SendGrpcFrameCommand`里啊。

是的没错，那么具体放了些什么呢?

这就回到了我们第一次博客说的`NettyServerHandler`和`NettyClientHandler`了。/

