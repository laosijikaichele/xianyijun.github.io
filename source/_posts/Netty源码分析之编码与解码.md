---
title: Netty源码分析之编码与解码
date: 2016-05-07 20:50:58
tags: Netty
---

# **Netty源码分析之编码与解码** #
## **解码** ##
### **ByteToMessageDecoder** ###
在进行网络编程的时候，我们往往需要将读取到的字节数组或者缓冲区数组转化为对应业务逻辑的POJO对象。Netty为我们提供ByteToMessageDecoder抽象工具类将ByteBuf转为POJO对象，我们只需要继续ByteToMessageDecoder抽象类，实现decode方法就可以了。不过ByteToMessageDecoder没有提供半包或者组包的处理，需要我们自己去实现。
decode方法在channelRead方法中被调用，我们先来看一下channelRead方法。
channelRead方法首先对要解码的消息msg进行类型判断，如果msg是ByteBuf对象的话，才需要转码，否则进行直接透传；
然后通过对cumulation是否为null进行判断，如果为null，说明这是第一次解码或者上一次解码结束，没有缓存的半包消息需要进行处理，直接把data赋值给cumulation；否则不为null的话，说明cumulation缓存中有上次还没有完全解码的ByteBuf，需要进行复制操作，将要解码的ByteBuf复制到cumulation中。
在进行复制的时候，需要对cumulation的可读缓冲区进行判断，如果不足的话需要进行扩展，扩展需要利用字节缓冲区分配器ByteBufAllocator重新分配一个缓冲区，将cumulation复制到新的缓冲区，然后释放cumulation；在复制完成之后对需要解码的缓冲区进行释放，调用callDecode进行解码.
在callDecode中对ByteBuf进行循环解码，只要ByteBuf还有可读的字节，在循环中调用decode方法进行解码，是由具体子类进行实现的。在解码完成之后会对pipeLine的状态和解码结果进行判断；如果ChannelHandlerContext被移除了，就不能继续进行解码，退出循环；如果输出的out列表长度没有变化说明没有解码成功，对ByteBuf的readableBytes进行比较，如果不等说明对消息进行消费，进行循环，否则的话说明消息是半包消息，需要继续接收消息，退出循环。如果解码出一个或多个消息，却没有消费消息，抛出DecoderException异常；最后对isSingleDecode进行判断，如果是单条消息解码器，第一次解码成功之后退出循环。
```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    if (msg instanceof ByteBuf) {
        RecyclableArrayList out = RecyclableArrayList.newInstance();
        try {
            ByteBuf data = (ByteBuf) msg;
            first = cumulation == null;
            if (first) {
                cumulation = data;
            } else {
                cumulation = cumulator.cumulate(ctx.alloc(), cumulation, data);
            }
            callDecode(ctx, cumulation, out);
        } catch (DecoderException e) {
            throw e;
        } catch (Throwable t) {
            throw new DecoderException(t);
        } finally {
            if (cumulation != null && !cumulation.isReadable()) {
                numReads = 0;
                cumulation.release();
                cumulation = null;
            } else if (++ numReads >= discardAfterReads) {
                numReads = 0;
                discardSomeReadBytes();
            }

            int size = out.size();
            decodeWasNull = !out.insertSinceRecycled();
            fireChannelRead(ctx, out, size);
            out.recycle();
        }
    } else {
        ctx.fireChannelRead(msg);
    }
}
```
```java
 public static final Cumulator MERGE_CUMULATOR = new Cumulator() {
    @Override
    public ByteBuf cumulate(ByteBufAllocator alloc, ByteBuf cumulation, ByteBuf in) {
        ByteBuf buffer;
        if (cumulation.writerIndex() > cumulation.maxCapacity() - in.readableBytes()
                || cumulation.refCnt() > 1) {
            buffer = expandCumulation(alloc, cumulation, in.readableBytes());
        } else {
            buffer = cumulation;
        }
        buffer.writeBytes(in);
        in.release();
        return buffer;
    }
};
```
```java
protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    try {
        while (in.isReadable()) {
            int outSize = out.size();

            if (outSize > 0) {
                fireChannelRead(ctx, out, outSize);
                out.clear();
                if (ctx.isRemoved()) {
                    break;
                }
                outSize = 0;
            }

            int oldInputLength = in.readableBytes();
            decode(ctx, in, out);
            if (ctx.isRemoved()) {
                break;
            }

            if (outSize == out.size()) {
                if (oldInputLength == in.readableBytes()) {
                    break;
                } else {
                    continue;
                }
            }

            if (oldInputLength == in.readableBytes()) {
                throw new DecoderException(
                        StringUtil.simpleClassName(getClass()) +
                        ".decode() did not read anything but decoded a message.");
            }

            if (isSingleDecode()) {
                break;
            }
        }
    } catch (DecoderException e) {
        throw e;
    } catch (Throwable cause) {
        throw new DecoderException(cause);
    }
}
```
### **MessageToMessageDecoder** ###
MessageToMessageDecoder是Netty的二次解码器，可以把一个POJO对象解码成另一个POJO对象。
MessageToMessageDecoder首先创建一个可重复回收的RecyclableArrayList，然后对msg的类型进行判断，如果是可以解析的消息的话，调用decode进行解码，由具体子类进行实现，解码完成之后对已经解码的msg进行释放。
如果消息的类型不可以被解码的话，就将消息添加到RecyclableArrayList不进行解码。
最后遍历RecyclableArrayList，调用ChannelHandlerContext.fireChannelRead,通知接下来的ChannelHandler进行处理。通知完成之后，调用RecyclableArrayList.recycle进行释放。
```java
public void channelRead(ChannelHandlerContext. ctx, Object msg) throws Exception {
    RecyclableArrayList out = RecyclableArrayList.newInstance();
    try {
        if (acceptInboundMessage(msg)) {
            @SuppressWarnings("unchecked")
            I cast = (I) msg;
            try {
                decode(ctx, cast, out);
            } finally {
                ReferenceCountUtil.release(cast);
            }
        } else {
            out.add(msg);
        }
    } catch (DecoderException e) {
        throw e;
    } catch (Exception e) {
        throw new DecoderException(e);
    } finally {
        int size = out.size();
        for (int i = 0; i < size; i ++) {
            ctx.fireChannelRead(out.get(i));
        }
        out.recycle();
    }
}
```
### **LengthFieldBasedFrameDecoder** ###
LengthFieldBasedFrameDecoder主要是用来解决半包问题，要解决半包问题，我们首先需要区分一个整包消息：

