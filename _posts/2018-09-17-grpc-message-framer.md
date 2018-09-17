---
layout: post
title: 转换的艺术：MessageFrame、MessageDeframer
comments: true
categories:
  - GRPC从入门到放弃
---

## 转换的艺术：MessageFrame、MessageDeframer
前边我们讲了gRPC通信的模型，以及`WriteQueue`还有Frame。哎？那我们最终看到的请求或者响应，都是已经序列化好的类啊？难道`Netty`还能帮我转换成字节不成？如果一次通信的Frame过大，要分两个Frame发，怎么整？本篇会来介绍处理这些问题的类，`MessageFrame`和`MessageDeframer`。

### MessageFrame
```java
Encodes gRPC messages to be delivered via the transport layer 
```
故名思义，将gRPC的请求变成`WritableBuffer`（这里要注意哦，只是把`InputStream`变成了`WritableBuffer`，真正的Proto的序列化还在外层），其实也就是`NettyWritableBuffer`，里边就会有我们熟悉的`ByteBuf`(`Netty`中通讯的数据单元)。
![placeholder](https://raw.githubusercontent.com/CodingRookieH/blog-image/master/2018-09-17-grpc-netty-message-frame/20180917-grpc-message-framer-1.png)
那`MessageFrame`里做了什么呢：
```
  @Override
  public void writePayload(InputStream message) {
      ...
      if (messageLength != 0 && compressed) {
        written = writeCompressed(message, messageLength);
      } else {
        written = writeUncompressed(message, messageLength);
      }
      ...
  }
```
看来写也要分压缩和非压缩，那我们看看非压缩的情况：
```
  private int writeUncompressed(InputStream message, int messageLength) throws IOException {
    if (messageLength != -1) {
      currentMessageWireSize = messageLength;
      return writeKnownLengthUncompressed(message, messageLength);
    }
    BufferChainOutputStream bufferChain = new BufferChainOutputStream();
    //将message写入bufferChain
    int written = writeToOutputStream(message, bufferChain);
    if (maxOutboundMessageSize >= 0 && written > maxOutboundMessageSize) {
      throw Status.RESOURCE_EXHAUSTED
          .withDescription(
              String.format("message too large %d > %d", written , maxOutboundMessageSize))
          .asRuntimeException();
    }
    //最终将bufferChain写入到sink
    writeBufferChain(bufferChain, false);
    return written;
  }
```
最终：
```
  private void writeBufferChain(BufferChainOutputStream bufferChain, boolean compressed) {
    ByteBuffer header = ByteBuffer.wrap(headerScratch);
    header.put(compressed ? COMPRESSED : UNCOMPRESSED);
    int messageLength = bufferChain.readableBytes();
    header.putInt(messageLength);
    //数据请求头
    WritableBuffer writeableHeader = bufferAllocator.allocate(HEADER_LENGTH);
    writeableHeader.write(headerScratch, 0, header.position());
    if (messageLength == 0) {
      // the payload had 0 length so make the header the current buffer.
      buffer = writeableHeader;
      return;
    }
    sink.deliverFrame(writeableHeader, false, false, messagesBuffered - 1);
    messagesBuffered = 1;
    List<WritableBuffer> bufferList = bufferChain.bufferList;
    for (int i = 0; i < bufferList.size() - 1; i++) {
      sink.deliverFrame(bufferList.get(i), false, false, 0);
    }
    //最后一个buffer等着flush（）的时候写end-of-stream=true
    buffer = bufferList.get(bufferList.size() - 1);
    currentMessageWireSize = messageLength;
  }
```
我去，细思恐极，原来发一个请求，要写入这么多次`WrtieQueue`(从上一篇博客中，我们已经知道，deliverFrame()一次，就是往`WriteQueue`里写入一次)。这里我们稍微捋一下：
1. `ClientCallImpl`调用`startCall()`发送建立`Stream`的`Header`。
2. `ClientCallImpl`中`sendMessage()`开始发送数据（ps，已经发送过`Header`了）
3. 当发送完成后，`ClientCallImpl`会`half-close`，此时会真正提交最后一个`buffer`(`end-of-stream`)到`WriteQueue`，并且发送出去(`flush=true`)。
4. 等待服务端响应。
OK，清晰明了，这里各位估计有个疑问，我们在发送`Data Frame`的时候，有一个5个字节的`writeableHeader`，WTF，这个是啥？原来这个这个`Header`还会标识这个`Frame`有没有压缩，`Frame`里边有多少个字节，看样子也是能放下一个`int`类型的了。用做什么呢？下边继续看。

### MessageDeframer
有Encode就有Decode，我们来看看`Frame`是怎么Decode的。
```
  @Override
  public void request(int numMessages) {
    checkArgument(numMessages > 0, "numMessages must be > 0");
    if (isClosed()) {
      return;
    }
    pendingDeliveries += numMessages;
    deliver();
  }
```
Sink可以通过这个方法，去对端拿`Frame`，这里就是取`Message`的个数，注意，不是`Frame`的，并且当读到`BODY`(也就是`Data Frame`)的时候才会减少。OK，这里怎么处理的呢：
直接到`deliver()`里边看：
```
  private void deliver() {
    if (inDelivery) {
      return;
    }
    inDelivery = true;
    try {
      while (!stopDelivery && pendingDeliveries > 0 && readRequiredBytes()) {
        switch (state) {
          case HEADER:
            processHeader();
            break;
          case BODY:
            // Read the body and deliver the message.
            processBody();

            // Since we've delivered a message, decrement the number of pending
            // deliveries remaining.
            pendingDeliveries--;
            break;
          default:
            throw new AssertionError("Invalid state: " + state);
        }
      }

      if (stopDelivery) {
        close();
        return;
      }
      if (closeWhenComplete && isStalled()) {
        close();
      }
    } finally {
      inDelivery = false;
    }
  }
```
原来是在`readRequiredBytes()`中进行循环读了，那`Header`和`Body`如何切换呢？这里不得不承认谷歌设计的很好。
```
private State state = State.HEADER;
private int requiredLength = HEADER_LENGTH;
```
第一次读，必须是读`State.HEADER`，还记得上边提到的5个字节的`writeableHeader`吗，原来如此。每次读到头，`processHeader()`确认下一次读的`requiredLength`，读到`BODY`，再设置读下一个`Header`（事实很少用到）。
这样，`readRequiredBytes()`才能读到指定长度的数据，才会进行处理，Google原来在Frame的下层，还会解包，社会社会。
了解了`request()`，大家也顺便知道了`deliver()`的作用，那最终`MessageDeframe`怎么调呢？
还记得古人云，`Netty`中一切都是异步的，看来`deframe()`不出意外应该是`Netty`来调了。bingo，当有响应时，`NettyServerHandler`、`NettyClientHandler`的`onDataRead`就会响起，最终调用到`deframe()`上边来。
```
  @Override
  public void deframe(ReadableBuffer data) {
    checkNotNull(data, "data");
    boolean needToCloseData = true;
    try {
      if (!isClosedOrScheduledToClose()) {
        if (fullStreamDecompressor != null) {
          fullStreamDecompressor.addGzippedBytes(data);
        } else {
          //先放入unprocessed里边
          unprocessed.addBuffer(data);
        }
        needToCloseData = false;
        deliver();
      }
    } finally {
      if (needToCloseData) {
        data.close();
      }
    }
  }
```
看来不管来的Data是啥，不管三七二十一，先放在`unprocessed`里边，交给`deliver()`处理。这就回到了`readRequiredBytes()`中的死循环里。
哎哎哎，不对啊，那`deframe()`的数据哪里去了？
原来，`processBody()`的时候传递给listener了，后续再去序列化：
```a
  private void processBody() {
    statsTraceCtx.inboundMessageRead(currentMessageSeqNo, inboundBodyWireSize, -1);
    inboundBodyWireSize = 0;
    InputStream stream = compressedFlag ? getCompressedBody() : getUncompressedBody();
    nextFrame = null;
    //重磅
    listener.messagesAvailable(new SingleMessageProducer(stream));
    state = State.HEADER;
    requiredLength = HEADER_LENGTH;
  }
```
原来如此，讲到这里，大家就知道了，原来gRPC中的请求和返回，是这样变成`BufBytes`的。

本文为作者原创，转载请注明出处 。**邮箱：568718043@qq.com**