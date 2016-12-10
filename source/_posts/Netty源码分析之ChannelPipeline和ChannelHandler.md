---
title: Netty源码分析之ChannelPipeline和ChannelHandler
date: 2016-05-07 11:05:44
tags: Netty
---

# Netty源码分析之ChannelPipeline和ChannelHandler
Netty的ChannelPipeline和ChannelHandler其实是责任链模式的一种变形，它将Channel的数据通道抽象成ChannelPipeline，
消息在ChannelPipeline中流动，ChannelPipeline中是事件拦截器ChannelHandler的容器，持有ChannelHandler的链表引用，对IO事件进行拦截和处理。
用户可以对ChannelHandler进行增加和删除来完成业务逻辑的处理。
![channelPipeline](http://7xrl91.com1.z0.glb.clouddn.com/ChannelPipeline.png)
## **ChannelPipeline** ##
ChannelPipeline负责对ChannelHandler的管理和事件拦截与调度，它可以运行时动态的添加和删除ChannelHandler，而且Channelpipeline是线程安全的，
可以在并发环境下对ChannelPipeline进行操作。
### **ChannelPipeline的事件处理机制** ###

- inbound事件和outBound事件
Netty中的事件分为inBound事件和outBound事件。
    -   inBound事件主要是由IO线程触发的：如Tcp链路的建立，链路关闭、读事件、异常通知事件等。
```java
 fireChannelRegistered()
 fireChannelActive()
 fireChannelRead(Object)
 fireChannelReadComplete()
 fireExceptionCaught(Throwable)
 fireUserEventTriggered(Object)
 fireChannelWritabilityChanged()
 fireChannelInactive()
 fireChannelUnregistered()
```

    -   outBound事件的话主要是由用户线程触发的：用户发起的连接操作、绑定操作、消息发送操作，
```java
 bind(SocketAddress, ChannelPromise)
connect(SocketAddress, SocketAddress, ChannelPromise)
write(Object, ChannelPromise)
flush()
read()
disconnect(ChannelPromise)
close(ChannelPromise)
deregister(ChannelPromise)
```
- 事件传播过程
![event](http://7xrl91.com1.z0.glb.clouddn.com/Selection_017.png)

### **ChannelHandler管理** ###
ChannelPipeline是ChannelHandler的管理容器，可以对ChannelHandler进行增删改查。
- 增加
我们可以通过addFirst、addLast、addBefore、addAfter方法将ChannelHandler添加到ChannelPipeline，我们可以指定添加的位置First、Last和在指定basename的ChannelHandler的前面或后面添加ChannelHandler，在添加ChannelHandler的时候，我们需要指定name，如果name为null的话，name会自动生成，而且name不能重复，否则会抛出IllegalArgumentException异常。
- 删除
我们可以通过remove、removeFirst、removeLast方法对指定的ChannelHandler或者First、Last位置的ChannelHandler进行删除，如果找不到对应的ChannelHandler的话，会抛出NoSuchElementException异常。
- 替换
我们可以通过replace方法将原来的ChannelHandler替换为新的ChannelHandler。如果找不到原来的ChannelHandler的话，就会抛出NoSuchElementException异常。进行替换时候，我们需要指定newName和newnewHandler,如果newName已经存在的话，会抛出IllegalArgumentException异常。
- 查找
我们可以通过get方法指定对应的name或者classType来获取对应的ChannelHandler，也可以通过first或者last方法来获取指定位置的ChannelHandler。如果没有找到的话，返回null。

### **DefaultChannelPipeline** ###
DefaultChannelPipeline是Netty中ChannelPipeline的默认实现类，它是包级私有和final不可继承的。ChannelPipeline不需要我们自己手动去创建的、在Bootstrap、ServerBootstrap启动的时候，为每一个Channel创建一个ChannelPipeline(默认为DefaultChannelPipeline)，然后可以添加我们自定义的ChannelPipeline。
DefaultChannelPipeline中聚合了两个AbstractChannelHandlerContext类型的实例tail和head,
AbstractChannelHandlerContext是实现了ChannelHandlerContext接口的抽象类，
维护ChannelPipeline和ChannelHandler的状态信息，在DefaultChannelHandler初始化构造函数中对它们进行了创建TailContext和HeadContext，构建了一个双向链表，在对inBound事件或者outBound事件进行响应传递的时候，会对应地从tail上浮或者从head下沉传递。
由于ChannelPipeline可以运行时动态修改ChannelHandler，DefaultChannelPipeline提供通过sychronized来保证同步代码块的操作原子性。

## **ChannelHandler** ##
ChannelHandler可以对IO事件进行拦截和处理，它可以选择性的对感兴趣的事件进行拦截和处理，也可以选择对事件传播进行传递或者终止。ChannelHandler内部提供了两个注解Skip和Sharable，用skip标志的方法不会被调用和sharable方法会多个ChannelPipeline共用一个ChannelHandler。
ChannelHandler接口本身没有提供很多的方法，通常需要实现它的子接口ChannelInboundHandler或者ChannelOutboundHandler，也可以继承ChannelInboundHandlerAdapter、ChannelOutboundHandlerAdapter和ChannelDuplexHandler来处理IO操作或者事件。
```java

void handlerAdded(ChannelHandlerContext ctx) throws Exception;

void handlerRemoved(ChannelHandlerContext ctx) throws Exception;
```
### **ChannelInboundHandler** ###
```java
void channelRegistered(ChannelHandlerContext ctx) throws Exception;

void channelUnregistered(ChannelHandlerContext ctx) throws Exception;

void channelActive(ChannelHandlerContext ctx) throws Exception;

void channelInactive(ChannelHandlerContext ctx) throws Exception;

void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception;

void channelReadComplete(ChannelHandlerContext ctx) throws Exception;

void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception;

void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception;

void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception;
```
### **ChannelOutboundHandler** ###
```java
void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) throws Exception;
void connect(
        ChannelHandlerContext ctx, SocketAddress remoteAddress,
        SocketAddress localAddress, ChannelPromise promise) throws Exception;
void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;
void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;
void deregister(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;
void read(ChannelHandlerContext ctx) throws Exception;
void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception;
void flush(ChannelHandlerContext ctx) throws Exception;
```
## **ChannelHandlerAdapter** ##
在一般情况下，我们不需要对所有的IO事件进行处理，只需要处理我们感兴趣的事件就可以了，但是如果我们要实现ChannelHandler接口的话，需要对所有的方法都进行实现，十分繁琐，Netty为我们提供了一个适配器ChannelHandlerAdapter，它实现ChannelHandler的所有方法，进行透传，如果我们有感兴趣的事件需要进行处理的话，只需要对对应的方法进行重写就可以了。
ChannelHandlerAdapter中的方法要么是空实现，要么就是委托给ChannelHandlerContext来处理。ChannelHandlerAdapter主要是对isSharable进行了实现。
通过ThreadLocal和WeakHashMap来消除volatile的读写，但是每个线程一个WeakHashMap的话，会受到线程数的限制。
```java
public boolean isSharable() {
    Class<?> clazz = getClass();
    Map<Class<?>, Boolean> cache = InternalThreadLocalMap.get().handlerSharableCache();
    Boolean sharable = cache.get(clazz);
    if (sharable == null) {
        sharable = clazz.isAnnotationPresent(Sharable.class);
        cache.put(clazz, sharable);
    }
    return sharable;
}
```
### **ChannelInboundHandlerAdapter** ###
### **ChannelOutboundHandlerAdapter** ###