- 固定长度整包，通过每固定长度代表一个整包消息，如果不足的话，在前面填充0补全。
- 通过回车换行符区分整包。
- 通过分隔符区分整包。
- 通过指定长度来区分整包

LengthFieldBasedFrameDecoder可以通过配置参数来解决不同的半包解决策略。
```java
private final int lengthFieldOffset;
private final int lengthFieldLength;
private final int lengthAdjustment;
private final int initialBytesToStrip;
```

decode方法首先判断需不需要丢弃当前可读的字节数。如果为真的话，需要判断要丢弃的字节数，由于丢弃的字节数不能大于可读字节数，取bytesToDiscard、readableBytes的最小值；计算完需要丢弃的字节数之后，调用ByteBuf.skip方法跳过指定的字节数，然后调用bytesToDiscard减去跳过的字节数和设置，然后failIfNecessary进行判断是否已经达到要跳过的字节数，达到的话对tooLongFrameLength进行设置。
然后对ByteBuf的可读字节数与长度偏移量进行比较，如果小于的话，说明当前缓冲区的数据不足，需要继续读取，返回null；然后通过readIndex和lengthFieldOffset得到实际的长度字段索引，获取消息报文的长度字段；获取到长度之后，需要对长度进行合法性校验，如果长度小于0的话，跳过lengthFieldEndOffset个字节，抛出CorruptedFrameException异常；
如果大于0的话，根据lengthAdjustment和lengthFieldEndOffset对长度进行修正，如果修正之后的长度小于lengthFieldEndOffset，说明数据报出错，跳过lengthFieldEndOffset个字节，抛出CorruptedFrameException异常；
如果修正后的长度大于maxFrameLength的话，需要设置tooLongFrameLength和计算要丢弃的字节数。
首先通过frameLength减去缓冲区的可读字节数，就是要丢弃的字节数，如果需要丢弃的字节数小于缓冲区可读的字节数，就直接丢弃整包消息，需要大于的话，则说明计算是丢弃整包消息也无法完成任务，需要设置discardingTooLongFrame，在下一次解码的时候继续丢弃，在丢弃完成之后，调用failIfNecessary根据情况抛出异常，返回null;
如果可读的消息小于frameLength，则说明是半包消息，返回null,继续读取数据:
然后对需要忽略的消息头字段initialBytesToStrip进行判断，如果大于frameLength的话，说明是非法数据报，抛出CorruptedFrameException异常。否则跳过忽略的消息头字段的字节数，获取整包ByteBuf；
然后通过ByteBuf进行extract获取解码后的整包消息缓冲区。
```java
protected Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
    if (discardingTooLongFrame) {
        long bytesToDiscard = this.bytesToDiscard;
        int localBytesToDiscard = (int) Math.min(bytesToDiscard, in.readableBytes());
        in.skipBytes(localBytesToDiscard);
        bytesToDiscard -= localBytesToDiscard;
        this.bytesToDiscard = bytesToDiscard;

        failIfNecessary(false);
    }

    if (in.readableBytes() < lengthFieldEndOffset) {
        return null;
    }

    int actualLengthFieldOffset = in.readerIndex() + lengthFieldOffset;
    long frameLength = getUnadjustedFrameLength(in, actualLengthFieldOffset, lengthFieldLength, byteOrder);

    if (frameLength < 0) {
        in.skipBytes(lengthFieldEndOffset);
        throw new CorruptedFrameException(
                "negative pre-adjustment length field: " + frameLength);
    }

    frameLength += lengthAdjustment + lengthFieldEndOffset;

    if (frameLength < lengthFieldEndOffset) {
        in.skipBytes(lengthFieldEndOffset);
        throw new CorruptedFrameException(
                "Adjusted frame length (" + frameLength + ") is less " +
                "than lengthFieldEndOffset: " + lengthFieldEndOffset);
    }

    if (frameLength > maxFrameLength) {
        long discard = frameLength - in.readableBytes();
        tooLongFrameLength = frameLength;

        if (discard < 0) {
            // buffer contains more bytes then the frameLength so we can discard all now
            in.skipBytes((int) frameLength);
        } else {
            // Enter the discard mode and discard everything received so far.
            discardingTooLongFrame = true;
            bytesToDiscard = discard;
            in.skipBytes(in.readableBytes());
        }
        failIfNecessary(true);
        return null;
    }

    // never overflows because it's less than maxFrameLength
    int frameLengthInt = (int) frameLength;
    if (in.readableBytes() < frameLengthInt) {
        return null;
    }

    if (initialBytesToStrip > frameLengthInt) {
        in.skipBytes(frameLengthInt);
        throw new CorruptedFrameException(
                "Adjusted frame length (" + frameLength + ") is less " +
                "than initialBytesToStrip: " + initialBytesToStrip);
    }
    in.skipBytes(initialBytesToStrip);

    // extract frame
    int readerIndex = in.readerIndex();
    int actualFrameLength = frameLengthInt - initialBytesToStrip;
    ByteBuf frame = extractFrame(ctx, in, readerIndex, actualFrameLength);
    in.readerIndex(readerIndex + actualFrameLength);
    return frame;
}

```
## **编码** ##
### **MessageToByteEncoder** ###
MessageToByteEncoder可以将POJO对象编码成ByteBuf进行传输。
首先调用acceptOutboundMessage对要发送的消息进行判断是否支持，如果部支持的话，直接透传；
否则的话根据缓冲区类型直接内存分配，直接内存或者堆内存,ioBuffer或者heapBuffer。
内存分配完成之后，调用encode进行编码，由具体子类实现，编码完成之后进行消息释放。
然后对缓冲区进行判断，如果有可读字节数，则调用ChannelHandlerContext.write进行发送；否则的话释放ByteBuf，发送一个Unpooled.EMPTY_BUFFER;
在发送完成之后，对缓冲区进行释放。
```java
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    ByteBuf buf = null;
    try {
        if (acceptOutboundMessage(msg)) {
            @SuppressWarnings("unchecked")
            I cast = (I) msg;
            buf = allocateBuffer(ctx, cast, preferDirect);
            try {
                encode(ctx, cast, buf);
            } finally {
                ReferenceCountUtil.release(cast);
            }

            if (buf.isReadable()) {
                ctx.write(buf, promise);
            } else {
                buf.release();
                ctx.write(Unpooled.EMPTY_BUFFER, promise);
            }
            buf = null;
        } else {
            ctx.write(msg, promise);
        }
    } catch (EncoderException e) {
        throw e;
    } catch (Throwable e) {
        throw new EncoderException(e);
    } finally {
        if (buf != null) {
            buf.release();
        }
    }
}
```
### **MessageToMessageEncoder** ###
MessageToMessageEncoder可以将一个POJO对象编码成另一个对象。
首先创建一个可重复利用的RecyclableArrayList，然后对需要编码的对象进行判断，看是否能进行编码，如果不可以的话，直接透传，进入下一次ChannelHandler；如果可以的话，调用encode方法进行编码，具体由子类进行实现；在完成编码之后，如果RecyclableArrayList为空，说明编码没有成功，需要释放RecyclableArrayList，抛出EncoderException异常。
如果编码成功的话，循环cyclableArrayList，调用ChannelHandlerContext发送编码之后的POJO对象。
```java
 public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    RecyclableArrayList out = null;
    try {
        if (acceptOutboundMessage(msg)) {
            out = RecyclableArrayList.newInstance();
            @SuppressWarnings("unchecked")
            I cast = (I) msg;
            try {
                encode(ctx, cast, out);
            } finally {
                ReferenceCountUtil.release(cast);
            }

            if (out.isEmpty()) {
                out.recycle();
                out = null;

                throw new EncoderException(
                        StringUtil.simpleClassName(this) + " must produce at least one message.");
            }
        } else {
            ctx.write(msg, promise);
        }
    } catch (EncoderException e) {
        throw e;
    } catch (Throwable t) {
        throw new EncoderException(t);
    } finally {
        if (out != null) {
            final int sizeMinusOne = out.size() - 1;
            if (sizeMinusOne == 0) {
                ctx.write(out.get(0), promise);
            } else if (sizeMinusOne > 0) {
                // Check if we can use a voidPromise for our extra writes to reduce GC-Pressure
                // See https://github.com/netty/netty/issues/2525
                ChannelPromise voidPromise = ctx.voidPromise();
                boolean isVoidPromise = promise == voidPromise;
                for (int i = 0; i < sizeMinusOne; i ++) {
                    ChannelPromise p;
                    if (isVoidPromise) {
                        p = voidPromise;
                    } else {
                        p = ctx.newPromise();
                    }
                    ctx.write(out.get(i), p);
                }
                ctx.write(out.get(sizeMinusOne), promise);
            }
            out.recycle();
        }
    }
}
```
#### **LengthFieldPrepender** ####
LengthFieldPrepender是MessageToMessageEncoder的实现类，实现了encode方法，可以在待发送的ByteBuf消息头中增加一个长度长度来标志消息的长度。
encode方法首先对长度进行设置，length等消息msg的可读字节数加上lengthAdjustment;如果长度包含长度消息长度本身，需要在原来length的基础上增加lengthFieldLength；
如果修改之后的length小于0，则抛出IllegalArgumentException异常；
否则的话对消息长度自身所占的字节数进行判断和校验，如果校验通过的话，将长度字段写入ByteBuf中，最后将要发送的ByteBuf复制到List<Object>out中，完成编码；否则抛出IllegalArgumentException异常。
```java
protected void encode(ChannelHandlerContext ctx, ByteBuf msg, List<Object> out) throws Exception {
    int length = msg.readableBytes() + lengthAdjustment;
    if (lengthIncludesLengthFieldLength) {
        length += lengthFieldLength;
    }

    if (length < 0) {
        throw new IllegalArgumentException(
                "Adjusted frame length (" + length + ") is less than zero");
    }

    switch (lengthFieldLength) {
    case 1:
        if (length >= 256) {
            throw new IllegalArgumentException(
                    "length does not fit into a byte: " + length);
        }
        out.add(ctx.alloc().buffer(1).order(byteOrder).writeByte((byte) length));
        break;
    case 2:
        if (length >= 65536) {
            throw new IllegalArgumentException(
                    "length does not fit into a short integer: " + length);
        }
        out.add(ctx.alloc().buffer(2).order(byteOrder).writeShort((short) length));
        break;
    case 3:
        if (length >= 16777216) {
            throw new IllegalArgumentException(
                    "length does not fit into a medium integer: " + length);
        }
        out.add(ctx.alloc().buffer(3).order(byteOrder).writeMedium(length));
        break;
    case 4:
        out.add(ctx.alloc().buffer(4).order(byteOrder).writeInt(length));
        break;
    case 8:
        out.add(ctx.alloc().buffer(8).order(byteOrder).writeLong(length));
        break;
    default:
        throw new Error("should not reach here");
    }
    out.add(msg.retain());
}
```