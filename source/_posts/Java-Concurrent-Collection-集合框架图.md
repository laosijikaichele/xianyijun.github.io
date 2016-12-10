---
title: Java Concurrent Collection 集合框架图
date: 2016-05-01 20:39:49
tags: [Concurrent,Collection]
---
# Java Concurrent Collection 集合框架图
![Concurrent Collection](http://7xrl91.com1.z0.glb.clouddn.com/ConcurrentCollection.png)

## **List** ##
### **CopyOnWriteArrayList** ###
CopyOnWriteArrayList实现了List接口，相当于一个线程安全的ArrayList，可以在高并发的场景下使用。它在执行对结构会发生变化的方法的时候，会通过对底层执行一个复制操作来实现的，会有很大的性能开销，一般
是在遍历操作的使用频率远大于修改操作的情景下使用，否则的话，我们可以使用同步遍历。
## **Set** ##
### **CopyOnWriteArraySet** ###
CopyOnWriteArraySet是一个线程安全的Ｓet集合，它是通过聚合一个CopyOnWriteArray来实现的，将对CopyOnWriteArraySet的操作都委托给CopyOnWriteArray来实现的。
### **ConcurrentSkipListSet** ###
ConcurrentSkipListSet是一个基于ConcurrentSkipListMap实现的可跳跃并发的NavigableSet，Set集合中的元素的可以通过它们的自然顺序来排序，可以通过自定义的Compartor来进行排序。
## **Map** ##
### **ConcurrentHashMap** ###
ConcurrentHashMap是一个线程安全的HashMap，它是通过锁分段的机制来实现，对Map中的元素进行操作的时候，不需要对整个元素的表进行加锁，只需要对要操作的元素所在的段进行加锁就可以了，减少了锁竞争的频率，不过在执行读操作的时候，不需要对表加锁。
### **ConcurrentSkipListMap** ###
ConcurrentSkipListMap可以看做是一个线程安全的TreeMap，它可以根据Key的自然顺序进行排序，也可以根据自定义的Compartor对Key进行排序。
## Queue和Deque##
### **ConcurrentLinkedQueue** ###
ConcurrentLinkedQueue是一个基于单向链表实现的无界线程安全队列，它是按照FIFO(先进先出)的元素对元素进行排序，元素不允许为null。在队列的尾部进行插入元素，在头部进行取出元素。
### **LinkedBlockingQueue** ###
一个基于单向链表实现的阻塞队列(可以指定大小)。
### **ArrayBlockingQueue** ###
一个基于数组实现的有界阻塞队列。
### **PriorityBlockingQueue** ###
一个基于链表实现的优先队列
### **LinkedBlockingDeque** ###
一个基于双向链表实现的阻塞队列，它可以支持FILO(先进后出)的操作方式。
### **ConcurrentLinkedDeque** ###
ConcurrentLinkedQueue是一个基于双向链表实现的无界线程安全队列。