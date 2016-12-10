---
title: Java如何关闭一个线程
date: 2016-05-23 20:22:13
tags: Java
---

# **Java如何关闭一个线程** #
今天下午滴滴电话面试，由于自己在自我介绍说自己比较熟悉Java并发，所以面试官一开始就问并发相关的问题：如何杀死一个线程，当时一听到这个问题，脑袋是有一点小蒙的，因为在记忆中一直没有Java手动杀死线程相关的概念，以为是linux下用kill来杀死线程，而且一直记得线程的状态只有７种:新建，可运行，运行中，睡眠，阻塞，等待，死亡，只有run方法exit或者调用执行线程的stop方法才会进入dead态，而且stop方法在java中是Deprecated的，是不推荐使用的，所有当时就回答了stop方法，然后面试官果然就问了stop是Deprecated的，问我为什么会stop被Deprecated，会出现什么问题，应该如何实现优雅关闭，不过当时问题没有太听清楚，以为要用原生方法来实现，不过记忆中一直没有印象有什么api可以实现这个功能，不过当时博主也有点想歪了，因为之前看Netty源码的时候，EventExecutorGroup有一个shutdownGracefully的方法，以为要用threadLocal来实现，不过其实在Java Api里面就有讲到这几点。
> http://docs.oracle.com/javase/1.5.0/docs/guide/misc/threadPrimitiveDeprecation.html

- 里面讲到为什么要废弃stop方法？
主要stop本身就是不安全的，stop一个线程会导致在该线程上所有锁定的monitor(管程)都会被解锁，如果之前被这些monitor保护的对象之前处于不一致状态，其他的线程看到这些对象也会处于不一致的状态，这种对象称为damaged object,如果线程在这些受损的对象上操作的时候，可能会导致任意行为，而且这种行为难以检测，不像unchecked异常，这样容易发现。ThreadDead异常会杀死其他线程，导致用户可能收不到警告，只有在正在崩坏的时候才能发现。

- 如何取代stop方法来关闭线程。
我们应该通过共享变量标志来表明是否线程需要停止运行的代码，而且目标线程应该有规律的去检测变量，如果变量指示线程需要停止的时候，目标线程应该有序地从它run方法中返回，同时，由于要保证变量的线程可见性，变量应该修饰为volatile或者进行同步访问。

```
package me.kuye.demo;

public class StopThread extends Thread {
	private volatile boolean stop;

	@Override
	public void run() {
		Thread thisThread = Thread.currentThread();
		while (!stop) {
			try {
				thisThread.sleep(1000);
			} catch (InterruptedException e) {
			}
		}
	}

	public void shutdown() {
		this.stop = true;
//		Thread.currentThread().interrupt();
	}
}
```

同样我们也可以用中断来实现stop线程，调用interrupt方法可能会导致几种情况：

- 如果线程阻塞在Object的wait方法或者Thread.join方法的话，中断标志位会被清除，然后抛出java.lang.InterruptedException。
- 如果线程阻塞在interruptible channel上的I/O操作，就是说通道将被关闭，同时线程的中断状态被设置，抛出一个java.nio.channels.ClosedByInterruptException。
- 如果线程正阻塞于一个java.nio.channels.Selector操作，则该线程的中断状态被设置，它将会立即从选择操作返回，并可能带有一个非零值，就好像调用java.nio.channels.Selector.wakeup()方法一样。
- 如果上述条件都不成立，则该线程的中断状态标志位被设置。
如果线程的中断标志位被设置之后，会在某一个时间被中断并进入dead状态。
但是这种方法并不能保证线程准确地被中断。
我们一般通过中断加共享变量是实现关闭线程。

```java
package me.kuye.demo;

public class StopThread extends Thread {
	private volatile boolean stop;

	@Override
	public void run() {
		Thread thisThread = Thread.currentThread();
		while (!stop) {
			try {
				System.out.println("the thread is running");
				thisThread.sleep(1000);
			} catch (InterruptedException e) {
				shutdown();
				System.out.println("the thread is shutdown");
			}
		}
	}

	public void shutdown() {
		System.out.println("the thread will be stop");
		this.stop = true;
		// Thread.currentThread().interrupt();
	}

	public static void main(String[] args) {
		StopThread thread = new StopThread();
		thread.start();
		try {
			Thread.currentThread().sleep(5000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		thread.shutdown();
		thread.interrupt();
		System.out.println("the main thread is exit");
	}
}

```
