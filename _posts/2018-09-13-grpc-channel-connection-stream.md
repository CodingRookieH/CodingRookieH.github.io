---
layout: post
title: Channel、Connection、Stream的那些事（基于Netty）
comments: true
categories:
  - GRPC从入门到放弃
---

## Channel、Connection、Stream的那些事（基于Netty）
看过了第一篇gRPC的网络模型，相信大家已经对gRPC的网络模型有了一定的了解，今天博主会结合大名鼎鼎的`Netty`，详细掰掰扯和数据交互密不可分的这些类，他们的区别和联系。

**系列目录**：
- [gRPC网络模型](https://codingrookieh.github.io/grpc%E4%BB%8E%E5%85%A5%E9%97%A8%E5%88%B0%E6%94%BE%E5%BC%83/2018/09/02/grpc-netty-analysis/)
- [Channel、Connection、Stream的那些事（基于Netty)](https://codingrookieh.github.io/grpc%E4%BB%8E%E5%85%A5%E9%97%A8%E5%88%B0%E6%94%BE%E5%BC%83/2018/09/13/grpc-channel-connection-stream/)
- [gRPC中的FRAME](https://codingrookieh.github.io/grpc%E4%BB%8E%E5%85%A5%E9%97%A8%E5%88%B0%E6%94%BE%E5%BC%83/2018/09/15/grpc-write-queue/)
- 待续

### Channel
`Channel`是JAVA针对NIO提出的一种类似于InputStream、OutputStream的概念。
```
   A channel represents an open connection to an entity such as a hardware
 * device, a file, a network socket, or a program component that is capable of
 * performing one or more distinct I/O operations, for example reading or
 * writing.
```
`Channel `类型有:
- FileChannel, 文件操作
- DatagramChannel, UDP 操作
- SocketChannel, TCP 操作
- ServerSocketChannel, TCP 操作, 使用在服务器端
这些通道涵盖了 UDP 和 TCP网络 IO以及文件 IO。

### Http2Connection
这里的Connection和连接池的连接是有区别的，Http2的连接默认使用的是`DefaultHttp2Connection`，这个类主要是对两个`EndPoint`进行连接管理（注意，这里的`EndPoint`可以视为两台通讯设备），本文中的另一主角`Http2Stream`就是被管理在这个类中的。
那么这个类在做些什么事情呢?
首先我们看看类里有什么:
```
        final IntObjectMap<Http2Stream> streamMap = new IntObjectHashMap<Http2Stream>();
        final ConnectionStream connectionStream = new ConnectionStream();
        final DefaultEndpoint<Http2LocalFlowController> localEndpoint;
        final DefaultEndpoint<Http2RemoteFlowController> remoteEndpoint;
        ...
        final List<Listener> listeners = new ArrayList<Listener>(4);
        final ActiveStreams activeStreams;
        Promise<Void> closePromise;
```
比较重要的应该就是上述的这些数据，其中`ConnectionStream`其实是初始化`Http2Connection`时为了区别其他`Stream`，添加的一个自身标识。
初次之外，我们看到，`Connection`管理`Stream`的应该就是**connectionStream**这个Map了。
那为什么还需要**activeStreams**呢，这是个很好的问题，博主追了下代码，从`DefaultEndpoint`中找到了答案，原来在`EndPoint`创建`Stream`的时候：
```
        @Override
        public DefaultStream createStream(int streamId, boolean halfClosed) throws Http2Exception {
            State state = activeState(streamId, IDLE, isLocal(), halfClosed);

            checkNewStreamAllowed(streamId, state);

            // Create and initialize the stream.
            DefaultStream stream = new DefaultStream(streamId, state);

            incrementExpectedStreamId(streamId);
            //放在streamMap中去
            addStream(stream);
            //放入activeStreams中去
            stream.activate();
            return stream;
        }
```
从这里就可以看出，`Stream`的**Add**和Stream的**Active**是两个不同的事件，除此之外呢？看来是会有`Stream`，是不属于**activeStreams**的行列的。什么`Stream`呢？发`Push promise`使用的`Stream`。 
```
        @Override
        public DefaultStream reservePushStream(int streamId, Http2Stream parent) throws Http2Exception {
            ...
            DefaultStream stream = new DefaultStream(streamId, state);

            incrementExpectedStreamId(streamId);

            // Register the stream.
            addStream(stream);
            return stream;
        }
```

### Http2Stream
终于，我们要到Stream了。
在一个HTTP/2的连接中, 流是服务器与客户端之间用于帧交换的一个独立双向序列. 流有几个重要的特点:
- 一个HTTP/2连接可以包含多个并发的流, 各个端点从多个流中交换frame
- 流可以被客户端或服务器单方面建立, 使用或共享
- 流也可以被任意一方关闭
- frames在一个流上的发送顺序很重要. 接收方将按照他们的接收顺序处理这些frame. 特别是`HEADERS`和`DATA` frame的顺序, 在协议的语义上显得尤为重要.
- 流用一个整数(流标识符)标记. 端点初始化流的时候就为其分配了标识符. 
拷贝RFC中HTT2中关于流的状态图如下：
```
                             +--------+
                     send PP |        | recv PP
                    ,--------|  idle  |--------.
                   /         |        |         \
                  v          +--------+          v
           +----------+          |           +----------+
           |          |          | send H /  |          |
    ,------| reserved |          | recv H    | reserved |------.
    |      | (local)  |          |           | (remote) |      |
    |      +----------+          v           +----------+      |
    |          |             +--------+             |          |
    |          |     recv ES |        | send ES     |          |
    |   send H |     ,-------|  open  |-------.     | recv H   |
    |          |    /        |        |        \    |          |
    |          v   v         +--------+         v   v          |
    |      +----------+          |           +----------+      |
    |      |   half   |          |           |   half   |      |
    |      |  closed  |          | send R /  |  closed  |      |
    |      | (remote) |          | recv R    | (local)  |      |
    |      +----------+          |           +----------+      |
    |           |                |                 |           |
    |           | send ES /      |       recv ES / |           |
    |           | send R /       v        send R / |           |
    |           | recv R     +--------+   recv R   |           |
    | send R /  `----------->|        |<-----------'  send R / |
    | recv R                 | closed |               recv R   |
    `----------------------->|        |<----------------------'
                             +--------+

       send:   endpoint sends this frame
       recv:   endpoint receives this frame

       H:  HEADERS frame (with implied CONTINUATIONs)
       PP: PUSH_PROMISE frame (with implied CONTINUATIONs)
       ES: END_STREAM flag
       R:  RST_STREAM frame
```
该图只展示了流的状态转换以及frame和标记如何对转换产生影响. 这方面。`CONTINUATION`frames不会导致状态的转换， 他们只是跟在`HEADERS`或`PUSH_PROMISE` frame后面的有效组成部分。
状态转换的用途, 对于设置了`END_STREAM`标记的frame来说，`END_STREAM`被当做一个分开的事件处理. 设置了`END_STREAM`标记的`HEADERS` frame会导致两次状态转换。
在传输过程中， 每个端点对流状态的主观认识可能不同。这些终端不会协商流的创建, 都是由终端独立创建的. 端点的流状态不同会带来负面影响: 在发送了`RST_STREAM`之后流处于关闭状态，而frame可能在流关闭之后才到达。
流有如下状态： 
- `idle` 所有流最初状态都是`idle`。
  下面描述了流从`idle`状态到其它状态的几种可能转换：
  - 发送或接收到一个`HEADERS`frame会使流状态变换`open`。 流标识符的选择参上图里的描述. 收到相同的`HEADERS`frame会导致流立即变为`half-close`状态。
  - (Sending a PUSH_PROMISE frame on another stream reserves the idle stream that is identified for later use.)在另一个流上发送一个`PUSH_PROMISE`frame 被标识为以后使用。预留流的状态对应转换到`reserved (local)`。
  - (Receiving a PUSH_PROMISE frame on another stream reserves an idle stream that is identified for later use.)在另一个流上接收一个`PUSH_PROMISE`frame 被标识为以后使用。预留流的状态对应转换到`reserved (remote)`。
  - 注意`PUSH_PROMISE`frame并不在idle流上发送，只是promised流的ID字段引用了新的reserved流。
    在`idle`状态接收到任何非`HEADERS`或`PUSH_PROMISE`frame必须视为连接错误, 错误类型为`PROTOCOL_ERROR`。

- `reserved (local)` 处于这种状态的流表示它已经发送了一个`PUSH_PROMISE`frame并成为promised流。`PUSH_PROMISE`frame通过关联一个由远程对等点初始化的流来转换idle流到reserved流。
  处于这个状态的流, 只有下面的几种可能状态转换：
  - 端点发送一个`HEADERS`frame， 流进入`half-closed (remote)`状态。
  - 任何一个端点发送一个`RST_STREAM`frame, 流变成`closed`状态. 这将释放一个流保留的资源.
    端点不准发送除`HEADERS`, `RST_STREAM`或`PRIORITY`之外任何类型的frame。
  这一状态可能收到`PRIORITY`或`WINDOW_UPDATE`frame。 除了`RST_STREAM`， `PRIORITY`以及`WINDOW_UPDATE`frame之外，收到其他类型的frame必须视为`PROTOCOL_EROR`类型的连接错误。

- `reserved (remote)` 如果一个流已被远程对等点保留, 状态就会变成`reserved(remote)`。
  可能的转换如下:
  - 收到一个`HEADERS` frame导致状态变为`half-close(local)`。
  - 任何端点发送一个`RST_STREAM`frame会导致状态变成`closed`， 并释放流保留的资源。
  端点可以发送一个`PRIORITY` frame以重新确定reserved流的优先级次序. 不允许发送除`RST_STREAM`, `WINDOW_UPDATE`或`PRIORITY`之外的frame.
  在一个流上拿到非`HEADERS`	,`RST_STREAM`或`PRIORITY`的frame必须视为`PROTOCOL_EROR`类型的连接错误。

- `open` 任何一对等方可以使用`open`状态的流发送任意类型的frame. 这一状态下, 发送方会监视给出的流级别和流控范围.
  在任意一方发送设置了`END_STREAM`标记的frame后, 流状态会变为`half-closed`的其中一个状态: 如果一方发送了该frame, 其流变为`half-closed(local)`；如果一方收到该frame, 流变为`half-closed(remote)`。
  在这个状态发送`RST_STREAM` frame可以使状态立即变成`closed`。

- `half-closed (local)` 处于这个状态的流不能发送除`WINDOW_UPDATE`，`PRIORITY`以及`RST_STREAM`之外的frame。
  收到一个标记了`END_STREAM`的frame或者发送一个`RST_STREAM` frame， 都会使状态变成closed。
  端点允许接收任意类型的frame。 便于后续接收用于流控的frame， 使用`WINDOW_UPDATE` frame提供流控credit很有必要. 接收方可以选择忽略`WINDWO_UPDATE` frame, (which might arrive for a short period after a frame bearing the END_STREAM flag is sent.)
  收到的`PRIORITY` frame用于重定流的优先级次序(依据流的标记而定)。

- `half-closed (remote)` 处于这个状态的流，对端不再用来发送frame了。 并且端点也无需继续维护接收方流控窗口。
  如果端点收到额外的frame,并且不是`WINDOW_UPDATE`，`PRIORITY`或`RST_STREAM`，那么必须响应一个类型为`STREAM_CLOSED`的流错误。
  这一状态下的流可以发送任意类型的frame. 端点仍会继续监视已知的流级别和流控范围.
  发送一个`END_STERAM`标记的frame或任意一个对等方发送了`RST_STREAM` frame都会使流变为`closed`。

- `closed` closed标识终止状态。
  在一个closed的流上不允许发送`PRIORITY`之外的其他frame. 端点在收到`RST_STREAM` frame后又收到非`PRIORITY`的frame的话， 一定被视为流错误对待(类型`STREAM_CLOSED`)。
  同样, 收到`END_STREAM`标记后又收到**非如下描述**的frame， 会触发一个连接错误(类型`STREAM_CLOSED`)：
  发送了包含`END_STREAM`标记的`DATA`或`HEADERS` frame后的一小段时间内，允许`WINDOW_UPDATE`或`RST_STREAM` frame被接收。 直到远程对等端收到并处理了`RST_STERAM`或包含`END_STREAM`标记的frame, 才可以发送这些类型的frame。 假如在发送了`END_STREAM`后已明显过了超时时间, 这时却再次收到frame， 尽管终端可以选择把这个frame当成`PROTOCOL_ERROR`类型的连接错误来处理, 但无论如何最终**必须**忽略这种情况下收到的`WINDOW_UPDATE`或`RST_STREAM` frame。
  `PRIORITY`帧可从`closed`流上发到优先级更高的流(取决于`closed`流)。终端应该处理`PRIORITY`帧, 尽管他们可能因为流已经从依赖树中移除而被忽略。
  如果是发送`RST_STREAM`帧的原因让状态转换到了`closed`，收到`RST_STREAM`的对等端这时可能已经发送了`RST_STREAM`或者入队等待发送中, 但是已经在流上传输的帧是不可以被撤销的. 这时, 终端必须忽略从`closed`的流上再取得的帧，如果这个`closed`流已经发送了`RST_STREAM`帧。终端也可以选择一个超时时间， 忽略在此之后到达的帧, 并一律视作错误。
  在发送了`RST_STREAM`之后收到的流控帧(比如DATA帧)也会被用于计算当前连接的流控窗口。(are counted toward the connection flow-control window.) 尽管这些帧有可能被忽略掉，但是因为他们在发送方收到`RST_STREAM`之前被发送了， 所以发送方仍有可能会根据这些帧计算流控窗口大小.
  终端发送了`RST_STREAM帧`之后可以再接收一个`PUSH_PROMISE`帧。`PUSH_PROMISE`帧会将流状态变为`reserved`即使相关的流已经被重置. 因此需要一个`RST_STREAM`帧去关闭不再需要的`promised`流。

本文档中没有给出更具体说明的地方, 对于收到的那些未在上述状态描述中明确认可的帧, 协议实现上应该视这种情况为一个类型为`PROTOCOL_ERROR`的连接错误，另外注意`PRIORITY`帧可以在流的任何一个状态被发送/接收。 忽略未知类型的帧。

### 三者的关系
其实通过上述介绍，读者们应该将他们的关系猜的八九不离十了，简单的讲就是：
`Http2Connection`管理`Http2Stream`，而`Http2Stream`只是一个`Stream`的状态管理，真正的数据传输还是要通过`Channel`，那`Channel`和`Http2Stream`的连接点在哪里呢？对于`Netty`来说，连接点就是`HttpConnectionHandler`，因此，对于一个`Channel`，就对应一个`Http2Connection`，一个`Http2Connection`对应多个`Http2Stream`。