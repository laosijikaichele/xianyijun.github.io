---
title: Java Concurrent Collection之SynchronousQueue
date: 2016-06-17 09:46:44
tags: Concurrent
---

# **SynchronousQueue** #
SynchronousQueue是一种线程安全的同步移交的阻塞队列。SynchronousQueue比较特殊，它不像其他阻塞队列一样，有工作队列缓冲，它的每一个入队操作都必须对应其他线程的出队操作，没有数据容量。
SynchronousQueue有分为公平与非公平两个策略，公平策略是通过使用一个先进先出的队列来实现的，而非公平策略的话是通过后进先出的栈来保存的。
## **原理分析** ##
SynchronousQueue主要是通过将对应的线程绑定在对应的队列节点中，通过LockSupport的unpark和park阻塞原语实现等待，先调用的线程需要调用LockSupport的park方法将调用方法的线程进行阻塞，直到另一个与之匹配的线程调用LockSupport.unpark来唤醒在该节点等待的线程。
在队列初始状态的时候，队列为null。当一个线程进行请求操作的时候，首先对队列进行判断，如果为null的空，将线程封装为对应的节点，加入到队列中等待，如果不为null的空，就对队列的第一个节点进行判断，是否匹配，如果匹配的话则进行交易，否则不匹配的话就加入到队列。
## **源码分析** ##
### **构造函数** ###
SynchronousQueue的构造函数可以传入一个boolean类型参数来决定使用公平策略还是非公平策略，默认使用非公平策略(false);
非公平策略是用TransferStack来实现的，它是SynchronousQueue的内部类，它继承了SynchronousQueue的Transferer内部抽象类
公平策略的话则是使用TransferQueue实现的，它跟TransferStack类似，不过一个后进先出，一个先进先出。
```java
 public SynchronousQueue(boolean fair) {
        transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
    }
```

### **常用函数** ###
poll出队和offer入队方法都是调用transferer.transfer方法来实现的，它通过根据传入的元素是否为null来判断是生产者还是消费者，来判断是入队操作还是出队操作。具体的实现由子类TransferQueue和TransferStack来实现。
```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E e = transferer.transfer(null, true, unit.toNanos(timeout));
    if (e != null || !Thread.interrupted())
        return e;
    throw new InterruptedException();
}
```
```java
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    if (e == null) throw new NullPointerException();
    if (transferer.transfer(e, true, unit.toNanos(timeout)) != null)
        return true;
    if (!Thread.interrupted())
        return false;
    throw new InterruptedException();
}
```

### **公平策略** ###
- - -
#### **成员变量** ####
公平策略主要是通过一个先进先出的队列TransferQueue来保存节点。通过head和tail来指向队列的头部和尾部，cleanme是清除节点的时候需要用到的节点。

```java
/** Head of queue */
transient volatile QNode head;
/** Tail of queue */
transient volatile QNode tail;
/**
 * Reference to a cancelled node that might not yet have been
 * unlinked from queue because it was the last inserted node
 * when it was cancelled.
 */
transient volatile QNode cleanMe;
```

- - -
next和tail节点都是SNode类型的,SNode中持有item字段表示要存储的元素数据，next指向队列的下一个节点，也是SNode类型的，waiter表示在该节点上等待的线程，isData用来标志该节点是由生产者还是消费者创建的(item!=null),生产者为true，消费者为false;
```java
volatile QNode next;          // next node in queue
volatile Object item;         // CAS'ed to or from null
volatile Thread waiter;       // to control park/unpark
final boolean isData;
```
#### **主要方法** ####
- transfer
transfer首先对当前队列进行判断，对当前队列为不为null的话或者当前线程节点与队尾节点的模式是否相等进行判断，如果为null或者模式相等的话，就将当前节点加入到队列中，即要保证队列中只能有一种模式，要么是消费者模式要么是生产者模式，因为一次入队必须有一次出队。
入队操作主要分为两步，cas修改当前tail节点的next节点为当前要入队的节点，然后更新tail新节点。
在多线程环境下，这两步可以在不同线程中完成。入队完成之后，需要使用LookSupport.park方法将当前线程阻塞，直到该线程被中断或者被唤醒或者设置了超时。
如果节点被取消的话，就需要调用clean方法进行清除。
如果节点匹配的话，因为是一个先进先出的队列，所以匹配的话肯定是发生在队列的头节点，只需要更新等待节点的item进行数据传输，然后调用LockSupport.unpark对等待线程进行唤醒。

