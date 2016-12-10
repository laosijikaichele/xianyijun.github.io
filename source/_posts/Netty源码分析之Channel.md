---
title: Netty源码分析之Channel
date: 2016-05-04 11:15:25
tags: Netty
---

# **Netty源码分析之Channel** #
Channel是Netty提供网络操作的接口，用于异步操作。Unsafe是内部接口，聚合在Channel协助进行网络读写的操作，是一个内部辅助类

## **Channel** ##
![channel](http://7xrl91.com1.z0.glb.clouddn.com/Channel.png)
Channel提供了一系列功能方法，如网络的读、写、客户端发起连接、主动关闭连接、链路关闭、获取通信双方的链路地址或者配置通过的Config和Channel的状态。
Channel的所有IO操作都是异步的。Channel是有层次的，可以通过parent方法来获取channel的parent，这取决与Channel是如何创建的，如果是SocketChannel的话，它是ServerSocketChannel接收连接的时候创建的，它的parent是ServerSocketChannel，如果是ServerSocketChannel的话，它的parent为null。channel的eventLoop方法获取Channel注册的EventLoop，channel需要注册到EventLoop的多路复用器上来处理IO事件，eventLoop实质是处理网络读写操作的Reactor模型，可以用来处理网络事件和执行系统任务和定时任务。
### **AbstractChannel** ###
- AbstractChannel定义了静态全局异常CLOSED_CHANNEL_EXCEPTION和NOT_YET_CONNECTED_EXCEPTION，在类加载的时候会静态初始化设置堆栈信息。
- estimatorHandle 用来预测下一个报文的大小，它是通过之前数据的采样进行分析预测的。
- parent 父类Channel
- id 采用默认的方法生成默认Id
- unsafe Unsafe实例，在对象初始化的时候在构造函数中调用newUnsafe方法创建，由具体子类实现。
- pipeline 当前Channel对应的DefaultChannelPipeline
- eventLoop 当前Channel注册的EventLoop
Netty是基于事件驱动的，当Channel进行IO操作的时候会产生相应的IO事件，然后驱动事件在ChannelPipeline中传播，由对应的ChannelHandler对事件进行拦截和处理。
AbstractChannel为Channel的公共API提供了基础实现。
remoteAddress/localAddress首先从缓存的成员变量中获取，如果取到的值为null,说明是第一次获取，可以通过unsafe的remoteAddress/localAddress获取，它由对应的Channel子类实现。
```
public SocketAddress remoteAddress() {
    SocketAddress remoteAddress = this.remoteAddress;
    if (remoteAddress == null) {
        try {
            this.remoteAddress = remoteAddress = unsafe().remoteAddress();
        } catch (Throwable t) {
            // Sometimes fails on a closed socket in Windows.
            return null;
        }
    }
    return remoteAddress;
}
public SocketAddress localAddress() {
    SocketAddress localAddress = this.localAddress;
    if (localAddress == null) {
        try {
            this.localAddress = localAddress = unsafe().localAddress();
        } catch (Throwable t) {
            return null;
        }
    }
    return localAddress;
}
```
### **AbstractNioChannel** ###
成员变量

- SelectableChannel 进行IO操作。
- readInterestOp read标志位
- selectionKey 选择键，channel注册到EventLoop返回的选择键集合
- connectPromise  连接操作返回的结果
- connectTimeoutFuture 连接超时定时器
- requestedRemoteAddress 请求的通信地址信息

主要方法

- doRegister
定义成员变量selected判断是否标志注册操作是否成功，调用SelectableChannel的register方法，将当前的Channel注册到EventLoop的多路复用器上，在注册Channel的时候需要指定监听的网络操作位表示Channel网络事件感兴趣。
AbstractNioChannel注册的是0，说明对任何事件都不感兴趣，仅仅完成注册事件，注册的时候可以指定附件，在Channel接收到网络事件通知时可以从SelectionKey中重新获取之前的附件进行处理，AbstractNioChannel将自身作为附件注册。
如果注册成功的话，会返回相应的SelectionKey，如果SelectionKey已经取消的话，将会抛出CancelledKeyException异常，会对CancelledKeyException进行捕获，如果是第一次捕获的话，将会调用EventLoop的selectNow方法，将已经取消的SelectionKey从多路复用器中删除，操作成功之后，将selected设置为true，将进行下一次注册操作，如果成功就退出，否则如果是进行抛出CancelledKeyException异常的话，说明可能出现bug了，需要将异常抛到上层进行处理。
```java
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            selectionKey = javaChannel().register(eventLoop().selector, 0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                eventLoop().selectNow();
                selected = true;
            } else {
                throw e;
            }
        }
    }
}
```
- doBeginRead
在准备读操作之前需要设置标志位为读，首先需要对channel是否关闭进行判断，如果处于关闭中，则直接返回，获取当前的SelectionKey进行判断，可以合法的话，可以进行正常的操作位修改，将当前SelectionKey当前的操作位与读操作位进行位与操作，如果等于0，说明当前没有设置读操作位，可以通过interestOps | readInterestOp进行设置读标志位，这样就可以监听读事件了。
```java
protected void doBeginRead() throws Exception {
    // Channel.read() or ChannelHandlerContext.read() was called
    if (inputShutdown) {
        return;
    }

    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }

    readPending = true;

    final int interestOps = selectionKey.interestOps();
    if ((interestOps & readInterestOp) == 0) {
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}
```
### **AbstractNioByteChannel** ###
AbstractNioByteChannel是Channel操作字节的基类，聚合了Runnable对象flushTask来负责写半包操作。
doWrite方法首先发送ChannelOutboundBuffer中获取消息，对消息进行判断，如果消息为null,即所有的要发送的消息已经发送完毕了，然后清除半包标志位，退出循环。
如果消息不为null,就对消息的类型进行判断，如果消息的类型是ByteBuf的话，就对消息进行转型，然后获取ByteBuf当前可读的字节数，如果可读字节数为0的话，说明当前消息不可读，需要丢弃，就从ChannelOutboundBuffer中删除该消息，继续循环处理其他的消息。
如果可读字节数不为0的话，就声明发送消息的变量:写半包标志、消息是否已经全部发送完成、发送信息的总字节数。
如果对writeSpinCount的值为-1的话，就从配置中获取最大循环次数进行设置，防止接收消息太慢或者网络IO阻塞、导致线程假死，因为在循环发送的时候，IO线程会尝试写操作，是没有办法去处理其他的IO操作。
然后调用doWriteBytes发送消息，返回发送的字节数，具体的实现由子类来完成，如果发送的字节数为0，说明缓冲区已满，发生了ZERO_WINDOW,就将半包标志设置为true，退出循环，释放IO线程。
如果发送的字节数不为空的话，就统计信息发送字节数，然后判断缓冲区是否可读，即判断信息是否已经发送完成，如果缓冲区不可读的话，说明信息已经发送完成就设置信息发送完成标志，退出循环，然后ChannelOutboundBuffer调用progress更新信息发送状态。最后判断信息是否已经发送完成，如果信息发送完成的话，就从ChannelOutboundBuffer中删除消息，否则退出循环，调用incompleteWrite方法。

- doWrite
```java
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    int writeSpinCount = -1;

    boolean setOpWrite = false;
    for (;;) {
        Object msg = in.current();
        if (msg == null) {
            // Wrote all messages.
            clearOpWrite();
            // Directly return here so incompleteWrite(...) is not called.
            return;
        }

        if (msg instanceof ByteBuf) {
            ByteBuf buf = (ByteBuf) msg;
            int readableBytes = buf.readableBytes();
            if (readableBytes == 0) {
                in.remove();
                continue;
            }

            boolean done = false;
            long flushedAmount = 0;
            if (writeSpinCount == -1) {
                writeSpinCount = config().getWriteSpinCount();
            }
            for (int i = writeSpinCount - 1; i >= 0; i --) {
                int localFlushedAmount = doWriteBytes(buf);
                if (localFlushedAmount == 0) {
                    setOpWrite = true;
                    break;
                }

                flushedAmount += localFlushedAmount;
                if (!buf.isReadable()) {
                    done = true;
                    break;
                }
            }

            in.progress(flushedAmount);

            if (done) {
                in.remove();
            } else {
                // Break the loop and so incompleteWrite(...) is called.
                break;
            }
        } else if (msg instanceof FileRegion) {
            FileRegion region = (FileRegion) msg;
            boolean done = region.transfered() >= region.count();

            if (!done) {
                long flushedAmount = 0;
                if (writeSpinCount == -1) {
                    writeSpinCount = config().getWriteSpinCount();
                }

                for (int i = writeSpinCount - 1; i >= 0; i--) {
                    long localFlushedAmount = doWriteFileRegion(region);
                    if (localFlushedAmount == 0) {
                        setOpWrite = true;
                        break;
                    }

                    flushedAmount += localFlushedAmount;
                    if (region.transfered() >= region.count()) {
                        done = true;
                        break;
                    }
                }

                in.progress(flushedAmount);
            }

            if (done) {
                in.remove();
            } else {
                // Break the loop and so incompleteWrite(...) is called.
                break;
            }
        } else {
            // Should not reach here.
            throw new Error();
        }
    }
    incompleteWrite(setOpWrite);
}
```
- incompleteWrite
incompleteWrite首先判断是否需要设置写半包标志，如果setOpWrite为true的话，就设置写半包标志。
如果设置了写半包标志，eventLoop就会不断轮询Channel用来处理还没有发送的消息。
如果setOpWrite为false的话，会设置flushTask，启动独立的runnable来flush缓冲区数组的数据。
```java
protected final void incompleteWrite(boolean setOpWrite) {
    // Did not write completely.
    if (setOpWrite) {
        setOpWrite();
    } else {
        // Schedule flush again later so other tasks can be picked up in the meantime
        Runnable flushTask = this.flushTask;
        if (flushTask == null) {
            flushTask = this.flushTask = new Runnable() {
                @Override
                public void run() {
                    flush();
                }
            };
        }
        eventLoop().execute(flushTask);
    }
}
```
### **AbstractNioMessageChannel** ###
AbstractNioMessageChannel主要是重写了doWrite方法，首先获取通道的SelectionKey和SelectionKey的interestOps，然后获取ChannelOutboundBuffer的消息，如果消息为null的话，说明所有的消息已经所有完毕，只需要退出写半包标志，退出循环。
如果消息不为null的话，声明发送标志done，然后从channel的config中获取最大循环写次数，进行循环，通过调用oWriteMessage判断消息是否发送完成。如果成功的话就将发送标志done设置为true，退出循环。
在进行writeSpinCount次发送消息之后，判断发送结果，如果当前的消息发送完成的话，就将消息从缓冲数组中清除消息，否则的话就设置写半包标志，注册SelectionKey.OP_WRITE到EventLoop，重新发送向未发送完全的半包消息。
```java
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    final SelectionKey key = selectionKey();
    final int interestOps = key.interestOps();

    for (;;) {
        Object msg = in.current();
        if (msg == null) {
            // Wrote all messages.
            if ((interestOps & SelectionKey.OP_WRITE) != 0) {
                key.interestOps(interestOps & ~SelectionKey.OP_WRITE);
            }
            break;
        }
        try {
            boolean done = false;
            for (int i = config().getWriteSpinCount() - 1; i >= 0; i--) {
                if (doWriteMessage(msg, in)) {
                    done = true;
                    break;
                }
            }

            if (done) {
                in.remove();
            } else {
                if ((interestOps & SelectionKey.OP_WRITE) == 0) {
                    key.interestOps(interestOps | SelectionKey.OP_WRITE);
                }
                break;
            }
        } catch (IOException e) {
            if (continueOnWriteError()) {
                in.remove(e);
            } else {
                throw e;
            }
        }
    }
}
```
### **AbstractNioMessageServerChannel** ###
AbstractNioMessageServerChannel声明了EventLoopGroup的成员变量，用于为新接入的客户端NioSocketChannel分配EventLoop。每当服务端接入一个客户端连接NioSocketChannel时，都会调用childEventLoopGroup获取EventLoopGroup线程组，用于给NioSocketChannel分配Reactor线程EventLoop。
### **NioServerSocketChannel** ###
NioServerSocketChannel声明ChannelMetadata和ServerSocketChannelConfig成员变量配置ServerSocketChannel的参数和元数据。通过newSocket静态方法创建ServerSocketChannel。

- doBind
Channel可以通过doBind方法来绑定地址,它首先通过isBound方法判断服务器端监听端口是否处于绑定状态，在绑定端口的时候，可以指定backlog，即允许客户端排队的最大长度。
```java
protected void doBind(SocketAddress localAddress) throws Exception {
    javaChannel().socket().bind(localAddress, config.getBacklog());
}
```    
- doReadMessages
首先通过ServerSocketChannel的accept接收新的客户端连接，如果连接不为空，则利用当前的NioServerSocketChannel、EventLoop、socketChannel创建新的NioSocketChannel,然后加入List中，最后返回1，表示服务器消息读取成功。
```java
protected int doReadMessages(List<Object> buf) throws Exception {
    SocketChannel ch = javaChannel().accept();

    try {
        if (ch != null) {
            buf.add(new NioSocketChannel(this, childEventLoopGroup().next(), ch));
            return 1;
        }
    } catch (Throwable t) {
        logger.warn("Failed to create a new channel from an accepted socket.", t);

        try {
            ch.close();
        } catch (Throwable t2) {
            logger.warn("Failed to close a socket.", t2);
        }
    }

    return 0;
}
```
### **NioSocketChannel** ###
首先判断本地地址是否为空，如果不为空的话则调用bind方法绑定本地地址，如果绑定成功则调用connect方法进行连接发起TCP连接，然后对连接结果进行判断。如果连接成功，则返回true;如果暂时没有连接上，服务端没有返回ACK应答，连接结果不确定，需要将NioSocketChannel的SelectionKey设置为OP_CONNECT,监听连接网络操作位，返回false;连接失败的话，就抛出异常。
- 连接操作
```
protected boolean doConnect(SocketAddress remoteAddress, SocketAddress localAddress) throws Exception {
    if (localAddress != null) {
        javaChannel().socket().bind(localAddress);
    }

    boolean success = false;
    try {
        boolean connected = javaChannel().connect(remoteAddress);
        if (!connected) {
            selectionKey().interestOps(SelectionKey.OP_CONNECT);
        }
        success = true;
        return connected;
    } finally {
        if (!success) {
            doClose();
        }
    }
}
```
- 写操作
首先获取要发送的ByteBuf个数，如果等于0的话，说明没有要写的通道，清除写操作位，退出循环；否则的话，获取写的ByteBuffer数组的个数和数组、要发送的总字节数、SocketChannel、是否发送完成的标志、是否写半包的标志;然后对nioBufferCnt进行switch判断，如果是0的话，就调用基类的doWrite方法，如果1的话，先获取ByteBuffer，然后Selector的写操作上限进行控制，调用channel的write方法，返回写入数据的字节数，如果写入字节数等于0的话，说明Tcp发送缓冲区已满，就设置写半包操作位退出循环，否则的话，进行数据统计，将需要发送的字节数减去已经发送的字节数，发送的字节数加上已发送的字节数，如果需要发送的字节数等于0，说明缓冲区的所有消息已经发送完成，设置是否发送完成标志位done为true，退出循环，如果没有发送完成，就继续循环。最后释放和跟新已写的缓冲区信息。如果退出循环的时候，消息还没有发送完成，就调用基类的incompleteWrite方法，这个在上文已经分析过，就不再分析了。
```java
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    for (;;) {
        int size = in.size();
        if (size == 0) {
            clearOpWrite();
            break;
        }
        long writtenBytes = 0;
        boolean done = false;
        boolean setOpWrite = false;

        ByteBuffer[] nioBuffers = in.nioBuffers();
        int nioBufferCnt = in.nioBufferCount();
        long expectedWrittenBytes = in.nioBufferSize();
        SocketChannel ch = javaChannel();

        switch (nioBufferCnt) {
            case 0:
                super.doWrite(in);
                return;
            case 1:
                ByteBuffer nioBuffer = nioBuffers[0];
                for (int i = config().getWriteSpinCount() - 1; i >= 0; i --) {
                    final int localWrittenBytes = ch.write(nioBuffer);
                    if (localWrittenBytes == 0) {
                        setOpWrite = true;
                        break;
                    }
                    expectedWrittenBytes -= localWrittenBytes;
                    writtenBytes += localWrittenBytes;
                    if (expectedWrittenBytes == 0) {
                        done = true;
                        break;
                    }
                }
                break;
            default:
                for (int i = config().getWriteSpinCount() - 1; i >= 0; i --) {
                    final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt);
                    if (localWrittenBytes == 0) {
                        setOpWrite = true;
                        break;
                    }
                    expectedWrittenBytes -= localWrittenBytes;
                    writtenBytes += localWrittenBytes;
                    if (expectedWrittenBytes == 0) {
                        done = true;
                        break;
                    }
                }
                break;
        }

        in.removeBytes(writtenBytes);

        if (!done) {
            incompleteWrite(setOpWrite);
            break;
        }
    }
}
```
- 读操作
NioSocketChannel的doReadBytes方法是通过封装ByteBuf和SocketChannel实现的，是通过SocketChannel读取ByteBuf最大读取数个字节到ByteBuf,然后调用ByteBuf的writeBytes进行写操作。
```java
protected int doReadBytes(ByteBuf byteBuf) throws Exception {
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
    allocHandle.attemptedBytesRead(byteBuf.writableBytes());
    return byteBuf.writeBytes(javaChannel(), allocHandle.attemptedBytesRead());
}
```
