---
title: Netty源码分析之NioEventLoop
date: 2016-04-30 10:19:14
tags: "Netty"
---

# Netty源码分析之Eventloop


---

## Netty线程模型 ##
Netty的线程模型可以根据不同的用户参数来选择不同的线程模型，Netty支持的线程模型有Reactor单线程模型、Reactor多线程模型和主从Reactor多线程模型。
Reactor是处理并发IO的一种常见模式。他主要是通过将所有的要处理的IO事件都注册到中心IO多路复用器上，同时主线程阻塞在多路复用器上，当有IO时间到来的时候或者准备就绪，多路复用器就会返回同时将IO事件分派到对应的事件处理器中进行处理。

- Resources 事件源，系统在指定的句柄(linux是指文件描述符，Windows则是指Socket或者handle)上注册关心的事件。
- Synchronous Event Demultiplexer 负责等待Resources的读写操作准备就绪，在非阻塞的情况下将就绪事件发送给Dispatcher。
- Dispatcher 将Synchronous Event Demultiplexer传递过来的就绪事件分发给对应的Handler处理
- Request Handler 应用程序定义的一组接口，当对应事件发生的时候进行调用，执行对应的事件处理。

### **Reactor单线程模型** ###
Reactor单线程模型是指所有的IO操作都在同一个Nio线程上面完成。由于Reactor模型是使用异步非阻塞IO，所有的IO操作都不会导致阻塞，一个线程可以独立负责处理所有的IO操作。但是对于一些高并发的系统可能会负载过重，处理速度过慢，性能下降，导致系统出现瓶颈。
### **Reactor多线程模型** ###
Reactor多线程模型是在单线程模型的基础上，使用Nio线程池来处理IO操作，使用一个专门的Nio线程来监听服务端，接收客户端的TCP连接请求。至于网络IO操作，则是通过Nio线程池来负责，线程池中包含一个任务队列和N个可用的线程，负责对信息的编码、解码、读取跟发送。
### **主从Reactor多线程模型** ###
主从Reactor多线程模型主要是由一个独立的Acceptor线程负责来处理接收客户端连接。当Acceptor接收到客户端Tcp连接请求并处理完成之后，会将新创建的SocketChannel注册到IO线程池的某个Nio线程上，由它来负责对SocketChannel的读写、编码解码操作。Acceptor线程主要是负责客户端的登陆、握手及安全验证。一旦连接成功，就会将链路注册到Nio线程池中的IO线程上，由IO线程负责相应的IO操作。
## Netty Eventloop源码分析 ##
![Eventloop](http://7xrl91.com1.z0.glb.clouddn.com/Eventloop.png)
### **EventExecutorGroup** ###
EventExecutorGroup实现了ScheduledExecutorService接口，表明EventExecutorGroup是JDK Executor类，提供了对定时任务的支持。EventExecutorGroup可以通过next方法来提供EventExecutor，除此之外，还提供了优雅关闭的方法shutdownGracefully()和管理EventExecutor的生命周期方法。

- 优雅关闭
EventExecutorGroup覆盖了在Executor中的shutdown和shutdownNow方法，并添加了@Deprecated注解，要求使用者不要再去使用，同时还提供了shutdownGracefully方法进行优雅关闭和isShuttingDown方法来判断是否关闭。shutdownGracefully方法会发送给Signals给Executor，告诉Executor开始关闭，一旦调用了shutdownGradefully方法，isShuttingDown就会返回true,优雅关闭它要保证在关闭之前的静默时间内没有任务提交，一旦在静默时间内有任务提交，静默时间又会从零开始计算和任务会被接受。shutdownGracefully返回Future，当这个EventExecutorGroup管理的所有EventExecutor都被终止的时候，可以得到通知。
```java
Future<?> shutdownGracefully(long quietPeriod, long timeout, TimeUnit unit);
Future<?> terminationFuture();
boolean isShuttingDown();
```
- EventExecutor管理
next方法会返回EventExecutorGroup管理的一个EventExecutor和child方法会返回EventExecutorGroup管理的所有EventExecutor不可变集合
```java
 EventExecutor next();
 <E extends EventExecutor> Set<E> children();
```

### **EventExecutor** ###
EventExecutor是一个特殊的EventExecutorGroup，主要是为在事件轮询中提供一些执行Thread操作的便利方法。
由于EventExecutorGroup和EventExecutor是聚合关系，所有EventExecutor的next和child方法被重载为返回自身引用或者包含自身引用的不可修改集合，同时增加了parent方法返回当前EventExecutor对应的EventExecutorGroup应用。inEventLoop方法用来查询给定的线程是否在EventExecutor管理的线程当中，还提供创建Promise和Future的方法。
```java
<V> Promise<V> newPromise();
<V> ProgressivePromise<V> newProgressivePromise();
<V> Future<V> newSucceededFuture(V result);
<V> Future<V> newFailedFuture(Throwable cause);
```
### **EventloopGroup** ###
EventloopGroup也是一种特殊的EventExecutorGroup，它允许在event loop中注册channel通道。next方法被重载为返回下一次使用的EventLoop，还新增了register方法来注册channel。register方法会返回ChannelFuture，当注册channel完成之后，channelFuture会被通知，如果是ChannelFuture register(Channel channel, ChannelPromise promise)方法的话，返回的channelFuture就是传入ChannelPromise，只有ChannelPromise成功之后，才能安全地提交任务到EventLoop中，否则任务可能会被拒绝。
```java
EventLoop next();
ChannelFuture register(Channel channel);
ChannelFuture register(Channel channel, ChannelPromise promise);
```
### **EventLoop** ###
EventLoop主要是用来处理在channel中注册的IO操作，一个EventLoop实例往往用来处理多个channel。parent方法会返回对应的EventLoopGroup，asInvoker方法会返回一个新创建的ChannelHandlerInvoker实例，来调用事件处理方法。
```java
EventLoopGroup parent();
ChannelHandlerInvoker asInvoker();
```
### **NioEventLoop** ###
NioEventLoop是SingleThreadEventLoop的具体实现，它除了负责IO操作的读写之外，还管理系统任务和定时任务的提交与执行，而且NioEventLoop使用单线程来管理Nio任务的。NioEventLoop的继承关系比较深，接下来我们从上到下进行分析
#### **JDK Executor** ####
##### **Executor** ##### 
Executor只提供了execute方法。用户只需要传入要执行的任务，不需要关注任务的执行方式，具体的实现由子类来完成。实现了执行任务与任务方法的隔离和解耦。
```java
void execute(Runnable command);
```
##### **ExecutorService** #####
ExecutorService在Executor的基础上进行了扩展，新增了关闭的相关方法、提交任务方法submit、执行任务invokeAll、invokeAny的方法。任务用Runnable和Callable来表示，Callable与Runnable的不同之处就是Callable的call方法允许返回结果和容许定义抛出受检异常。
ExecutorService的submit方法都支持返回结果，通过Future实现异步不阻塞。
```java
void shutdown();
List<Runnable> shutdownNow();
boolean isShutdown();
<T> Future<T> submit(Runnable task, T result);
<T> Future<T> submit(Callable<T> task);
<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,long timeout, TimeUnit unit) throws InterruptedException;
<T> T invokeAny(Collection<? extends Callable<T>> tasks,long timeout, TimeUnit unit)
throws InterruptedException, ExecutionException, TimeoutException;
```
##### **AbstractExecutorService** #####
AbstractExecutorService提供了ExecutorService执行方法的默认实现，通过newTaskFor方法返回的RunnableFuture实现submit、invokeAny、invokeAll方法，FutureTask是RunnableFuture的默认实现。
- **submit**
通过newTaskFor方法将Callable类型的task包装为RunnableFuture，调用ExecutorService的execute方法执行任务，返回任务执行的任务。submit方法通过将Callable包装为RunnableFuture，将submit(callable)方法代理给execute(Runnable)执行，同时还提供了Future来返回结果。newTaskFor方法是protected的，具体的实现子类可以对方法进行重写来返回其他RunnableFuture实现。
``` java
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}
```
- **invokeAny**
invokeAny方法是通过调用doInvokeAny来实现的。
首先对传入的tasks集合进行检查，检查成功之后，在开始mainloop之间，会先向线程池提交一个任务，因为如果我们的线程池只有一个线程的话，线程池中的任务是串行执行的，没有并发能力，那么在调用doInvokeAny方法的时候，只要有一个任务能够正常执行完成(没有抛出异常)，方法就会返回任务的结果。
finally块表示只要有一个任务正常执行完成，就会对其他还没有执行完成的任务调用Future.cancell进行取消。
在main loop中，声明了ntask和active变量，ntask表示还有多少个任务没有提交，active表示已经提交到线程池但还没有执行完成的任务的数目。
既当ntask等于0时，表明所有的任务已经提交到线程池中，active等于0表明所有的任务已经完成。
当向线程池提交一个任务的时候，ntask--,active++,当有任务执行完成的时候(正常完成或者异常退出)，active--。
当调用ExecutorCompletionService的poll，返回执行结果。当执行结果为null，说明当前还没有任务执行完成，如果ntask大于0的话，就将当前任务提交到线程池中，++active；如果active等于0的话，就会退出循环，表示所有的任务都是异常结束的，抛出ExecutionException异常；
如果所有的任务都提交到线程池，而且还没有任务没有执行完成的话，就判断是否设置了超时时间，如果设置了超时时间，调用ExecutorCompletionService.poll()方法，如果返回的值为null的话，说明在超时时间内没有任务执行完成，抛出TimeoutException异常；
否则的话说明有任务已经执行完成，设置超时时间为截止时间减去当前的时间；否则调用ExecutorCompletionService.take方法。如果future的值不为空的话，说明已经有任务执行完成，--active，然后调用future.get方法获取执行结果进行返回，
如果期间抛出ExecutionException或者RuntimeException异常，说明任务是异常结束的，需要继续for循环执行，查看其他任务是否执行完成；
如果抛出的InterruptedException异常，说明任务执行过程中，当前线程被中断；如果没有抛出异常的话，说明任务正常执行完成，会进行finally块，对其他还没有执行完成的任务进行取消，最后返回结果。
```java
public <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                       long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException {
    return doInvokeAny(tasks, true, unit.toNanos(timeout));
}
private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks,
                          boolean timed, long nanos)
    throws InterruptedException, ExecutionException, TimeoutException {
    if (tasks == null)
        throw new NullPointerException();
    int ntasks = tasks.size();
    if (ntasks == 0)
        throw new IllegalArgumentException();
    ArrayList<Future<T>> futures = new ArrayList<Future<T>>(ntasks);
    ExecutorCompletionService<T> ecs = new ExecutorCompletionService<T>(this);
    try {
        ExecutionException ee = null;
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        Iterator<? extends Callable<T>> it = tasks.iterator();
        futures.add(ecs.submit(it.next()));
        --ntasks;
        int active = 1;
        for (;;) {
            Future<T> f = ecs.poll();
            if (f == null) {
                if (ntasks > 0) {
                    --ntasks;
                    futures.add(ecs.submit(it.next()));
                    ++active;
                }
                else if (active == 0)
                    break;
                else if (timed) {
                    f = ecs.poll(nanos, TimeUnit.NANOSECONDS);
                    if (f == null)
                        throw new TimeoutException();
                    nanos = deadline - System.nanoTime();
                }
                else
                    f = ecs.take();
            }
            if (f != null) {
                --active;
                try {
                    return f.get();
                } catch (ExecutionException eex) {
                    ee = eex;
                } catch (RuntimeException rex) {
                    ee = new ExecutionException(rex);
                }
            }
        }

        if (ee == null)
            ee = new ExecutionException();
        throw ee;

    } finally {
        for (int i = 0, size = futures.size(); i < size; i++)
            futures.get(i).cancel(true);
    }
}

```
- **invokeAll不限时版本**
invokeAll方法中使用for循环对主线程进行阻塞，直到所有的任务都执行完成。它通过调用future.isDone进行判断，如果返回false的话，说明任务还没有执行完成，调用get方法进行阻塞。方法还声明了done来判断任务是否已经全部执行完成，当主线程被中断的时候来响应中断，将任务进行取消。
```java
public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
        if (tasks == null)
            throw new NullPointerException();
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
        boolean done = false;
        try {
            for (Callable<T> t : tasks) {
                RunnableFuture<T> f = newTaskFor(t);
                futures.add(f);
                execute(f);
            }
            for (int i = 0, size = futures.size(); i < size; i++) {
                Future<T> f = futures.get(i);
                if (!f.isDone()) {
                    try {
                        f.get();
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    }
                }
            }
            done = true;
            return futures;
        } finally {
            if (!done)
                for (int i = 0, size = futures.size(); i < size; i++)
                    futures.get(i).cancel(true);
        }
    }
```
- **invokeAll限时版本**
限时版本在不限时版本的基础上添加了超时的判断，主要是2处if(nanos < 0L)的判断。
第一个判断是在向线程池提交任务的时候进行判断，如果达到用户设置的超时时间，就立即返回futures。
第二个判断是在for循环判断任务是否已经执行完成的时候，如果任务已经完成，不会进行超时判断，如果没有完成的话，就会进行超时判断，如果超时的话，方法会立即返回。同时要对超时时间进行更新。
```java
 public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                         long timeout, TimeUnit unit)
        throws InterruptedException {
    if (tasks == null)
        throw new NullPointerException();
    long nanos = unit.toNanos(timeout);
    ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
    boolean done = false;
    try {
        for (Callable<T> t : tasks)
            futures.add(newTaskFor(t));

        final long deadline = System.nanoTime() + nanos;
        final int size = futures.size();

        for (int i = 0; i < size; i++) {
            execute((Runnable)futures.get(i));
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L)
                return futures;
        }

        for (int i = 0; i < size; i++) {
            Future<T> f = futures.get(i);
            if (!f.isDone()) {
                if (nanos <= 0L)
                    return futures;
                try {
                    f.get(nanos, TimeUnit.NANOSECONDS);
                } catch (CancellationException ignore) {
                } catch (ExecutionException ignore) {
                } catch (TimeoutException toe) {
                    return futures;
                }
                nanos = deadline - System.nanoTime();
            }
        }
        done = true;
        return futures;
    } finally {
        if (!done)
            for (int i = 0, size = futures.size(); i < size; i++)
                futures.get(i).cancel(true);
    }
}
```
### **AbstractEventExecutor** ###
AbstractEventExecutor继承了AbstractExecutorService类和实现了EventExecutor接口，是EventExecutor的实现基类。
AbstractEventExecutor对EventExecutor的方法进行了基础实现，主要是对submit方法进行覆盖，将返回的Future的覆盖为io.netty.util.concurrent.Future和将schedule相关的方法设为不可用，抛出UnsupportedOperationException异常。
同时把newTaskFor方法返回的RunnableFuture实现改为PromiseTask。
```java
protected final <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new PromiseTask<T>(this, callable);
}
```
### **AbstractScheduledEventExecutor** ###
AbstractScheduledEventExecutor继承了AbstractEventExecutor类，新增了对定时任务的支持。内部聚合了一个Queue来保持对定时任务的持有。

#### **ScheduledFutureTask** ####
ScheduledFutureTask继承PromiseTask类和实现ScheduledFuture。ScheduledFutureTask其实是java.util.concurrent.ScheduledFuture的子类，最重要的方法是getDelay方法来获取任务离执行时间的延迟值。

- **getDelay方法**
getDelay通过调用delayNanos方法获取时间结果，然后转化为对应的时间单位返回。delayNanos则是通过计算deadlineNanos与nanoTime的差值与0进行最大值比较，deadlineNanos是任务要执行的时间点，nanoTime则是当前的时间点，二者的差值就是delay值。
```java
public long getDelay(TimeUnit unit) {
	return unit.convert(delayNanos(), TimeUnit.NANOSECONDS);
}
public long delayNanos() {
	return Math.max(0, deadlineNanos() - nanoTime());
}
static long nanoTime() {
	return System.nanoTime() - START_TIME;
}
```
- **run方法**
主要调用task.call执行任务，在执行任务需要设置不能cancelled标志，在任务执行完成之后设置成功标志。
```java
public void run() {
		assert executor().inEventLoop();
		try {
			if (periodNanos == 0) {
				if (setUncancellableInternal()) {
					V result = task.call();
					setSuccessInternal(result);
				}
			} else {
				if (!isCancelled()) {
					task.call();
					if (!executor().isShutdown()) {
						long p = periodNanos;
						if (p > 0) {
							deadlineNanos += p;
						} else {
							deadlineNanos = nanoTime() - p;
						}
						if (!isCancelled()) {
							Queue<ScheduledFutureTask<?>> scheduledTaskQueue = ((AbstractScheduledEventExecutor) executor()).scheduledTaskQueue;
							assert scheduledTaskQueue != null;
							scheduledTaskQueue.add(this);
						}
					}
				}
			}
		} catch (Throwable cause) {
			setFailureInternal(cause);
		}
	}
```
#### ScheduledFutureTask管理 ####
- **添加定时任务**
定时任务的添加，首先根据传入的参数创建ScheduledFutureTask对象，然后调用schedule(final ScheduledFutureTask<V> task)方法进行添加，在schedule方法中会调用inEventLoop方法判断当前线程是不是工作线程，如果是的话，直接添加任务到队列中，否则的话创建一个OneTimeTask任务，进行execute调用，在OneTimeTask任务的run方法进行添加任务入队列，在event loop中的工作线程会执行这个OneTimeTask的。
```java
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit) {
    ObjectUtil.checkNotNull(command, "command");
    ObjectUtil.checkNotNull(unit, "unit");
    if (initialDelay < 0) {
        throw new IllegalArgumentException(
                String.format("initialDelay: %d (expected: >= 0)", initialDelay));
    }
    if (delay <= 0) {
        throw new IllegalArgumentException(
                String.format("delay: %d (expected: > 0)", delay));
    }

    return schedule(new ScheduledFutureTask<Void>(
            this, Executors.<Void>callable(command, null),
            ScheduledFutureTask.deadlineNanos(unit.toNanos(initialDelay)), -unit.toNanos(delay)));
}
<V> ScheduledFuture<V> schedule(final ScheduledFutureTask<V> task) {
    if (inEventLoop()) {
        scheduledTaskQueue().add(task);
    } else {
        execute(new OneTimeTask() {
            @Override
            public void run() {
                scheduledTaskQueue().add(task);
            }
        });
    }

    return task;
}
```
- **获取定时任务**
pollScheduledTask方法首先对任务队列进行检验是否的为空，如果为空，直接返回null；如果任务队列不为空的话，对任务队列的头元素进行判断，如果任务队列的头元素不为空且任务满足deadline时间，将任务从任务队列中删除，返回对应的任务；否则的话，都返回null。
```java
protected final Runnable pollScheduledTask() {
    return pollScheduledTask(nanoTime());
}
protected final Runnable pollScheduledTask(long nanoTime) {
    assert inEventLoop();

    Queue<ScheduledFutureTask<?>> scheduledTaskQueue = this.scheduledTaskQueue;
    ScheduledFutureTask<?> scheduledTask = scheduledTaskQueue == null ? null : scheduledTaskQueue.peek();
    if (scheduledTask == null) {
        return null;
    }

    if (scheduledTask.deadlineNanos() <= nanoTime) {
        scheduledTaskQueue.remove();
        return scheduledTask;
    } 
    return null;
}
```
### **SingleThreadEventExecutor** ###
SingleThreadEventExecutor是AbstractScheduledEventExecutor的子类，使用单线程来对任务进行处理，避免对共享资源的访问冲突，内部聚合了两个任务队列，一个是从AbstractScheduledEventExecutor继承得来的定时任务队列scheduledTaskQueue，一个是自身实现的可执行的任务队列taskQueue。
taskQueue是SingleThreadEventExecutor实例化的时候在构造函数调用newTaskQueue进行初始化的，默认是是LinkedBlockingQueue,不过newTaskQueue是protected方法，子类可以进行重写覆盖。
SingleThreadEventExecutor设定了几个状态标志位来保证任务队列在执行任务的时候保持状态的一致性。
```java
private static final int ST_NOT_STARTED = 1;
private static final int ST_STARTED = 2;
private static final int ST_SHUTTING_DOWN = 3;
private static final int ST_SHUTDOWN = 4;
private static final int ST_TERMINATED = 5;
```
- **pollTask**
从taskQueue中获取task，如果task是WAKEUP_TASK，则跳过
```java
protected Runnable pollTask() {
    assert inEventLoop();
    for (;;) {
        Runnable task = taskQueue.poll();
        if (task == WAKEUP_TASK) {
            continue;
        }
        return task;
    }
}
```
- **takeTask**
由于任务的来源有两种：taskQueue和scheduledTaskQueue，需要分情况讨论。
首先对taskQueue进行校验，如果不是阻塞队列就会抛出UnsupportedOperationException异常。
如果校验通过就进入循环直到返回任务，首先从scheduledTaskQueue中获取任务，如果任务为null的话，
说明此时没有定时任务，就从taskQueue中获取任务，如果获取的任务不是WAKEUP_TASK，就返回对应任务，否则返回null;
如果获取的定时任务不为null,scheduledTaskQueue中有定时任务，就对scheduledTask的执行时间进行判断，
如果delayNanos大于0，即定时任务还没有到执行时间，就设置超时时间为delayNanos，从taskQueue中获取任务，
如果在指定的时间内没有获取到任务(此时线程会阻塞)，就会调用fetchFromScheduledTaskQueue方法，
为了防止taskQueue一直有任务执行，scheduledTask没有机会去执行，
会从scheduledTaskQueue获取满足条件的task添加到taskQueue当中，然后再次调用taskQueue.poll方法获取task。
如果task不为空就会返回task，否则进入下一次循环。
```java
protected Runnable takeTask() {
    assert inEventLoop();
    if (!(taskQueue instanceof BlockingQueue)) {
        throw new UnsupportedOperationException();
    }

    BlockingQueue<Runnable> taskQueue = (BlockingQueue<Runnable>) this.taskQueue;
    for (;;) {
        ScheduledFutureTask<?> scheduledTask = peekScheduledTask();
        if (scheduledTask == null) {
            Runnable task = null;
            try {
                task = taskQueue.take();
                if (task == WAKEUP_TASK) {
                    task = null;
                }
            } catch (InterruptedException e) {
            }
            return task;
        } else {
            long delayNanos = scheduledTask.delayNanos();
            Runnable task = null;
            if (delayNanos > 0) {
                try {
                    task = taskQueue.poll(delayNanos, TimeUnit.NANOSECONDS);
                } catch (InterruptedException e) {
                    return null;
                }
            }
            if (task == null) {
                fetchFromScheduledTaskQueue();
                task = taskQueue.poll();
            }

            if (task != null) {
                return task;
            }
        }
    }
}
```
- **execute**
execute方法只是将任务入任务队列，并没有执行任务。它会对当前线程进行判断，如果当前线程是工作线程，就直接将任务入队列，否则就调用startThread方法，在将任务入队列。由于SingleThreadEventExecutor是单线程工作的，如果线程还没有开始的话，就通过cas调用doStartThread启动线程。
```java
public void execute(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    }

    boolean inEventLoop = inEventLoop();
    if (inEventLoop) {
        addTask(task);
    } else {
        startThread();
        addTask(task);
        if (isShutdown() && removeTask(task)) {
            reject();
        }
    }

    if (!addTaskWakesUp && wakesUpForTask(task)) {
        wakeup(inEventLoop);
    }
}

private void startThread() {
    if (STATE_UPDATER.get(this) == ST_NOT_STARTED) {
        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
            doStartThread();
        }
    }
}

private void doStartThread() {
    assert thread == null;
    executor.execute(new Runnable() {
        @Override
        public void run() {
            thread = Thread.currentThread();
            if (interrupted) {
                thread.interrupt();
            }

            boolean success = false;
            updateLastExecutionTime();
            try {
                SingleThreadEventExecutor.this.run();
                success = true;
            } catch (Throwable t) {
                logger.warn("Unexpected exception from an event executor: ", t);
            } finally {
                for (;;) {
                    int oldState = STATE_UPDATER.get(SingleThreadEventExecutor.this);
                    if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
                            SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
                        break;
                    }
                }

                if (success && gracefulShutdownStartTime == 0) {
                    logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " +
                            SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must be called " +
                            "before run() implementation terminates.");
                }

                try {
                    for (;;) {
                        if (confirmShutdown()) {
                            break;
                        }
                    }
                } finally {
                    try {
                        cleanup();
                    } finally {
                        STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
                        threadLock.release();
                        if (!taskQueue.isEmpty()) {
                            logger.warn(
                                    "An event executor terminated with " +
                                            "non-empty task queue (" + taskQueue.size() + ')');
                        }

                        terminationFuture.setSuccess(null);
                    }
                }
            }
        }
    });
}
```
- **Wake UP机制** 

WAKEUP_TASK
定义一个特殊Runnable作为标志。在pollTask或者takeTask方法如果遇到WAKE_TASK的话，会跳过或者设置为null处理。
```java
private static final Runnable WAKEUP_TASK = new Runnable() {
    @Override
    public void run() {
        // Do nothing.
    }
};
```
WAKEUP_TASK入队列
WAKEUP_TASK任务只有调用wakeup方法才会被入队列，同时还需要检验，只有是外部线程调用或者event loop线程状态为ST_SHUTTING_DOWN，即Executor正在关闭中，WAKEUP_TASK任务才会被添加。
```java
protected void wakeup(boolean inEventLoop) {
    if (!inEventLoop || STATE_UPDATER.get(this) == ST_SHUTTING_DOWN) {
        taskQueue.add(WAKEUP_TASK);
    }
}
```
wakeup方法调用
wakeup方法只有在shutdown相关方法或者execute方法中才会被调用。execute方法是在添加任务入队列之后才判断是否需要调用wakeup方法，只有addTaskWakesUp为false和wakesUpForTask返回true的时候，才会调用wakeup方法。wakesUpForTask方法默认返回true，可以在子类中被覆盖，即只要addTaskWakesUp为false就会调用wakeup方法，addTaskWakesUp是final属性，是在Executor初始化的时候进行设置的。最后也就是说只有是外部线程调用execute方法的时候才会添加WAKEUP_TASK任务作为waakeup标志。
```java
if (!addTaskWakesUp && wakesUpForTask(task)) {
        wakeup(inEventLoop);
}
```
### **SingleThreadEventLoop** ###
SingleThreadEventLoop主要是重写register方法和wakesUpForTask方法。

- **register**
调用unsafe作为register
```java
public ChannelFuture register(final Channel channel, final ChannelPromise promise) {
    if (channel == null) {
        throw new NullPointerException("channel");
    }
    if (promise == null) {
        throw new NullPointerException("promise");
    }

    channel.unsafe().register(this, promise);
    return promise;
}
```
- **wakesUpForTask**
SingleThreadEventLoop中新增了NonWakeupRunnable接口作为标志，然后wakesUpForTask方法被覆盖为检查task是不是NonWakeupRunnable。
```java
interface NonWakeupRunnable extends Runnable { }
protected boolean wakesUpForTask(Runnable task) {
    return !(task instanceof NonWakeupRunnable);
}
```
### NioEventLoop ###
NioEventLoop聚合了一个Selector作为多路复用器，负责处理IO读写事件。
Selector的初始化，是在NioEventLoop的构造函数中调用OpenSelector方法创建，同时可以通过DISABLE_KEYSET_OPTIMIZATION标志位是否对Selector的SelectionKey进行优化，标志位是通过io.netty.noKeySetOptimization进行设置。
如果没有开启SelectionKey优化，在创建Selector之后就直接返回了，否则的话就进行SelectionKey优化操作。
优化操作首先通过反射获取selectionKey和publicSelectedKeys属性，将它们的访问权限设置为true，然后使用Netty的包装类SelectedSelectionKeySet进行替换。
```java
private Selector openSelector() {
    final Selector selector;
    try {
        selector = provider.openSelector();
    } catch (IOException e) {
        throw new ChannelException("failed to open a new selector", e);
    }

    if (DISABLE_KEYSET_OPTIMIZATION) {
        return selector;
    }

    try {
        SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();

        Class<?> selectorImplClass =
                Class.forName("sun.nio.ch.SelectorImpl", false, PlatformDependent.getSystemClassLoader());

        // Ensure the current selector implementation is what we can instrument.
        if (!selectorImplClass.isAssignableFrom(selector.getClass())) {
            return selector;
        }

        Field selectedKeysField = selectorImplClass.getDeclaredField("selectedKeys");
        Field publicSelectedKeysField = selectorImplClass.getDeclaredField("publicSelectedKeys");

        selectedKeysField.setAccessible(true);
        publicSelectedKeysField.setAccessible(true);

        selectedKeysField.set(selector, selectedKeySet);
        publicSelectedKeysField.set(selector, selectedKeySet);

        selectedKeys = selectedKeySet;
        logger.trace("Instrumented an optimized java.util.Set into: {}", selector);
    } catch (Throwable t) {
        selectedKeys = null;
        logger.trace("Failed to instrument an optimized java.util.Set into: {}", selector, t);
    }

    return selector;
}
```
在分析完Selector之后，再来看run方法。
```java
protected void run() {
    for (;;) {
        boolean oldWakenUp = wakenUp.getAndSet(false);
        try {
            if (hasTasks()) {
                selectNow();
            } else {
                select(oldWakenUp);
                if (wakenUp.get()) {
                    selector.wakeup();
                }
            }

            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            if (ioRatio == 100) {
                processSelectedKeys();
                runAllTasks();
            } else {
                final long ioStartTime = System.nanoTime();

                processSelectedKeys();

                final long ioTime = System.nanoTime() - ioStartTime;
                runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
            }

            if (isShuttingDown()) {
                closeAll();
                if (confirmShutdown()) {
                    break;
                }
            }
        } catch (Throwable t) {
            logger.warn("Unexpected exception in the selector loop.", t);

            // Prevent possible consecutive immediate failures that lead to
            // excessive CPU consumption.
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                // Ignore.
            }
        }
    }
}

```
首先将wakeup原来的值保存到oldWakenUp中，然后设置为false。
然后通过hasTasks判断任务队列中是否有任务没有处理，如果有的话就调用selectNow进行select操作，selectNow方法会调用Selector的selectNow方法，如果有准备就绪的Channel，就返回就绪的Channel集合，否则返回0，在选择操作完成之后，再次判断用户是否调用了Selector的wakeup方法，如果调用了，就调用selector的wakeup方法。
如果没有任务要处理的话，就执行select方法。
```java
private void select(boolean oldWakenUp) throws IOException {
    Selector selector = this.selector;
    try {
        int selectCnt = 0;
        long currentTimeNanos = System.nanoTime();
        long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);
        for (;;) {
            long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
            if (timeoutMillis <= 0) {
                if (selectCnt == 0) {
                    selector.selectNow();
                    selectCnt = 1;
                }
                break;
            }

            int selectedKeys = selector.select(timeoutMillis);
            selectCnt ++;

            if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
                break;
            }
            if (Thread.interrupted()) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurely because " +
                            "Thread.currentThread().interrupt() was called. Use " +
                            "NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
                }
                selectCnt = 1;
                break;
            }

            long time = System.nanoTime();
            if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                selectCnt = 1;
            } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                    selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
                rebuildSelector();
                selector = this.selector;
                selector.selectNow();
                selectCnt = 1;
                break;
            }

            currentTimeNanos = time;
        }

        if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS) {
        }
    } catch (CancelledKeyException e) {
    }
}
```
首先获取系统当前纳秒时间，调用delayNanos获取定时任务的执行时间计算出超时时间，再给超时时间添加0.5毫秒的调整时间，然后对超时时间进行判断，如果需要立即执行或者已经超时了，然后调用selector.selectNow进行事件轮询，同时将selectCnt设置为1，然后退出循环；否则将剩余时间作为参数进行select操作，同时selectCnt加1(每完成一次select操作+1)，在select操作完成之后，对当前状态进行判断，如果selectionKey不等于0，即有Channel就绪、oldWakeUp为true、有用户调用过wakeup方法进行唤醒、有定时任务需要执行，就退出当前循环。否则的话，就是说当前的select操作是空轮询，就有可能导致java的epoll bug，导致IO状态时刻处于100%状态。
Netty对Bug进行了规避。它对Selector 的select操作周期进行统计，每进行一次空轮询进行一次计数，当在某周期时间内达到一定次数的时候，就认为触发BUG了，就通过重建Selector进行恢复。
```java
 public void rebuildSelector() {
    if (!inEventLoop()) {
        execute(new Runnable() {
            @Override
            public void run() {
                rebuildSelector();
            }
        });
        return;
    }

    final Selector oldSelector = selector;
    final Selector newSelector;

    if (oldSelector == null) {
        return;
    }

    try {
        newSelector = openSelector();
    } catch (Exception e) {
        logger.warn("Failed to create a new Selector.", e);
        return;
    }

    // Register all channels to the new Selector.
    int nChannels = 0;
    for (;;) {
        try {
            for (SelectionKey key: oldSelector.keys()) {
                Object a = key.attachment();
                try {
                    if (!key.isValid() || key.channel().keyFor(newSelector) != null) {
                        continue;
                    }

                    int interestOps = key.interestOps();
                    key.cancel();
                    SelectionKey newKey = key.channel().register(newSelector, interestOps, a);
                    if (a instanceof AbstractNioChannel) {
                        // Update SelectionKey
                        ((AbstractNioChannel) a).selectionKey = newKey;
                    }
                    nChannels ++;
                } catch (Exception e) {
                    logger.warn("Failed to re-register a Channel to the new Selector.", e);
                    if (a instanceof AbstractNioChannel) {
                        AbstractNioChannel ch = (AbstractNioChannel) a;
                        ch.unsafe().close(ch.unsafe().voidPromise());
                    } else {
                        @SuppressWarnings("unchecked")
                        NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                        invokeChannelUnregistered(task, key, e);
                    }
                }
            }
        } catch (ConcurrentModificationException e) {
            // Probably due to concurrent modification of the key set.
            continue;
        }

        break;
    }

    selector = newSelector;

    try {
        oldSelector.close();
    } catch (Throwable t) {
        if (logger.isWarnEnabled()) {
            logger.warn("Failed to close the old Selector.", t);
        }
    }

    logger.info("Migrated " + nChannels + " channel(s) to the new Selector.");
}
```
首先通过inEventLoop判断是否其他线程发起的rebuildSelector操作，如果是其他线程发起的话，就将rebuildSelector封装成Task，添加到任务队列中，防止多线程并发操作共享资源。
首先通过oldSelector持有原来的selector，然后通过openSelector创建新的selector，然后通过循环将原来在oldSelector注册的socketChannel重新注册到新的selector上。最后将新的selector设置为selector，旧的selector关闭。

在select操作完成之后，如果轮询有就绪的channel，就调用processSelectedKeys执行处理。
```java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized(selectedKeys.flip());
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
```
如果没有开启SelectionKey优化的话，就会调用processSelectedKeysPlain方法，首先对selectionKeys进行检验，然后获取selectionKeys的迭代器进行循环，通过获取SelectionKey和attachment，同时进行remove操作，将selectionKey从set中移除，防止下次被重复选择和处理。然后对attachment的类型进行判断，如果是AbstractNioChannel类型，说明是NioSocketChannel或者NioSeverSocketChannel,需要进行IO读写的相关操作。如果是NioTask，需要进行类型转换，调用processSelectedKey
进行处理。

```java
private void processSelectedKeysPlain(Set<SelectionKey> selectedKeys) {

    if (selectedKeys.isEmpty()) {
        return;
    }

    Iterator<SelectionKey> i = selectedKeys.iterator();
    for (;;) {
        final SelectionKey k = i.next();
        final Object a = k.attachment();
        i.remove();

        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }

        if (!i.hasNext()) {
            break;
        }

        if (needsToSelectAgain) {
            selectAgain();
            selectedKeys = selector.selectedKeys();

            // Create the iterator again to avoid ConcurrentModificationException
            if (selectedKeys.isEmpty()) {
                break;
            } else {
                i = selectedKeys.iterator();
            }
        }
    }
}
```
首先通过channel获取unsafe,判断选择键是否合法，如果不合法就调用close方法进行资源释放。否则，就对网络操作位进行判断，如果是read或者accept操作的话，就调用unsafe.read方法，如果channel是ServerSocketChannel的话，它的read操作是接收客户端连接，如果是SocketChannel的话，就是从SocketChannel读取ByteBuffer；如果网络操作位是write的话，就说明有半包消息没有发送完成，需要调用flush方法进行发送；如果网络操作位是connect的话，则需要对调用finishConnectd对连接结果进行判断，不过在判断之前需要对网络操作位进行修改，将connect操作位进行注销。
```java
  private static void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) {
        unsafe.close(unsafe.voidPromise());
        return;
    }

    try {
        int readyOps = k.readyOps();
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();
            if (!ch.isOpen()) {
                // Connection already closed - no need to handle write.
                return;
            }
        }
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            ch.unsafe().forceFlush();
        }
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);

            unsafe.finishConnect();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}

```
在处理完成IO操作之后，就可以对定时任务或者非IO任务进行执行。它首先对定时任务进行处理，如果当前没有定时任务的话，就直接返回，否则的话就对定时任务的执行时间进行判断，如果有需要立即执行或者已经超时的，就将定时任务添加到taskQueue中，同时从定时任务中删除；否则的话就说明没有定时任务需要执行，直接退出。然后通过pollTask从taskQueue获取任务，调用task.run执行任务，直到task为null，即没有需要执行的任务才退出循环。
```java
 protected boolean runAllTasks() {
    fetchFromScheduledTaskQueue();
    Runnable task = pollTask();
    if (task == null) {
        return false;
    }

    for (;;) {
        try {
            task.run();
        } catch (Throwable t) {
            logger.warn("A task raised an exception.", t);
        }

        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            return true;
        }
    }
}

```

由于NioEventLoop可以同时处理IO事件和非IO任务，可以通过IO比例ioRatio进行定制，默认为50%.同时可以设定Task的执行时间。由于获取系统时间是个耗时的操作，设定每循环60次进行判断一次，如果超过分配给非IO操作的超时时间，就会退出。
```java
final long ioStartTime = System.nanoTime();
processSelectedKeys();
final long ioTime = System.nanoTime() - ioStartTime;
runAllTasks(ioTime * (100 - ioRatio) / ioRatio);

protected boolean runAllTasks(long timeoutNanos) {
    fetchFromScheduledTaskQueue();
    Runnable task = pollTask();
    if (task == null) {
        return false;
    }

    final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
    long runTasks = 0;
    long lastExecutionTime;
    for (;;) {
        try {
            task.run();
        } catch (Throwable t) {
            logger.warn("A task raised an exception.", t);
        }

        runTasks ++;

        if ((runTasks & 0x3F) == 0) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            if (lastExecutionTime >= deadline) {
                break;
            }
        }

        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            break;
        }
    }

    this.lastExecutionTime = lastExecutionTime;
    return true;
}
```



