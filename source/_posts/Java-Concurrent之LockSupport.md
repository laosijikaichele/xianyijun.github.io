---
title: Java Concurrent之LockSupport
date: 2016-05-16 11:30:15
tags: Concurrent
---

# **Java Concurrent之LockSupport** #
LockSupport为Java锁和同步类提供了基础的线程阻塞原语。
LockSupport可以通过park或者unpark方法来阻塞和唤醒线程，通过unpark/park方法来阻塞唤醒线程不会出现Thread.suspend/Thread.resume引起的死锁问题。
LockSupport和每一个使用它的线程都会与一个许可关联，而且许可只能有一个，不可累积。如果许可可用的话，并且可以在进程使用的话，调用park就立即返回，否则如果许可已经被占用的话，当前线程就会阻塞，直到调用unpark方法，获取许可。
由于许可是默认被占用的，当你调用park的话就获取不到许可，所以就进入阻塞状态。
而且LockSupport是不可重入的，当你一次线程执行两次LockSupport.park的话，线程就会一直阻塞下去。
## **源码分析** ##
### **成员变量** ###
sun.misc.Unsafe可以直接操作内存，可以用来做性能监控和开发工具。
parkBlockerOffset是指parkBlocker的偏移量，parkBlocker主要用来记录线程被阻塞的时候是被谁阻塞的，可以通过getBlocker获取阻塞的对象。在静态初始化的时候，通过反射获取Thread类parkBlocker字段对象，然后调用unsafe获取字段对象在内存中偏移量，由于parkBlocker是在线程阻塞的时候在会被设置的，直接调用线程内的方法是不会被调用的，只能通过操作内存的方式才能设置。

```java
private static final sun.misc.Unsafe UNSAFE;
private static final long parkBlockerOffset;
private static final long SEED;
private static final long PROBE;
private static final long SECONDARY;
```

静态初始化

```java
static {
    try {
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class<?> tk = Thread.class;
        parkBlockerOffset = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("parkBlocker"));
        SEED = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomSeed"));
        PROBE = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomProbe"));
        SECONDARY = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomSecondarySeed"));
    } catch (Exception ex) { throw new Error(ex); }
}
```

### **主要方法** ###
- park
为了线程调度，在许可可用之前阻塞当前线程。如果许可可用，则使用该许可，并且该调用立即返回；否则的话就会阻塞当前线程，除非其他线程调用将当前线程作为目标调用unpark、或者其他线程中断当前线程、或者该调用不合逻辑地返回。 


```java
public static void park(Object blocker) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    UNSAFE.park(false, 0L);
    setBlocker(t, null);
}
public static void parkNanos(Object blocker, long nanos) {
    if (nanos > 0) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        UNSAFE.park(false, nanos);
        setBlocker(t, null);
    }
}
public static void parkUntil(Object blocker, long deadline) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    UNSAFE.park(true, deadline);
    setBlocker(t, null);
}
public static void parkNanos(long nanos) {
    if (nanos > 0)
        UNSAFE.park(false, nanos);
}
public static void parkNanos(long nanos) {
    if (nanos > 0)
        UNSAFE.park(false, nanos);
}
```

- unPark
如果给定线程的许可尚不可用，则使其可用。如果线程在park上阻塞，则它将解除其阻塞状态。否则，保证下一次调用park 不会受阻塞。如果给定线程尚未启动，则无法保证此操作有任何效果。 

```java
public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
```

- getBlocker
从给定线程中获取对应的parkBlocker对象，返回的是阻塞线程的Blocker对象。

```java
public static Object getBlocker(Thread t) {
    if (t == null)
        throw new NullPointerException();
    return UNSAFE.getObjectVolatile(t, parkBlockerOffset);
}
```

- setBlocker
对跟定线程的parkBlocker进行设置，是一个私有方法，不允许被其他地方进行调用，防止误用。

```java
private static void setBlocker(Thread t, Object arg) {
    UNSAFE.putObject(t, parkBlockerOffset, arg);
}
```





