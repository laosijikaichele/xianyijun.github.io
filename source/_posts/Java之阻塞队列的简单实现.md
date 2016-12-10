---
title: Java之阻塞队列的简单实现
date: 2016-05-21 21:53:18
tags: Java
---
## **Java之阻塞队列的简单实现** ##
阻塞队列是指当队列为空的时候，从队列中取出元素的操作会被阻塞;当队列为满的时候，向队列添加元素的操作会被阻塞。
如果一个线程阻塞了，只能通过其他线程进行唤醒或者中断，或者如果设置了超时时间的话，等待超时时间。
因此，如果当前线程如果对一个空队列进行取出元素操作的话，当前线程会被阻塞，只能通过其他向队列添加元素的线程在添加元素成功之后对线程进行唤醒。同理，如果当前线程向一个满队列添加一个元素的话也会被阻塞，只能通过其他向队列取出元素的线程来唤醒。
我们可以通过Java对象的阻塞唤醒原语或者显性锁和条件变量来实现阻塞队列
### **ReentrantLock和Condtion实现** ###
```java
package me.kuye.demo;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class LockBlockingQueue<E> {
	private Object[] items;
	private int count;
	private int takeIndex;
	private int offerIndex;
	private ReentrantLock lock;
	private Condition notEmpty = lock.newCondition();
	private Condition notFull = lock.newCondition();
	private static final int DEFAULT_QUEUE_CAPACITY = 10;

	public LockBlockingQueue() {
		this(DEFAULT_QUEUE_CAPACITY);
	}

	public LockBlockingQueue(int capacity) {
		this.items = new Object[capacity];
	}

	public void offer(E item) throws InterruptedException {
		final ReentrantLock lock = this.lock;
		try {
			lock.lockInterruptibly();
			while (count == items.length) {
				notFull.await();
			}
			enqueue(item);
		} finally {
			lock.unlock();
		}
	}

	private void enqueue(E item) {
		final Object[] items = this.items;
		items[offerIndex] = item;
		if (++offerIndex == count) {
			offerIndex = 0;
		}
		count++;
		notEmpty.signal();
	}

	public E take() throws InterruptedException {
		final ReentrantLock lock = this.lock;
		lock.lock();
		try {
			while (count == 0) {
				notEmpty.await();
			}
			return unqueue();
		} finally {
			lock.unlock();
		}
	}

	private E unqueue() {
		final Object[] items = this.items;
		E item = (E) items[takeIndex];
		items[takeIndex] = null;
		if (++takeIndex == items.length) {
			takeIndex = 0;
		}
		count--;
		notFull.signal();
		return item;
	}
}

```
### **notify和wait实现** ###
```java
package me.kuye.demo;

public class SyncBlockingQueue<E> {
	private final Object[] items;
	private int count = 0;
	private int takeIndex = 0;
	private int offerIndex = 0;
	private static final int DEFAULT_QUEUE_CAPACITY = 10;
	private Object notEmpty = new Object();
	private Object notFull = new Object();

	public SyncBlockingQueue() {
		this(DEFAULT_QUEUE_CAPACITY);
	}

	public SyncBlockingQueue(int capacity) {
		this.items = new Object[capacity];
	}

	public void offer(E item) {
		synchronized (notFull) {
			while (count == items.length) {
				try {
					notFull.wait();
				} catch (InterruptedException e) {
					e.printStackTrace();
					notFull.notify();
				}
			}
		}
		enqueue(item);
		synchronized (notEmpty) {
			notEmpty.notify();
		}
	}

	private void enqueue(E item) {
		synchronized (items) {
			final Object[] items = this.items;
			items[offerIndex] = item;
			if (++offerIndex == items.length) {
				offerIndex = 0;
			}
			count++;
		}
	}

	private E take() {
		synchronized (notEmpty) {
			while (count == 0) {
				try {
					notEmpty.wait();
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
		}
		E object = dequeue();
		synchronized (notFull) {
			notFull.notify();
		}
		return object;
	}

	private E dequeue() {
		synchronized (items) {
			final Object[] items = this.items;
			Object object = items[takeIndex];
			items[takeIndex] = null;
			if (++takeIndex == items.length) {
				takeIndex = 0;
			}
			count--;
			return (E) object;
		}
	}

}

```