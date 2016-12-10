---
title: Java Collection 集合框架图
date: 2016-05-01 20:38:55
tags: [Collection]
---
# Java Collection 集合框架图
![collections](http://7xrl91.com1.z0.glb.clouddn.com/Collection.png)
//先写个Java Collection集合框架图，接下来慢慢填坑(立flag)。
## **Collection接口** ##
Collection是一个集合接口，主要提供了集合相关操作的方法，要求其实现类都必须要实现对应的方法。Collection集合可以分为List和Set集合两种
### **List接口** ###
List接口是一个有序列表，每一个元素都有对应的索引，第一个元素的索引为0。List的主要实现类有LinkedList、ArrayList、Stack和Vector。
### **Set接口** ###
Set接口是一个不允许有重复元素的集合，包含的元素之间没有特定的顺序。Set的主要实现类有HashSet、LinkedHashSet和TreeSet。
### **Iteratir迭代器** ###
Iterator是个用来遍历集合的迭代器工具，它取代了古老的Enumeration来遍历集合，Collection依赖于Iterator，因为Collection的实现类都需要实现iterator方法，返回对应集合的Iterator对象。ListIterator是Iterator的子类，是用遍历List集合。
## **Map接口** ##
Map是一个保存映射关系对象的集合，里面存储Key，Value两组值，Key和Value可以引用任意类型对象的引用，Key不允许重复。Map的主要实现类有HashMap、TreeMap、LinkedHashMap、IdentityHashMap和WeakHashMap。
## **Comprable和Comprator比较** ##
Comprable是一个标志接口，实现了该接口的类可以用来比较排序，需要实现compareTo方法，我们可以通过Arrays或者Collections的sort方法对它们进行自然排序。
Comprator是一个比较器，需要实现接口的方法，实现比较逻辑，可以需要进行比较的地方，传入对应的Compartor来指定特定的比较逻辑，将排序实现和比较逻辑分离。
## **Arrays和Collections工具类** ##
用来操作Collection和Array的工具类，提供了一系列的操作。