```java
E transfer(E e, boolean timed, long nanos) {
    QNode s = null; // constructed/reused as needed
    boolean isData = (e != null);

    for (;;) {
        QNode t = tail;
        QNode h = head;
        if (t == null || h == null)         // saw uninitialized value
            continue;                       // spin

        if (h == t || t.isData == isData) { // empty or same-mode
            QNode tn = t.next;
            if (t != tail)                  // inconsistent read
                continue;
            if (tn != null) {               // lagging tail
                advanceTail(t, tn);
                continue;
            }
            if (timed && nanos <= 0)        // can't wait
                return null;
            if (s == null)
                s = new QNode(e, isData);
            if (!t.casNext(null, s))        // failed to link in
                continue;

            advanceTail(t, s);              // swing tail and wait
            Object x = awaitFulfill(s, e, timed, nanos);
            if (x == s) {                   // wait was cancelled
                clean(t, s);
                return null;
            }

            if (!s.isOffList()) {           // not already unlinked
                advanceHead(t, s);          // unlink if head
                if (x != null)              // and forget fields
                    s.item = s;
                s.waiter = null;
            }
            return (x != null) ? (E)x : e;

        } else {                            // complementary-mode
            QNode m = h.next;               // node to fulfill
            if (t != tail || m == null || h != head)
                continue;                   // inconsistent read

            Object x = m.item;
            if (isData == (x != null) ||    // m already fulfilled
                x == m ||                   // m cancelled
                !m.casItem(x, e)) {         // lost CAS
                advanceHead(h, m);          // dequeue and retry
                continue;
            }

            advanceHead(h, m);              // successfully fulfilled
            LockSupport.unpark(m.waiter);
            return (x != null) ? (E)x : e;
        }
    }
}

```
- awaitFulfill
awaitFulfill阻塞线程或者自旋直到匹配，只有等待被取消、线程被中断或者匹配到对应的节点才会返回，如果等待被取消或者线程被中断都会当前节点，否则匹配到对应节点的话，就会返回匹配到的节点。
如果节点不是队列的第一个节点，不需要自旋，减少空循环浪费cpu,如果是第一个节点的话，就根据是否设置超时来决定自旋次数。
```java
 Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
    /* Same idea as TransferStack.awaitFulfill */
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    Thread w = Thread.currentThread();
    int spins = ((head.next == s) ?
                 (timed ? maxTimedSpins : maxUntimedSpins) : 0);
    for (;;) {
        if (w.isInterrupted())
            s.tryCancel(e);
        Object x = s.item;
        if (x != e)
            return x;
        if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                s.tryCancel(e);
                continue;
            }
        }
        if (spins > 0)
            --spins;
        else if (s.waiter == null)
            s.waiter = w;
        else if (!timed)
            LockSupport.park(this);
        else if (nanos > spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanos);
    }
}
```
- clean
clean清除节点，不过不清除尾节点。如果需要清除尾节点的话，需要将cleanme赋值为尾节点，然后清除cleanme.
```java
  void clean(QNode pred, QNode s) {
    s.waiter = null; // forget thread
    /*
     * At any given time, exactly one node on list cannot be
     * deleted -- the last inserted node. To accommodate this,
     * if we cannot delete s, we save its predecessor as
     * "cleanMe", deleting the previously saved version
     * first. At least one of node s or the node previously
     * saved can always be deleted, so this always terminates.
     */
    while (pred.next == s) { // Return early if already unlinked
        QNode h = head;
        QNode hn = h.next;   // Absorb cancelled first node as head
        if (hn != null && hn.isCancelled()) {
            advanceHead(h, hn);
            continue;
        }
        QNode t = tail;      // Ensure consistent read for tail
        if (t == h)
            return;
        QNode tn = t.next;
        if (t != tail)
            continue;
        if (tn != null) {
            advanceTail(t, tn);
            continue;
        }
        if (s != t) {        // If not tail, try to unsplice
            QNode sn = s.next;
            if (sn == s || pred.casNext(s, sn))
                return;
        }
        QNode dp = cleanMe;
        if (dp != null) {    // Try unlinking previous cancelled node
            QNode d = dp.next;
            QNode dn;
            if (d == null ||               // d is gone or
                d == dp ||                 // d is off list or
                !d.isCancelled() ||        // d not cancelled or
                (d != t &&                 // d not tail and
                 (dn = d.next) != null &&  //   has successor
                 dn != d &&                //   that is on list
                 dp.casNext(d, dn)))       // d unspliced
                casCleanMe(dp, null);
            if (dp == pred)
                return;      // s is already saved node
        } else if (casCleanMe(null, pred))
            return;          // Postpone cleaning s
    }
}
```
### **非公平策略** ###
SynchronousQueue的非公平策略主要是通过一个后进先出的栈TransferStack来实现的，通过持有一个head节点指向栈顶元素对栈的元素进行访问。
#### **成员变量** ####

