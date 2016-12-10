---
title: Java Concurrent 框架图
date: 2016-05-01 20:39:24
tags: [Concurrent]
---

# **Java Concurrent 框架图** #
![concurrent](http://7xrl91.com1.z0.glb.clouddn.com/concurrent.png)
## **Condition** ##
Condition是条件变量的抽象接口，它把Object监视器(管程)的方法(wait、notify、notifyAll)分解成不同的对象，然后通过这些对象
与Lock的实现进行组合使用,为每一个对象提供多个wait set,其中,Lock替代了synchronized的使用,Condition替代了Object监视器方法的使用。
## **Lock** ##
Lock是Java显性锁的接口，用来控制多个线程对共享变量进行访问的工具，它比synchronzied关键字提供灵活的锁操作，允许锁在不同的作用范围进行加锁和释放操作，并且可以以任何顺序来获取与释放锁。
## **ReentrantLock** ##
ReentrantLock是一个可重入的互斥锁Lock，与synchronzied修饰的方法与语句访问的隐式监视器锁具有基本的相同语义。
## **ReadWriteLock** ##
ReadWriteLock维护了一对相关的锁，一个用于只读操作，另一个用于写入操作。只要没有 writer，读取锁可以由多个 reader 线程同时保持。写入锁是独占的。
ReadWriteLock的实现保证成功获取读锁的线程能够看到写入锁之前所做的所有更新。
### **ReentrantReadWriteLock** ###
ReentrantReadWriteLock是实现了ReentrantLock语义的ReentrantReadWriteLock。
## **AbstractOwnableSynchonizer** ##
AbstractOwnableSynchonizer是由线程以独占方式拥有的同步器。此类为创建锁和相关同步器（伴随着所有权的概念）提供了基础。AbstractOwnableSynchronizer类本身不管理或使用此信息。但是，子类和工具可以使用适当维护的值帮助控制和监视访问以及提供诊断。
### **AbstractQueuedLongSynchronizer** ###
AbstractQueuedLongSynchronizer是以long形式维护同步状态的一个AbstractQueuedSynchronizer版本，创建需要 64 位状态的多级别锁和屏障等同步器时使用。    
### **AbstractQueuedSynchonizer** ###
AbstractQueuedSynchonizer为实现依赖于先进先出 (FIFO) 等待队列的阻塞锁和相关同步器（信号量、事件，等等）提供一个框架。此类的设计目标是成为依靠单个原子int 值来表示状态的大多数同步器的一个有用基础。
子类必须定义更改此状态的受保护方法，并定义哪种状态对于此对象意味着被获取或被释放。
假定这些条件之后，此类中的其他方法就可以实现所有排队和阻塞机制。
子类可以维护其他状态字段，但只是为了获得同步而只追踪使用getState()、setState(int)和compareAndSetState(int, int)方法来操作以原子方式更新的int值。
## **LockSupport** ##
LockSupport为工具类，用来创建锁和其他同步类的基本线程阻塞原语
## **CountDownLatch** ##
CountDownLatch是同步辅助类，在完成一组正在其他线程中执行的操作之前，它允许一个或多个线程一直等待。
## **Semaphore** ##
## **CyclicBarrier** ##
CyclicBarrier是一个同步辅助类，它允许一组线程互相等待，直到到达某个公共屏障点 (common barrier point)
