---
title: Java Concurrent Collection之CopyOnWriteArrayList
date: 2016-05-15 15:05:50
tags: Concurrent
---

# **Java Concurrent Collection之CopyOnWriteArrayList** #
## **CopyOnWriteArrayList** ##
CopyOnWriteArrayList是ArrayList的线程安全实现，它是在执行可变操作(mutative operations)的时候通过对底层数组进行复制来实现线程安全的，CopyOnWriteArrayList的元素允许为null。
## **源码分析**　##
### **成员变量**　###

- ReentrantLock lock lock是ReentrantLock类型成员变量，是一个可重入锁，利用lock可以实现互斥保护数据并发修改，在对数据进行添加、修改和删除等修改数组结构等操作的时候，需要获取锁，在执行完成操作之后对数据进行更新再释放锁。
- Object[] array　array使用了volatile进行修饰，保证线程对变量的可见性，确保在任何时候，线程对数据进行读写的时候都是最新的，volatile要求线程对数据进行读操作的时候需要从主内存中获取，进行写操作的时候都会把数据刷新到主内存中。

CopyOnWriteArrayList对数据进行添加、删除和修改的时候，都会创建一个新的数据，然后把数据复制到新数组中，最后把新创建的数组赋给array。
### **主要方法** ###
#### **构造函数** ####
CopyOnWriteArrayList在通过构造函数进行初始化的时候，如果传入了Collection<E>或者E[]参数的话，会通过setArray方法对array进行设置。如果Collection的类型的话是CopyOnWriteArrayList，直接调用getArray方法就可以获取Object[]数组，否则的需要调用toArray方法进行转化，如果array数组的实际类型不是Object[]的话，还是调用Array.copyOf方法进行转化。
```java
 public CopyOnWriteArrayList(Collection<? extends E> c) {
    Object[] elements;
    if (c.getClass() == CopyOnWriteArrayList.class)
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        elements = c.toArray();
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elements.getClass() != Object[].class)
            elements = Arrays.copyOf(elements, elements.length, Object[].class);
    }
    setArray(elements);
}
```

```java
 public CopyOnWriteArrayList(E[] toCopyIn) {
    setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}
```
#### **add** ####
在调用add方法的时候需要调用ReentrantLock.lock进行加锁，然后创建一个新的数组,大小为原数组的length＋１,然后把原数组复制到新创建的数组中，把len位置的元素设置为要添加的元素，最后调用setArray设置新数组。
```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```
addIfAbsent方法如果原数组已经存在要添加的元素，会返回false。
addIfAbsent需要传入快照数组(snapshot)和要添加元素，跟add方法类似，需要在方法执行初调用ReentrantLock.lock进行加锁，然后将snapshot和当前数组进行比较，如果不相等的话，说明在执行方法的时候，有其他的线程对数组执行了修改操作，需要取snapshot和原数组的长度最小值，看具有公共长度部分的数组元素是否相等，如果不相等的话，直接返回false，然后在非公共部分，判断是否已经存在要添加的元素，如果存在就返回false；否则的话，就进行数组复制，添加新元素，返回true,释放锁。
```java
 private boolean addIfAbsent(E e, Object[] snapshot) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] current = getArray();
        int len = current.length;
        if (snapshot != current) {
            // Optimize for lost race to another addXXX operation
            int common = Math.min(snapshot.length, len);
            for (int i = 0; i < common; i++)
                if (current[i] != snapshot[i] && eq(e, current[i]))
                    return false;
            if (indexOf(e, current, common, len) >= 0)
                    return false;
        }
        Object[] newElements = Arrays.copyOf(current, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```
#### **set** ####
set方法首先使用ReentrantLock进行加锁，然后获取数组指定位置的元素，跟要修改的元素进行比较，如果相等的话，就调用setArray方法确保对数组的volatile写语义，否则不等的话，就对数组进行复制，然后对指定位置的元素进行赋值修改，然后setArray刷新数组。
```java
public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        E oldValue = get(elements, index);

        if (oldValue != element) {
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len);
            newElements[index] = element;
            setArray(newElements);
        } else {
            // Not quite a no-op; ensures volatile write semantics
            setArray(elements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```
#### **remove** ####
remove需要传入要删除的元素、快照数组和元素在快照数组的索引位置。
remove也是需要调用ReentrantLock的lock方法进行加锁，然后获取最新的数组，跟快照数组进行比较，如果不等，说明在执行方法期间，数组发生了变化，需要获取快照数组和最新状态的数组的公共长度部分的数据进行比较，如果发现要删除的元素在数组中的位置索引发生了变化，需要对索引进行更新，如果最新状态的索引位置大于当前数组的长度，返回false，然后在非公共部分查找要删除元素的索引，如果位置索引小于０，返回false。最后对数组进行复制，删除指定位置的元素，刷新数组和释放锁。
```java
private boolean remove(Object o, Object[] snapshot, int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] current = getArray();
        int len = current.length;
        if (snapshot != current) findIndex: {
            int prefix = Math.min(index, len);
            for (int i = 0; i < prefix; i++) {
                if (current[i] != snapshot[i] && eq(o, current[i])) {
                    index = i;
                    break findIndex;
                }
            }
            if (index >= len)
                return false;
            if (current[index] == o)
                break findIndex;
            index = indexOf(o, current, index, len);
            if (index < 0)
                return false;
        }
        Object[] newElements = new Object[len - 1];
        System.arraycopy(current, 0, newElements, 0, index);
        System.arraycopy(current, index + 1,
                         newElements, index,
                         len - index - 1);
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}

```
#### **iterator** ####
CopyOnWriteArrayList内部实现了COWSubListIterator的迭代器，通过iterator方法进行获取。
COWSubListIterator迭代不支持remove、set和add方法，在执行这些方法的时候会抛出UnsupportedOperationException异常。COWSubListIterator不是fail-fast的，即不会抛出ConcurrentModificationException异常的。
```java
private static class COWSubListIterator<E> implements ListIterator<E> {
    private final ListIterator<E> it;
    private final int offset;
    private final int size;

    COWSubListIterator(List<E> l, int index, int offset, int size) {
        this.offset = offset;
        this.size = size;
        it = l.listIterator(index+offset);
    }

    public boolean hasNext() {
        return nextIndex() < size;
    }

    public E next() {
        if (hasNext())
            return it.next();
        else
            throw new NoSuchElementException();
    }

    public boolean hasPrevious() {
        return previousIndex() >= 0;
    }

    public E previous() {
        if (hasPrevious())
            return it.previous();
        else
            throw new NoSuchElementException();
    }

    public int nextIndex() {
        return it.nextIndex() - offset;
    }

    public int previousIndex() {
        return it.previousIndex() - offset;
    }

    public void remove() {
        throw new UnsupportedOperationException();
    }

    public void set(E e) {
        throw new UnsupportedOperationException();
    }

    public void add(E e) {
        throw new UnsupportedOperationException();
    }

    @Override
    public void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        int s = size;
        ListIterator<E> i = it;
        while (nextIndex() < s) {
            action.accept(i.next());
        }
    }
}
```