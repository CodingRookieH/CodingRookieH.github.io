---
layout: post
title: gRPC网络模型
comments: true
categories:
  - GRPC从入门到放弃
---

## gRPC网络模型
gRPC 一开始由 google 开发，是一款语言中立、平台中立、开源的远程过程调用(RPC)系统。其内部使用Netty作为网络架构，但是Netty的使用姿势有千千万万种，究竟gRPC是如何与Netty进行融合，并且处理通信请求的，本篇博客会讲解讲解。

**系列目录**：
- [gRPC网络模型](https://codingrookieh.github.io/grpc%E4%BB%8E%E5%85%A5%E9%97%A8%E5%88%B0%E6%94%BE%E5%BC%83/2018/09/02/grpc-netty-analysis/)
- [Channel、Connection、Stream的那些事（基于Netty)](https://codingrookieh.github.io/grpc%E4%BB%8E%E5%85%A5%E9%97%A8%E5%88%B0%E6%94%BE%E5%BC%83/2018/09/13/grpc-channel-connection-stream/)
- 待续

### NettyServer的构造
gRPC的Server是通过NettyServer构造的，首先我们看看构造函数：
```
    NettyServer(SocketAddress address, Class<? extends ServerChannel> channelType, Map<ChannelOption<?>, ?> channelOptions, @Nullable EventLoopGroup bossGroup, @Nullable EventLoopGroup workerGroup, ProtocolNegotiator protocolNegotiator, List<Factory> streamTracerFactories, io.grpc.internal.TransportTracer.Factory transportTracerFactory, int maxStreamsPerConnection, int flowControlWindow, int maxMessageSize, int maxHeaderListSize, long keepAliveTimeInNanos, long keepAliveTimeoutInNanos, long maxConnectionIdleInNanos, long maxConnectionAgeInNanos, long maxConnectionAgeGraceInNanos, boolean permitKeepAliveWithoutCalls, long permitKeepAliveTimeInNanos, Channelz channelz) {
        ...
        this.bossGroup = bossGroup;
        this.workerGroup = workerGroup;
        ...
        this.usingSharedBossGroup = bossGroup == null;
        this.usingSharedWorkerGroup = workerGroup == null;
        ...
    }
```
我们可以看到，为什么```bossGroup```和```workGroup```可以为空呢？为空时，网络模型是如何构造呢？
别着急，我们在```start()```函数里还是看到，最终还是分配了。
```
    private void allocateSharedGroups() {
        if (this.bossGroup == null) {
            this.bossGroup = (EventLoopGroup)SharedResourceHolder.get(Utils.DEFAULT_BOSS_EVENT_LOOP_GROUP);
        }

        if (this.workerGroup == null) {
            this.workerGroup = (EventLoopGroup)SharedResourceHolder.get(Utils.DEFAULT_WORKER_EVENT_LOOP_GROUP);
        }

    }
    ############################## Utils中的代码 ######################################
    static {
        ...
        DEFAULT_BOSS_EVENT_LOOP_GROUP = new Utils.DefaultEventLoopGroupResource(1, "grpc-default-boss-ELG");
        DEFAULT_WORKER_EVENT_LOOP_GROUP = new Utils.DefaultEventLoopGroupResource(0, "grpc-default-worker-ELG");
        ...
    }
```
是不是有点熟悉，如果使用过NIO线程处理过回调的同学，应该知道，线程名都是```grpc-default-worker-ELG```，并且为0时，还是默认还是可用CPU核数*2。
**[FLAG1]** 但是细心的同学会发现，不对啊，在构造NettyServer的时候，我明明是可以指定Executor的，那个线程池是做什么用的呢？这里先立个flag，一会细讲。
```
    public void start(ServerListener serverListener) throws IOException {
        this.listener = (ServerListener)Preconditions.checkNotNull(serverListener, "serverListener");
        this.allocateSharedGroups();
        ServerBootstrap b = new ServerBootstrap();
        b.group(this.bossGroup, this.workerGroup);
        ...
        //开始初始化channel的处理链
        b.childHandler(new ChannelInitializer<Channel>() {
            public void initChannel(Channel ch) throws Exception {
                ChannelPromise channelDone = ch.newPromise();
                long maxConnectionAgeInNanos = NettyServer.this.maxConnectionAgeInNanos;
                if (maxConnectionAgeInNanos != 9223372036854775807L) {
                    maxConnectionAgeInNanos = (long)((0.9D + Math.random() * 0.2D) * (double)maxConnectionAgeInNanos);
                }

                //具体逻辑都放在 NettyServerTransport 这一层
                NettyServerTransport transport = new NettyServerTransport(ch, channelDone, NettyServer.this.protocolNegotiator, NettyServer.this.streamTracerFactories, NettyServer.this.transportTracerFactory.create(), NettyServer.this.maxStreamsPerConnection, NettyServer.this.flowControlWindow, NettyServer.this.maxMessageSize, NettyServer.this.maxHeaderListSize, NettyServer.this.keepAliveTimeInNanos, NettyServer.this.keepAliveTimeoutInNanos, NettyServer.this.maxConnectionIdleInNanos, maxConnectionAgeInNanos, NettyServer.this.maxConnectionAgeGraceInNanos, NettyServer.this.permitKeepAliveWithoutCalls, NettyServer.this.permitKeepAliveTimeInNanos);
                ...
            }
        });
        ChannelFuture future = b.bind(this.address);
        ...
    }
```
看到这里，大家应该心知肚明了，看来GRPC中最终的处理逻辑，应该都由```NettyServerTransport```完成。那么这个类里边又做了什么呢?
我们直接看他的```start()```函数
```
    public void start(ServerTransportListener listener) {
        Preconditions.checkState(this.listener == null, "Handler already registered");
        this.listener = listener;
        //创建真正处理请求的handler
        this.grpcHandler = this.createHandler(listener, this.channelUnused);
        NettyHandlerSettings.setAutoWindow(this.grpcHandler);

        final class TerminationNotifier implements ChannelFutureListener {
            boolean done;

            TerminationNotifier() {
            }

            public void operationComplete(ChannelFuture future) throws Exception {
                if (!this.done) {
                    this.done = true;
                    NettyServerTransport.this.notifyTerminated(NettyServerTransport.this.grpcHandler.connectionError());
                }

            }
        }

        ChannelFutureListener terminationNotifier = new TerminationNotifier();
        this.channelUnused.addListener(terminationNotifier);
        this.channel.closeFuture().addListener(terminationNotifier);
        //包装成ChannelHandler
        ChannelHandler negotiationHandler = this.protocolNegotiator.newHandler(this.grpcHandler);
        this.channel.pipeline().addLast(new ChannelHandler[]{negotiationHandler});
    }
```
看来比想象的复杂，有两个handler，那么为什么要分两个呢，先看第一个handler：
第一个handler通过大家追踪代码，应该很容易看出来是一个```NettyServerHandler```，在其构造函数中，我们看到了我们之前谈到的粘包拆包的解决方式：
```
Http2ConnectionEncoder encoder = new DefaultHttp2ConnectionEncoder(connection, frameWriter);
Http2ConnectionDecoder decoder = new DefaultHttp2ConnectionDecoder(connection, encoder, frameReader);
```
而这两种方式也是原生Netty支持的。
那如果简单的只是Netty原生的encoder和decoder，难道HTTP2已经用了Google的ProtoBuffer了？
显然不是，在```NettyServerHandler```构造的时候，我们可以看到有一行很隐蔽的代码：
```this.decoder().frameListener(new NettyServerHandler.FrameListener());```
原来如此，在HTTP2解包后，看来是用了```NettyServerHandler.FrameListener```进行了处理，OK，读到了这里，到底是不是这样呢？我们可以看到```NettyServerHandler.FrameListener``` 中确实实现了```onHeadersRead```和```onDataRead```，而```onDataRead```最终就会调用到请求处理的类。
```
##################ServerCallImpl################################################
    @SuppressWarnings("Finally") // The code avoids suppressing the exception thrown from try
    @Override
    public void messagesAvailable(final MessageProducer producer) {
      if (call.cancelled) {
        GrpcUtil.closeQuietly(producer);
        return;
      }

      InputStream message;
      try {
        while ((message = producer.next()) != null) {
          try {
            listener.onMessage(call.method.parseRequest(message));
          } catch (Throwable t) {
            GrpcUtil.closeQuietly(message);
            throw t;
          }
          message.close();
        }
      } catch (Throwable t) {
        GrpcUtil.closeQuietly(producer);
        MoreThrowables.throwIfUnchecked(t);
        throw new RuntimeException(t);
      }
    }
```

### NIO线程与用户线程的切换
OK，解读到这里，我们回到前边说的第一个 **[Flag1]** 处，为什么NettyServer在构造的时候，会传递一个线程池进去呢？
我们可以看看最终响应请求的类：
```
################################# ServerImpl ##########################################
    @Override
    public void streamCreated(
        final ServerStream stream, final String methodName, final Metadata headers) {
      ...
      final Context.CancellableContext context = createContext(stream, headers, statsTraceCtx);
      final Executor wrappedExecutor;
      // This is a performance optimization that avoids the synchronization and queuing overhead
      // that comes with SerializingExecutor.
      if (executor == directExecutor()) {
        wrappedExecutor = new SerializeReentrantCallsDirectExecutor();
      } else {
        wrappedExecutor = new SerializingExecutor(executor);
      }

      final JumpToApplicationThreadServerStreamListener jumpListener
          = new JumpToApplicationThreadServerStreamListener(
              wrappedExecutor, executor, stream, context);
      stream.setListener(jumpListener);
      // Run in wrappedExecutor so jumpListener.setListener() is called before any callbacks
      // are delivered, including any errors. Callbacks can still be triggered, but they will be
      // queued.

      ...
      wrappedExecutor.execute(new StreamCreated());
    }
```
其中```JumpToApplicationThreadServerStreamListener```就是NIO线程到用户线程（即我们外部传入的Executor）的转换过程，NIO线程会把执行任务扔给用户线程，完成线程的转换。

### GRPC网络模型
首先我们来看一下 Reactor 的线程模型.
Reactor 的线程模型有三种:
- 单线程模型
- 多线程模型（gRPC使用的Reactor模型）
- 主从多线程模型
首先来看一下 **单线程模型**:

![placeholder](https://raw.githubusercontent.com/CodingRookieH/blog-image/master/1491872898-58200c24e15eb_articlex.png)

所谓单线程, 即 acceptor 处理和 handler 处理都在一个线程中处理. 这个模型的坏处显而易见: 当其中某个 handler 阻塞时, 会导致其他所有的 client 的 handler 都得不到执行, 并且更严重的是, handler 的阻塞也会导致整个服务不能接收新的 client 请求(因为 acceptor 也被阻塞了)。 因为有这么多的缺陷, 因此单线程Reactor 模型用的比较少。
那么什么是 **多线程模型** 呢? Reactor 的多线程模型与单线程模型的区别就是 acceptor 是一个单独的线程处理, 并且有一组特定的 NIO 线程来负责各个客户端连接的 IO 操作. Reactor 多线程模型如下：

![placeholder](https://raw.githubusercontent.com/CodingRookieH/blog-image/master/226082138-58200c365ec44_articlex.png)

Reactor 多线程模型 有如下特点:
- 有专门一个线程, 即 Acceptor 线程用于监听客户端的TCP连接请求。
- 客户端连接的 IO 操作都是由一个特定的 NIO 线程池负责. 每个客户端连接都与一个特定的 NIO 线程绑定, 因此在这个客户端连接中的所有 IO 操作都是在同一个线程中完成的。
- 客户端连接有很多, 但是 NIO 线程数是比较少的, 因此一个 NIO 线程可以同时绑定到多个客户端连接中。 

接下来我们再来看一下 Reactor 的主从多线程模型.
一般情况下, Reactor 的多线程模式已经可以很好的工作了, 但是我们考虑一下如下情况: 如果我们的服务器需要同时处理大量的客户端连接请求或我们需要在客户端连接时, 进行一些权限的检查, 那么单线程的 Acceptor 很有可能就处理不过来, 造成了大量的客户端不能连接到服务器.
Reactor 的主从多线程模型就是在这样的情况下提出来的, 它的特点是: 服务器端接收客户端的连接请求不再是一个线程, 而是由一个独立的线程池组成. 它的线程模型如下:

![placeholder](https://raw.githubusercontent.com/CodingRookieH/blog-image/master/3149276718-58200c2f1e15d_articlex.png)

可以看到, Reactor 的主从多线程模型和 Reactor 多线程模型很类似, 只不过 Reactor 的主从多线程模型的 acceptor 使用了线程池来处理大量的客户端请求。

最终，我们来看一下整个gRPC的网络模型
- Server端：
![placeholder](https://raw.githubusercontent.com/CodingRookieH/blog-image/master/2018-09-09-grpc-netty/2018-09-09-grpc-netty1.png)

- Client端：
![placeholder](https://raw.githubusercontent.com/CodingRookieH/blog-image/master/2018-09-09-grpc-netty/2018-09-09-grpc-netty2.png)


