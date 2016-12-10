---
title: 浅析设计模式之单例模式(Java版)
date: 2016-05-31 22:25:00
tags:   设计模式
---

# **浅析设计模式之单例模式(Java版)** #
## **单例模式** ##
单例模式能够保证一个类只有一个实例，并且自行实例化并向整个系统提供实例。
### **单例模式的优点**　###

-   单例模式在内存中只有一个实例，减少了系统的内存开支，减少了对象创建和销毁时的性能损耗。
-   单例模式只有一个实例，如果对象在创建的时候需要进行一些耗资源的操作的话，可以减少系统的资源损耗。
-   单例模式可以避免对资源的多重占用，避免同时对同一个资源文件操作。
-   单例模式可以在系统设置全局访问点，优化和共享资源访问。

### **单例模式的缺点** ###

-　单例模式一般没有接口，难以扩展。

-　单例模式对测试是不利，在开发环境中，如果单例对象没有完成的话，是不能进行测试的。

-　单例模式与单一职责冲突，单例类把“单例”与业务逻辑耦合在一起的。

## **单例模式实现** ##

### **懒汉模式** ###
- 双重检验锁

为了实现线程安全，Java使用了双重检验锁来实现，在第一次检查是在同步块之外，是用来判断当前的singleton是否为null，如果为null,为了防止并发环境下，多个线程同时去创建singleton实例，导致在内存中singleton实例不唯一，使用sychronzied关键字进行同步，进入同步块之后，还需要进行null判断，防止之前if语句有多个线程进入，最后进行singleton实例的创建和初始化。
在这里singleton变量使用volatile修饰，volatile修饰主要是为了保证变量的线程可见性和禁止重排序。对象的创建并不是原子操作，它在不影响最后结果的情况下，会对操作指令执行优化和重排序，创建操作可分为三个操作：为实例对象分配内存、初始化singleton的实例变量、将实例对象指向分配的内存空间。在实际情况下，可能会发生对象还没有完成初始化就将对象指向给分配的内存空间，导致singleton不为null，即发生了不安全发布，导致在使用singleton的时候出错。
```java
public class ConcurrentSafeSingleton {
	private static volatile ConcurrentSafeSingleton singleton;

	private ConcurrentSafeSingleton() {

	}

	public static ConcurrentSafeSingleton getSingleton() {
		if (singleton == null) {
			synchronized (ConcurrentSafeSingleton.class) {
				if (singleton == null) {
					singleton = new ConcurrentSafeSingleton();
				}
			}
		}
		return singleton;
	}
}
```
- 静态内部类

由于内部类只有在调用的时候才会进行类加载和初始化，因此我们可以通过静态内存类对singleton对象进行持有，在需要使用singleton对象的时候，才调用getSingleton进行实例对象的初始化。类加载是单线程的操作，JVM保证只有一个线程在执行类的加载操作，其他试图去加载类的线程都会阻塞在执行类加载的线程的外面，等待类加载操作的完成返回。这样保证了singleton实例初始化的线程安全性。
```java
public class InnerClassSingleton {
	private static class SingletonHolder {
		public static final InnerClassSingleton SINGLETON = new InnerClassSingleton();
	}

	private InnerClassSingleton() {
	}

	public InnerClassSingleton getSingleton() {
		return SingletonHolder.SINGLETON;
	}
}
```
### **饿汉模式** ###
我们可以通过static final对变量进行修饰，保证对象的安全发布。
```java
public class Singleton {
	private static final Singleton SINGLETON = new Singleton();

	private Singleton() {

	}

	public static Singleton getSingleton() {
		return SINGLETON;
	}
}

```