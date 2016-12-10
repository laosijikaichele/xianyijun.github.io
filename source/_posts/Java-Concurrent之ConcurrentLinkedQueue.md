---
title: Java Concurrent之ConcurrentLinkedQueue
date: 2016-06-04 22:20:32
tags: Concurrent
---

# **Java Concurrent之ConcurrentLinkedQueue** #
## **ConcurrentLinkedQueue** ##
ConcurrentLinkedQueue是一个基于单向链表实现的无界线程安全队列，它是按照FIFO(先进先出)的元素对元素进行排序，元素不允许为null。在队列的尾部进行插入元素，在头部进行取出元素。
## **源码分析** ##
### **成员变量** ###
- head

```java
private transient volatile Node<E> head;
```

- tail

```java
private transient volatile Node<E> tail;
```

ConcurrentLinkedQueue中包含head和tail，用来表示队列的头节点和尾节点。
head和tail的ConcurrentLinkedQueue的内部类Node类型，表示链表的节点，内部持有item和next,item是指存储的数据，next指向节点的下一个节点，Node还提供了设置下一个节点和设置item的功能，通过UNSAFE机制和cas机制来直接操作内存保证数据的一致性和原子性。
### **主要方法** ###
#### **构造函数** ####
ConcurrentLinkedQueue在通过构造函数进行初始化的时候，会创建空的ConcurrentLinkedQueue，head和tail指向同一个节点，该节点的item域都为null,如果传入了Collection参数的话，会对该Collection的迭代器进行遍历添加元素。

```java
 public ConcurrentLinkedQueue() {
    head = tail = new Node<E>(null);
}
```

```java
public ConcurrentLinkedQueue(Collection<? extends E> c) {
    Node<E> h = null, t = null;
    for (E e : c) {
        checkNotNull(e);
        Node<E> newNode = new Node<E>(e);
        if (h == null)
            h = t = newNode;
        else {
            t.lazySetNext(newNode);
            t = newNode;
        }
    }
    if (h == null)
        h = t = new Node<E>(null);
    head = h;
    tail = t;
}
```

#### **offer** ####
offer方法会将传入元素创建一个节点，然后添加到队列尾部。
offer方法主要是完成两件事情，将入队节点设置为当前尾节点的next节点和更新tail节点，要注意tail节点指向的不一定为队列的尾节点。

```java
public boolean offer(E e) {
    checkNotNull(e);
    final Node<E> newNode = new Node<E>(e);

    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {
            if (p.casNext(null, newNode)) {
    
                if (p != t) 
                    casTail(t, newNode); 
                return true;
            }
        }
        else if (p == q)
            p = (t != (t = tail)) ? t : head;
        else
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

offer方法首先定位尾节点，由于tail节点不一定是尾节点，在元素入队的时候需要根据tail节点来定位尾节点，尾节点可能是tail节点，也可能是tail的next节点。在循环中，我们获取tail节点的next节点是否为null，如果为null的话，说明tail节点为尾节点，然后通过cas机制设置传入节点为tail的next节点和设置尾节点，如果cas失败的话，即next的值不为null说明期间有其他线程更新了尾节点，需要重新获取尾节点；否则tail的next节点不为null的话，即next可能为尾节点，如果p节点和p的next节点相等的话，即队列正在初始化，正准备添加第一个节点的时候，需要返回head节点。

#### **poll** ####
poll方法获取队列的头节点，并且将节点的item设置为null。

```java
public E poll() {
    restartFromHead:
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {
            E item = p.item;

            if (item != null && p.casItem(item, null)) {
                if (p != h) // hop two nodes at a time
                    updateHead(h, ((q = p.next) != null) ? q : p);
                return item;
            }
            else if ((q = p.next) == null) {
                updateHead(h, p);
                return null;
            }
            else if (p == q)
                continue restartFromHead;
            else
                p = q;
        }
    }
}

```

poll方法从头节点循环遍历队列，对队列节点的item域进行判断，如果item域不为null的话，就通过cas机制将item设置为null,然后更新头节点，返回item,否则item为null的话，说明有其他线程在期间弹出了节点，需要对节点的next进行判断，如果next节点也为null的话，就更新头节点，返回null;如果next节点不为null的话，就重新获取头节点，进入下一次循环。

#### **peek** ####
获取队列的头节点

```java
public E peek() {
    restartFromHead:
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {
            E item = p.item;
            if (item != null || (q = p.next) == null) {
                updateHead(h, p);
                return item;
            }
            else if (p == q)
                continue restartFromHead;
            else
                p = q;
        }
    }
}
```
peek方法从头节点遍历队列，首先对节点的item域进行判断，如果不为null的话或者item域和next域都为null的话，就更新头节点，返回item；否则的话，说明有可能其他线程在期间进行了poll操作，需要重新获取头节点，进行循环。

#### **remove** ####
remove方法从队列中清除指定节点。

```java
public boolean remove(Object o) {
    if (o == null) return false;
    Node<E> pred = null;
    for (Node<E> p = first(); p != null; p = succ(p)) {
        E item = p.item;
        if (item != null &&
            o.equals(item) &&
            p.casItem(item, null)) {
            Node<E> next = succ(p);
            if (pred != null && next != null)
                pred.casNext(p, next);
            return true;
        }
        pred = p;
    }
    return false;
}
```
first
```java
Node<E> first() {
    restartFromHead:
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {
            boolean hasItem = (p.item != null);
            if (hasItem || (q = p.next) == null) {
                updateHead(h, p);
                return hasItem ? p : null;
            }
            else if (p == q)
                continue restartFromHead;
            else
                p = q;
        }
    }
}
```
succ
```java
final Node<E> succ(Node<E> p) {
    Node<E> next = p.next;
    return (p == next) ? head : next;
}
```
remove方法首先对传入的对象进行判断，如果为null的话，返回false，否则的话进行循环，通过first方法获取队列的头节点(跳过poll的节点,即item域为nulld的节点)，通过succ方法返回下一个节点来遍历去查找匹配的节点，如果匹配成功的话，就通过cas对节点进行更新，将删除节点的pre节点的next域设置为删除节点的next域，进行队列节点更新，然后返回true。如果遍历队列结束都没有找到对应的对象，即队列中不存在该节点，返回false;