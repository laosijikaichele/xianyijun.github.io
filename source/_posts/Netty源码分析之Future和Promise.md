---
title: Netty源码分析之Future和Promise
date: 2016-05-02 22:24:46
tags: Netty
---

# Netty源码分析之Future和Promise #
## **Future** ##
Future表示异步操作的结果，可以通过get方法来获取结果，如果操作还没有完成的话，线程会被阻塞，有可能会发生无限期阻塞，可以调用带超时时间的get方法来获取结果，如果操作达到超时时间还没有完成，就会抛出TimeoutException异常。可以通过调用cancel方法尝试对异步操作进行取消，如果操作已经完成的话，会返回false，否则返回true。
可以通过isDone和isCancelled来判断任务是否已经完成或者任务取消。
### **ChannelFuture** ###
ChannelFuture是channel异步操作的结果，由于Netty的IO操作都是异步的，所以无论调用的IO操作是否已经完成，方法都会立即返回，可以通过返回的ChannelFuture实例来获取IO操作的状态信息或者结果。ChannelFuture有两种状态：uncompleted和completed，当一个IO操作开始的时候，一个Future实例会新创建，此时的Future是uncompleted的,有三种可能结果:非成功、非失败或者非取消。当IO操作完成的时候，状态就会从uncompleted变成completed的，结果会有三种可能：成功、失败或者已取消。
ChannelFuture在Future基础之上添加了获取操作结果、添加和删除监听器、取消IO操作、同步等待的。我们可以通过添加监听器的方法来获取IO操作结果或者继续后续的处理操作。GenericFutureListener监听器接口中只有一个operationComplete方法，当IO操作完成的时候会去回调该方法，如果需要进行上下文操作，可以把上下文信息传入ChannelFuture中。同时要记得不要在ChannelHander中调用ChannelFuture的await方法，可能会导致死锁，如果发起IO操作之后，由IO操作线程去通知用户线程，如果IO操作线程和用户线程是同一线程的话，会导致IO线程等待自己等待通知自己完成，导致死锁的发生。
异步IO操作的超时有两种，一是TCP层面的IO超时，一是业务逻辑上的超时。一般情况下业务逻辑的超时时间应该大于IO操作的超时时间。
### **AbstractFuture** ###
AbstractFuture实现了Future接口，不过它不允许取消IO操作。AbstractFuture主要对get进行了重写。
get方法首先调用await进行阻塞，如果IO操作完成就会被notify,这时候会对IO操作进行检查，判断是否在IO操作期间是否有发生异常，如果没有发生异常的话，调用getNow获取结果并立即返回。否则的话，对异常进行检验，如果异常是CancellationException就抛出CancellationException异常，否则就会异常进行封装返回。在支持超时的get方法中，调用的是await(timeout, unit)方法，如果在指定的超时时间内，IO操作没有完成的话，就会抛出TimeoutException异常。
```java
public V get() throws InterruptedException, ExecutionException {
    await();

    Throwable cause = cause();
    if (cause == null) {
        return getNow();
    }
    if (cause instanceof CancellationException) {
        throw (CancellationException) cause;
    }
    throw new ExecutionException(cause);
}

public V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
    if (await(timeout, unit)) {
        Throwable cause = cause();
        if (cause == null) {
            return getNow();
        }
        if (cause instanceof CancellationException) {
            throw (CancellationException) cause;
        }
        throw new ExecutionException(cause);
    }
    throw new TimeoutException();
}
```
## **Promise** ##
Promise是特殊的可写Future。Promise在继承Future的基础之上进行了扩展，用来设置IO操作的结果。
当Netty进行IO操作的时候，会创建一个Promise对象，当操作完成或者失败的时候就会对Promise进行结果设置。
```java
Promise<V> setSuccess(V result);

boolean trySuccess(V result);

Promise<V> setFailure(Throwable cause);

boolean tryFailure(Throwable cause);

boolean setUncancellable();
```

### **DefaultPromise** ###
Netty提供了一个Promise的默认实现DefaultPromise。主要是setSuccess方法和await方法的实现

- setSuccess
set方法首先对操作结果进行判断，如果操作成功的话就调用notifyListeners通知监听器，否则的话就抛出IllegalStateException异常。
setSuccess0方法首先判断当前Promise的操作结果是否已经被设置，如果已经设置的话，不允许重复设置，会返回false;由于可能会出现IO线程与用户线程同时操作Promise，需要对设置操作结果的时候进行加锁，防止并发操作。
在加锁的临界区会对操作结果是否设置进行二次判断，如果已经设置的话，返回false。
然后对操作结果进行判断，如果结果为null，就将结果设置为默认的SUCCESS，否则的话就设置为result。
如果有正在等待异步操作完成的线程，就调用notifyAll对它们进行唤醒。
```
public Promise<V> setSuccess(V result) {
    if (setSuccess0(result)) {
        notifyListeners();
        return this;
    }
    throw new IllegalStateException("complete already: " + this);
}
private boolean setSuccess0(V result) {
    if (isDone()) {
        return false;
    }

    synchronized (this) {
        // Allow only once.
        if (isDone()) {
            return false;
        }
        if (result == null) {
            this.result = SUCCESS;
        } else {
            this.result = result;
        }
        if (hasWaiters()) {
            notifyAll();
        }
    }
    return true;
}
```
- await
await方法首先通过isDone方法判断当前的Promise是否已经设置，如果设置的了话，就直接返回。如果线程已经设置中断标志位，就抛出InterruptedException。通过加锁对当前的Promise进行锁定，使用循环对isDone结果进行判断，直到isDone返回true，如果是在IO操作中调用Promise的await或者sync方法可能会导致死锁，因此需要对死锁进行检验，防止死锁被挂死，然后调用wait方法进行等待，直到IO线程调用了setFailure、tryFailure或者setSuccess、tryFailure方法。
```
public Promise<V> await() throws InterruptedException {
    if (isDone()) {
        return this;
    }

    if (Thread.interrupted()) {
        throw new InterruptedException(toString());
    }

    synchronized (this) {
        while (!isDone()) {
            checkDeadLock();
            incWaiters();
            try {
                wait();
            } finally {
                decWaiters();
            }
        }
    }
    return this;
}
```
### **ChannelPromise** ###
ChannelPromise是特殊的可写ChannelFuture，它继承了ChannelFuture、Promise接口，它在ChannelFuture的基础没有新增方法，主要是对方法的返回值进行了覆盖，设置返回ChannelPromise。