```java
/* Modes for SNodes, ORed together in node fields */
/** Node represents an unfulfilled consumer */
static final int REQUEST    = 0;
/** Node represents an unfulfilled producer */
static final int DATA       = 1;
/** Node is fulfilling another unfulfilled DATA or REQUEST */
static final int FULFILLING = 2;
volatile SNode head;

```
head字段是一个SNode类型的成员变量，SNode中next指向节点的下一个节点，也是SNode，match则是与当前节点匹配的节点，waiter则是在该节点上等待的线程，item则是节点要保存的数据，mode表示栈中节点的状态(REQUEST/DATA/FULFILLING|REQUEST/FULFILLING|DATA),next、match、waiter使用了volatile修饰，保证了其线程可见性和禁止重排序。
```java
volatile SNode next;        // next node in stack
volatile SNode match;       // the node matched to this
volatile Thread waiter;     // to control park/unpark
Object item;                // data; or null for REQUESTs
int mode;
```
- - -

#### **主要方法** ####
- transfer
TransferStack的transfer与TransferQueue不同，它不是将被匹配的节点出队列，然后将被匹配到节点压栈，然后将匹配的两个节点出栈。
transfer首先对当前栈的状态进行判断。如果当前栈为null或者当前栈节点与栈顶节点的模式相等的话，就进行压栈操作。压栈的话，主要分为两步，将节点入栈，然后通过cas机制将head节点更新为当前节点。
在完成压栈操作之后，需要通过LockSupport.park将当前线程阻塞，直到线程中断，或者节点被取消或者被匹配的节点唤醒。
如果节点被取消的话，需要调用clean方法进行清理。
由于是栈操作，后进先出，我们匹配到的节点肯定是栈顶的两个节点，我们需要原子性更新节点的匹配节点match和更新head节点。因为我们无法保证两步操作为原子性，需要将栈顶节点的mode设置为fulfiling，防止其他线程在栈顶进行匹配操作的时候对该节点进行其他操作，其他线程可以匹配操作进行辅助操作。
```java
E transfer(E e, boolean timed, long nanos) {

    SNode s = null; // constructed/reused as needed
    int mode = (e == null) ? REQUEST : DATA;

    for (;;) {
        SNode h = head;
        if (h == null || h.mode == mode) {  // empty or same-mode
            if (timed && nanos <= 0) {      // can't wait
                if (h != null && h.isCancelled())
                    casHead(h, h.next);     // pop cancelled node
                else
                    return null;
            } else if (casHead(h, s = snode(s, e, h, mode))) {
                SNode m = awaitFulfill(s, timed, nanos);
                if (m == s) {               // wait was cancelled
                    clean(s);
                    return null;
                }
                if ((h = head) != null && h.next == s)
                    casHead(h, s.next);     // help s's fulfiller
                return (E) ((mode == REQUEST) ? m.item : s.item);
            }
        } else if (!isFulfilling(h.mode)) { // try to fulfill
            if (h.isCancelled())            // already cancelled
                casHead(h, h.next);         // pop and retry
            else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                for (;;) { // loop until matched or waiters disappear
                    SNode m = s.next;       // m is s's match
                    if (m == null) {        // all waiters are gone
                        casHead(s, null);   // pop fulfill node
                        s = null;           // use new node next time
                        break;              // restart main loop
                    }
                    SNode mn = m.next;
                    if (m.tryMatch(s)) {
                        casHead(s, mn);     // pop both s and m
                        return (E) ((mode == REQUEST) ? m.item : s.item);
                    } else                  // lost match
                        s.casNext(m, mn);   // help unlink
                }
            }
        } else {                            // help a fulfiller
            SNode m = h.next;               // m is h's match
            if (m == null)                  // waiter is gone
                casHead(h, null);           // pop fulfilling node
            else {
                SNode mn = m.next;
                if (m.tryMatch(h))          // help match
                    casHead(h, mn);         // pop both h and m
                else                        // lost match
                    h.casNext(m, mn);       // help unlink
            }
        }
    }
}
```

- awaitFulfill

```java
  SNode awaitFulfill(SNode s, boolean timed, long nanos) {

    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    Thread w = Thread.currentThread();
    int spins = (shouldSpin(s) ?
                 (timed ? maxTimedSpins : maxUntimedSpins) : 0);
    for (;;) {
        if (w.isInterrupted())
            s.tryCancel();
        SNode m = s.match;
        if (m != null)
            return m;
        if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                s.tryCancel();
                continue;
            }
        }
        if (spins > 0)
            spins = shouldSpin(s) ? (spins-1) : 0;
        else if (s.waiter == null)
            s.waiter = w; // establish waiter so can park next iter
        else if (!timed)
            LockSupport.park(this);
        else if (nanos > spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanos);
    }
}

```

- clean

```java
void clean(SNode s) {
    s.item = null;   // forget item
    s.waiter = null; // forget thread


    SNode past = s.next;
    if (past != null && past.isCancelled())
        past = past.next;

    // Absorb cancelled nodes at head
    SNode p;
    while ((p = head) != null && p != past && p.isCancelled())
        casHead(p, p.next);

    // Unsplice embedded nodes
    while (p != null && p != past) {
        SNode n = p.next;
        if (n != null && n.isCancelled())
            p.casNext(n, n.next);
        else
            p = n;
    }
}
```