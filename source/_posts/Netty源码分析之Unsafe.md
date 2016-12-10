---
title: Netty源码分析之Unsafe
date: 2016-05-06 16:22:08
tags: Netty
---
# **Netty源码分析之Unsafe** #
![unsafe](http://7xrl91.com1.z0.glb.clouddn.com/Unsafe.png)
## **Unsafe** ##
Unsafe是Channel实现的辅助接口，不应该被用户直接调用。
## **AbstractUnsafe** ##
AbstractUnsafe实现了Unsafe接口，为Unsafe提供了基础API的实现。

- register
register方法将当前unsafe对应的Channel注册到EventLoop中的多路复用器上，然后调用ChannelPipeline的fireChannelRegistered方法，如果通道是第一次注册和激活的话，则调用ChannelPipeline的fireChannelActive方法。
register方法首先对通道的状态进行验证，如果验证通过的话，就判断当前线程是否Channel对应的NioEventLoop的线程，如果是的话，直接调用register0方法进行注册，否则的话，则是用户线程或者其他线程的发起的注册操作，需要把register0方法封装成Task，添加到NioEventLoop任务队列中。在register0方法中首先调用ensureOpen判断当前Channel是否打开，如果没有打开则无法注册，否则调用doRegister,这个方法是由子类具体实现，
```java
 public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    if (eventLoop == null) {
        throw new NullPointerException("eventLoop");zi
    }
    if (isRegistered()) {
        promise.setFailure(new IllegalStateException("registered to an event loop already"));
        return;
    }
    if (!isCompatible(eventLoop)) {
        promise.setFailure(
                new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
        return;
    }

    AbstractChannel.this.eventLoop = eventLoop;

    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        try {
            eventLoop.execute(new OneTimeTask() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            logger.warn(
                    "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                    AbstractChannel.this, t);
            closeForcibly();
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }
}

private void register0(ChannelPromise promise) {
    try {
        // check if the channel is still open as it could be closed in the mean time when the register
        // call was outside of the eventLoop
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        doRegister();
        neverRegistered = false;
        registered = true;
        safeSetSuccess(promise);
        pipeline.fireChannelRegistered();
        // Only fire a channelActive if the channel has never been registered. This prevents firing
        // multiple channel actives if the channel is deregistered and re-registered.
        if (firstRegistration && isActive()) {
            pipeline.fireChannelActive();
        }
    } catch (Throwable t) {
        // Close the channel directly to avoid FD leak.
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}

```
- bind
bind方法主要是用来绑定指定端口，对于服务器端来说，是绑定监听端口，对与客户端来说的就是绑定Socket地址。
具体的绑定是在doBind方法中实现，是由子类具体实现，如果在绑定过程中出现错误，会把错误设置到ChannelPromise中。
```java
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }
    if (Boolean.TRUE.equals(config().getOption(ChannelOption.SO_BROADCAST)) &&
        localAddress instanceof InetSocketAddress &&
        !((InetSocketAddress) localAddress).getAddress().isAnyLocalAddress() &&
        !PlatformDependent.isWindows() && !PlatformDependent.isRoot()) {
        logger.warn(
                "A non-root user can't receive a broadcast packet if the socket " +
                "is not bound to a wildcard address; binding to a non-wildcard " +
                "address (" + localAddress + ") anyway as requested.");
    }

    boolean wasActive = isActive();
    try {
        doBind(localAddress);
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        closeIfClosed();
        return;
    }
    
    if (!wasActive && isActive()) {
        invokeLater(new OneTimeTask() {
            @Override
            public void run() {
                pipeline.fireChannelActive();
            }
        });
    }
    
    safeSetSuccess(promise);
}

```
- disconnect
disconnect是用来主动关闭连接的，它首先缓存wasActive保持channel的连接状态，然后通过doDisconnect方法来交由子类来关闭连接，如果在关闭期间，发送错误，则设置promise来保存错误信息，然后立即返回；否则的话通过判断调用doDisconnect方法判断channel的连接状态，即在连接状态下成功关闭连接了，调用ChannelPipeline.fireChannelInactive方法，最后设置successPromise。
```java
  public final void disconnect(final ChannelPromise promise) {
    if (!promise.setUncancellable()) {
        return;
    }

    boolean wasActive = isActive();
    try {
        doDisconnect();
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        closeIfClosed();
        return;
    }

    if (wasActive && !isActive()) {
        invokeLater(new OneTimeTask() {
            @Override
            public void run() {
                pipeline.fireChannelInactive();
            }
        });
    }

    safeSetSuccess(promise);
    closeIfClosed(); // doDisconnect() might have closed the channel
}
```
- close
close方法是用来关闭通道的，首先会获取通道的ChannelOutboundBuffer，如果ChannelOutboundBuffer不为空的话，说明缓冲区数据数组中还没有消息没有发送出去，然后对promise的类型进行判断，如果不是VoidChannelPromise类型的话，就为closeFuture添加监听器设置结果，即在调用close方法之前，我们需要注册监听器和返回；
我们需要从closeFuture判断关闭操作是否完成，如果已经完成的话，我们只需要设置结果返回就可以了，否则的话，需要去关闭链路，将消息发送缓冲数组设置为null,通知JVM进行垃圾回收，调用prepareToClose获取close前的准备工作任务，返回Executor，如果Executor为null的话，调用EventLoop线程执行doClose0方法，否则的话将doClose0封装成Task，交由closeExecutor执行。最后将Channel从多路复用器中取消注册。
```java
private void close(final ChannelPromise promise, final Throwable cause, final boolean notify) {
    if (!promise.setUncancellable()) {
        return;
    }

    final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        // Only needed if no VoidChannelPromise.
        if (!(promise instanceof VoidChannelPromise)) {
            // This means close() was called before so we just register a listener and return
            closeFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    promise.setSuccess();
                }
            });
        }
        return;
    }

    if (closeFuture.isDone()) {
        // Closed already.
        safeSetSuccess(promise);
        return;
    }

    final boolean wasActive = isActive();
    this.outboundBuffer = null; // Disallow adding any messages and flushes to outboundBuffer.
    Executor closeExecutor = prepareToClose();
    if (closeExecutor != null) {
        closeExecutor.execute(new OneTimeTask() {
            @Override
            public void run() {
                try {
                    // Execute the close.
                    doClose0(promise);
                } finally {
                    // Call invokeLater so closeAndDeregister is executed in the EventLoop again!
                    invokeLater(new OneTimeTask() {
                        @Override
                        public void run() {
                            // Fail all the queued messages
                            outboundBuffer.failFlushed(cause, notify);
                            outboundBuffer.close(CLOSED_CHANNEL_EXCEPTION);
                            fireChannelInactiveAndDeregister(wasActive);
                        }
                    });
                }
            }
        });
    } else {
        try {
            // Close the channel and fail the queued messages in all cases.
            doClose0(promise);
        } finally {
            // Fail all the queued messages.
            outboundBuffer.failFlushed(cause, notify);
            outboundBuffer.close(CLOSED_CHANNEL_EXCEPTION);
        }
        if (inFlush0) {
            invokeLater(new OneTimeTask() {
                @Override
                public void run() {
                    fireChannelInactiveAndDeregister(wasActive);
                }
            });
        } else {
            fireChannelInactiveAndDeregister(wasActive);
        }
    }
}
```
- write
write方法只是通过ChannelPipeline将消息写入环形发送数组中，没有真正去传输数据，需要去调用flush将所有没实际写入的数据进行传输。write首先从outboundBuffer环形发送数组中获取消息，如果消息为null,说明通道已经关闭，我们需要马上失败，进行失败设置和释放消息立即返回；否则的话调用addMessage将消息放入环形发送数组。


```java
  public final void write(Object msg, ChannelPromise promise) {
        ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
        if (outboundBuffer == null) {
            // If the outboundBuffer is null we know the channel was closed and so
            // need to fail the future right away. If it is not null the handling of the rest
            // will be done in flush0()
            // See https://github.com/netty/netty/issues/2362
            safeSetFailure(promise, CLOSED_CHANNEL_EXCEPTION);
            // release message now to prevent resource-leak
            ReferenceCountUtil.release(msg);
            return;
        }

        int size;
        try {
            msg = filterOutboundMessage(msg);
            size = estimatorHandle().size(msg);
            if (size < 0) {
                size = 0;
            }
        } catch (Throwable t) {
            safeSetFailure(promise, t);
            ReferenceCountUtil.release(msg);
            return;
        }

        outboundBuffer.addMessage(msg, size, promise);
    }
```
- flush
flush将所有环形发送数组中的数据写入到Channel中，然后发送消息给通信方。
flush方法首先调用outboundBuffer的addFlush方法将发送环形数组的unflushed指针指向tail,设置要发送消息的范围，然后调用flush0进行消息发送。
首先对环形发送数组进行判断，如果为null或者为空，说明没有数组需要发送，立即返回;然后通道状态进行判断，如果通道没有连接的话,就根据通道是否关闭对failure进行相应设置和设置isFlush为false;如果没有错误的话，最后调用doWrite方法进行消息发送。
```java
 public final void flush() {
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        return;
    }

    outboundBuffer.addFlush();
    flush0();
}
    
 protected void flush0() {
    if (inFlush0) {
        return;
    }

    final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null || outboundBuffer.isEmpty()) {
        return;
    }

    inFlush0 = true;

    if (!isActive()) {
        try {
            if (isOpen()) {
                outboundBuffer.failFlushed(NOT_YET_CONNECTED_EXCEPTION, true);
            } else {
                outboundBuffer.failFlushed(CLOSED_CHANNEL_EXCEPTION, false);
            }
        } finally {
            inFlush0 = false;
        }
        return;
    }

    try {
        doWrite(outboundBuffer);
    } catch (Throwable t) {
        if (t instanceof IOException && config().isAutoClose()) {
            close(voidPromise(), t, false);
        } else {
            outboundBuffer.failFlushed(t, true);
        }
    } finally {
        inFlush0 = false;
    }
}

```
## **NioUnsafe** ##
NioUnsafe主要是在Unsafe新增了几个Nio相关的接口方法。
```java
SelectableChannel ch();

void finishConnect();

void read();

void forceFlush();
```
## **AbstractNioUnsafe** ##
AbstractNioUnsafe是AbstractUnsafe的Nio实现，主要是实现了connect和finishConnect.

- connect
connect首先对connectPromise进行判断，如果在并行条件下，有其他线程同时进行connect操作，将会抛出IllegalStateException异常。然后对当前的连接状态进行缓冲，然后调用doConnect连接操作，如果连接成功的话，就返回true，调用fulfillConnectPromise，触发ChannelActive事件，它会设置SelectionKey.OP_READ读操作标志位，否则的话，即暂时没有连接上，它会根据配置的连接超时时间设置定时任务，超时时间到了之后会突发连接校验，如果连接还是没有成功的话，就进行关闭操作，释放资源和设置异常堆栈和发起注册操作，还有设置连接结果监听器，如果连接完成会回调对连接进行判断是否取消，如果取消通道则关闭连接，释放资源，发起取消注册的操作。
```java
public final void connect(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }

    try {
        if (connectPromise != null) {
            throw new IllegalStateException("connection attempt already made");
        }

        boolean wasActive = isActive();
        if (doConnect(remoteAddress, localAddress)) {
            fulfillConnectPromise(promise, wasActive);
        } else {
            connectPromise = promise;
            requestedRemoteAddress = remoteAddress;

            // Schedule connect timeout.
            int connectTimeoutMillis = config().getConnectTimeoutMillis();
            if (connectTimeoutMillis > 0) {
                connectTimeoutFuture = eventLoop().schedule(new OneTimeTask() {
                    @Override
                    public void run() {
                        ChannelPromise connectPromise = AbstractNioChannel.this.connectPromise;
                        ConnectTimeoutException cause =
                                new ConnectTimeoutException("connection timed out: " + remoteAddress);
                        if (connectPromise != null && connectPromise.tryFailure(cause)) {
                            close(voidPromise());
                        }
                    }
                }, connectTimeoutMillis, TimeUnit.MILLISECONDS);
            }

            promise.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    if (future.isCancelled()) {
                        if (connectTimeoutFuture != null) {
                            connectTimeoutFuture.cancel(false);
                        }
                        connectPromise = null;
                        close(voidPromise());
                    }
                }
            });
        }
    } catch (Throwable t) {
        promise.tryFailure(annotateConnectException(t, remoteAddress));
        closeIfClosed();
    }
}
```
- finishConnect
finishConnect可以对连接结果进行判断，首先对通道的连接状态进行缓存，然后调用doFinishConnect方法来判断连接结果，如果连接过程发生了错误，就会抛出异常，否则如果返回true的话，说明连接成功，执行fulfillConnectPromise修改读操作位来监听网络的读事件：否则连接失败，就会关闭链路释放资源。我们还对连接超时进行判断，如果连接超时的话，需要由定时任务去关闭客户端的连接，从多路复用器中删除，释放资源。

```java
public final void finishConnect() {
    assert eventLoop().inEventLoop();

    try {
        boolean wasActive = isActive();
        doFinishConnect();
        fulfillConnectPromise(connectPromise, wasActive);
    } catch (Throwable t) {
        fulfillConnectPromise(connectPromise, annotateConnectException(t, requestedRemoteAddress));
    } finally {
        if (connectTimeoutFuture != null) {
            connectTimeoutFuture.cancel(false);
        }
        connectPromise = null;
    }
}

```
## **NioByteUnsafe** ##
NioByteUnsafe主要是对read方法的实现。
read方法首先获取config对象，如果config不是autoRead(默认为true)的话，就清除读网络操作位，进行返回；
获取ChannelPipeline管道和ByteBufAllocator内存分配器和alllocHandle内存分配算法；
首先对累积数据的相关计数器进行重置和估计下次EventLoop会有多少字节会被读取。
根据allocator进行内存分配，可能是direct或者heap内存，返回ByteBuf数组，然后allocHandle根据返回的ByteBuf设置读取字节数，如果可读取的字节数小于0的话，说明没有可读的数据，释放资源，进行垃圾回收；否则的话对当前EventLoop的读取消息数进行加一，对needReadPendingReset进行相关设置，当完成一次读操作之后，就会触发一次ChannelRead事件，对ChannelPipeline进行通知，但一次读操作并不意味着读取到一条完整的消息，要对是否还需要进行数据读取进行判断；当一条消息读取完成之后，会触发一次ChannelReadComplete事件，对ChannelPipeline进行通知。
```java
public final void read() {
    final ChannelConfig config = config();
    if (!config.isAutoRead() && !isReadPending()) {
        // ChannelConfig.setAutoRead(false) was called in the meantime
        removeReadOp();
        return;
    }

    final ChannelPipeline pipeline = pipeline();
    final ByteBufAllocator allocator = config.getAllocator();
    final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
    allocHandle.reset(config);

    ByteBuf byteBuf = null;
    try {
        boolean needReadPendingReset = true;
        do {
            byteBuf = allocHandle.allocate(allocator);
            allocHandle.lastBytesRead(doReadBytes(byteBuf));
            if (allocHandle.lastBytesRead() <= 0) {
                // nothing was read. release the buffer.
                byteBuf.release();
                byteBuf = null;
                break;
            }

            allocHandle.incMessagesRead(1);
            if (needReadPendingReset) {
                needReadPendingReset = false;
                setReadPending(false);
            }
            pipeline.fireChannelRead(byteBuf);
            byteBuf = null;
        } while (allocHandle.continueReading());

        allocHandle.readComplete();
        pipeline.fireChannelReadComplete();

        if (allocHandle.lastBytesRead() < 0) {
            closeOnRead(pipeline);
        }
    } catch (Throwable t) {
        handleReadException(pipeline, byteBuf, t, allocHandle.lastBytesRead() < 0, allocHandle);
    } finally {
        if (!config.isAutoRead() && !isReadPending()) {
            removeReadOp();
        }
    }
}
```