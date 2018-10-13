---
layout: post
title: gRPC中的FRAME
comments: true
categories:
  - GRPC从入门到放弃
---

## gRPC中的FRAME
经过前边两章，大家应该了解了gRPC中的网络相关的知识，但是真的要通讯起来，网络包、通讯流程、数据结构又是如何的呢？直接使用HTTP2进行通讯不就完事了？远没有那么简单，本文会从gRPC中的`WriteQueue`出发，介绍一下gRPC中请求的封装结构，以及Frame的概念。下文所有的Frame，也叫帧，各位看官注意一下。

**系列目录**：
- [gRPC网络模型](https://codingrookieh.github.io/grpc%E4%BB%8E%E5%85%A5%E9%97%A8%E5%88%B0%E6%94%BE%E5%BC%83/2018/09/02/grpc-netty-analysis/)
- [Channel、Connection、Htt2Stream、Stream的那些事（基于Netty)](https://codingrookieh.github.io/grpc%E4%BB%8E%E5%85%A5%E9%97%A8%E5%88%B0%E6%94%BE%E5%BC%83/2018/09/13/grpc-channel-connection-stream/)
- [gRPC中的FRAME](https://codingrookieh.github.io/grpc%E4%BB%8E%E5%85%A5%E9%97%A8%E5%88%B0%E6%94%BE%E5%BC%83/2018/09/15/grpc-write-queue/)
- [转换的艺术：MessageFrame、MessageDeframer](https://codingrookieh.github.io/grpc%E4%BB%8E%E5%85%A5%E9%97%A8%E5%88%B0%E6%94%BE%E5%BC%83/2018/09/17/grpc-message-framer/)
- 待续

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

### 数据写入中转站:WriteQueue
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
- `SendResponseHeadersCommand`创建回复Header，一般用于Server处理完请求回写给客户端。
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
在`AbstractServerStream`中我们找到了答案，原来在`NettyServerStream.Sink`和`NettyClientStream.Sink`中，会进行数据的写入。这里我们注意到：
```
new SendGrpcFrameCommand(transportState(), bytebuf, false)  //NettyServerStream
new new SendGrpcFrameCommand(transportState(), bytebuf, endOfStream //NettyClientStram
```
两者还是有区别的。看来，Server端写出去的`Frame`是不会为`Stream`中最后一个`Frame`的，只有client端会控制是否是最后一个`Frame`。
**真相大白，原来请求经过序列化后(`MessageFramer`)，就会封装成`WritableBuffer`投递给`NettyServerStream.Sink`和`NettyClientStream.Sink`，投递后，再由`Sink`加入到`WriteQueue`，最终由`WriteQueue`写进`Channel`。**
综上，其实`WriteQueue`是在网络层与真正的服务实现类之间写信息的媒介，无论Client还是Server，最终都是通过`WriteQueue`开始转到`Netty`网络层的。
那gRPC中的`Frame`到底是怎样的呢？
首先我们先讲讲gRPC中的`Stream`，
`* A single stream of communication between two end-points within a transport.`
这里很显然`Stream`被视为两个物理机通信的数据结构。
然而，在RFC中，我们又找到一句话：
```
The order in which frames are sent on a stream is significant. Recipients process frames in the order they are received. In particular, the order of HEADERS and DATA frames is semantically significant.
```
RFC中的`Stream`说的是`Http2Stream`，大致的意思是，`Stream`中的`Frame`发送和处理的顺序需要某种机制去保证一致，这里大家理解一下，如果A端发送的数据包为1、2，而B端接收的数据包为1，2，但是处理后回写成2的返回、1的返回，那通讯不就乱了？是的，TCP是不会给你做请求对应这种事情的，管你IO就已经很不错了。
那这么说，gRPC中的`Frame`如何保证数据包对应的呢？
这里楼主说实话，想了很久，查了很久，最后找到了`SendResponseHeadersCommand`，这是`Server`在处理完请求后回写给客户端的数据载体，我们翻到`NettuServerHandler`中去。
```
  private void sendResponseHeaders(ChannelHandlerContext ctx, SendResponseHeadersCommand cmd,
      ChannelPromise promise) throws Http2Exception {
    int streamId = cmd.stream().id();
    Http2Stream stream = connection().stream(streamId);
    if (stream == null) {
      resetStream(ctx, streamId, Http2Error.CANCEL.code(), promise);
      return;
    }
    if (cmd.endOfStream()) {
      //如果这个数据是Stream中最后一条，就在数据发送完后关闭Stream
      closeStreamWhenDone(promise, streamId);
    }
    encoder().writeHeaders(ctx, streamId, cmd.headers(), 0, cmd.endOfStream(), promise);
  }
  
  ...
  private void sendGrpcFrame(ChannelHandlerContext ctx, SendGrpcFrameCommand cmd,
      ChannelPromise promise) throws Http2Exception {
    if (cmd.endStream()) {
      //如果这个数据是Stream中最后一条，就在数据发送完后关闭Stream
      closeStreamWhenDone(promise, cmd.streamId());
    }
    // Call the base class to write the HTTP/2 DATA frame.
    encoder().writeData(ctx, cmd.streamId(), cmd.content(), 0, cmd.endStream(), promise);
  }
```
啊，原来如此，看来gRPC中一个`Stream`只完成一次响应的请求，果然是`within a transport`，原来Client端在发送了请求后，就已经将`Stream`置成`half-close`状态了，我们看下**`ClientCalls`**类： 
```
    private static <ReqT, RespT> void asyncUnaryRequestCall(ClientCall<ReqT, RespT> call, ReqT param, Listener<RespT> responseListener, boolean streamingResponse) {
        startCall(call, responseListener, streamingResponse); //想也知道，先发Header，中间处理       可不能异步哦。
        try {
            call.sendMessage(param); //发送请求
            call.halfClose();  //半关闭
        } catch (RuntimeException var5) {
            throw cancelThrow(call, var5);
        } catch (Error var6) {
            throw cancelThrow(call, var6);
        }
    }
```
那真正的`close()`在哪里呢？
看来还是在大Server端，`ServerCallImpl`中：
```
  @Override
  public void close(Status status, Metadata trailers) {
    checkState(!closeCalled, "call already closed");
    try {
      closeCalled = true;

      if (status.isOk() && method.getType().serverSendsOneMessage() && !messageSent) {
        internalClose(Status.INTERNAL.withDescription(MISSING_RESPONSE));
        return;
      }
      stream.close(status, trailers);
    } finally {
      serverCallTracer.reportCallEnded(status.isOk());
    }
  }
```
怪不得，我们每次Server接收到响应后，要调用一下`onComplete()`方法，因为要将流关闭。那`onNext()`方法呢？`onNext()`实际上就是回写返回给调用方了。

### 总结
看来gRPC基于`Http2`，每次完整的请求周期，只对应一个`Stream`，在`Stream`上，Client和Server创建`Stream`，发送数据，处理，返回，关闭流。这么看来，使用完全异步的`Netty`也是明智之举，毕竟有StreamId，请求的处理顺序控制的很好。

本文为作者原创，转载请注明出处 。**邮箱：568718043@qq.com**
