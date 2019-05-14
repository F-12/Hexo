---
title: 砖瓦集——Java Lock
date: 2019-05-11 13:37:49
toc: true
thumbnail: /images/banner.jpg
tags:
    - Java
    - 线程
    - 锁
    - Lock
---

# 目标
- Lock和ReadWriteLock
- 显示锁和监视器锁
- ReentrantLock
- ReentrantReadWriteLock

# 背景知识

## 操作系统层面
**锁的特性**  

## java对象锁
java中实现了对象锁，又叫监视器锁。每个对象占据的内存开始有段区域叫 Mark Word。Mark Word 里存在一些位来管理当前持有此对象锁的线程信息。Mark Word由jvm管理。

java一开始提供了 synchronized 和 volatile 关键字来支持线程同步。

## synchronized  
synchronized和对象锁的交互：
- synchronized修饰的static方法使用类的Class对象的对象锁
- synchronized修饰的非static方法使用当前类实例对象的对象锁
- synchronized块使用指定的对象的对象锁


synchronized使用注意点：
- synchronized关键字不能继承，子类覆盖方法需要重新声明
- 接口方法时不能使用synchronized关键字
- 构造方法不能声明synchronized关键字，内部可以使用synchronized代码块但不推荐

synchronized实现了java中的锁的语义：
- 互斥性
- 可见性
- 原子性

## volatile
volatile的功能和Java Memory Model的实现有关。

JMM示意图如下：

{% asset_img Java_Memory_Model.png %}

**volatile保证保证共享变量可见性**  
volatile保证了其修饰的共享变量在线程间的可见性。JMM保证了线程不会读到过期的共享变量。

**volatile禁止指令重排**  
编译器和处理器优化的原则是：对于单线程程序和正确同步的多线程程序，只要不改变程序的执行结果，编译器和处理器可以随意优化。

优化过程中的指令重排很常见。
```java
// 共享变量
int a;
boolean b;
// 线程1
a = 42;
b = false

// 线程2
if(b) {
    System.out.println("do something");
}
System.out.println("a="+a);
```
未指令重排的情况下，因为线程1里对于a和b的初始化不存在依赖，优化后的指令执行顺序可能时先写b，再写a。会导致线程2执行结果不确定，引发线程安全问题。

导致这种情况的本质原因是多个线程对不同共享变量的访问有顺序依赖，但jvm优化时指令重排会破坏这种顺序，引发线程安全问题。解决方法是对这种共享变量用volatile修饰以禁止指令重排。

**volatile不保证原子性**
上面例子通过给a和b加上volatile修饰，可以解决指令重排导致的线程安全问题，但是不能保证a和b的初始化是原子操作，依然存在线程安全问题。

**happens-before原则**  
happens-before原则是JMM在优化过程中和程序员直接的协议。
具体有8条：
- 程序顺序规则
- 监视器锁规则
- volatile变量规则
- 传递性
- start规则
- join规则
- 中断规则
- finalize规则

# lock
lock包是java提供的的显式锁的抽象。设计的关切点是实现灵活的锁操作。

{% asset_img Lock.png %}

**synchronized难用**  
- 隐式锁
- 链式锁实现难： 申请和释放是按照严格词法作用域进行，需要精心组织代码块
- 只支持互斥锁语义，不支持共享锁及其他锁语义

**锁语义策略**  
- implicit monitor lock
- guaranteed ordering
- non-reentrant 
- deadlock detection

## 加锁方式
三种加锁的形式：
- 可中断式（interruptible）
- 不可中断式（non-interruptible）
- 限时式（timed）

根据申请时是否阻塞分为：
- 阻塞式加锁（lock方法）
- 非阻塞式加锁（tryLock方法）

## 锁的类型
按加锁目的分为：
- 读锁
- 写锁

按分配锁的方式分为：
- 公平锁：直接排队获取，实现FIFO方式加锁
- 非公平锁：先尝试获取锁，获取不到再排队获取，性能更好一点


# Lock接口
## 使用方法
### 阻塞式加锁
```java
//  Lock l = someLock;
 l.lock();
 try {
   // access the resource protected by this lock
 } finally {
   l.unlock();
 }
 ```

lock方法不抛出异常，但lock的具体实现可以进行死锁检测，从而抛出 unchecked exception。所以`lock()`使用时需要具体实现捕获加锁失败的异常进行处理。

### 非阻塞加锁
```java
 Lock lock = ...;
 if (lock.tryLock()) {
   try {
     // manipulate protected state
   } finally {
     lock.unlock();
   }
 } else {
   // perform alternative actions
 }
 ```
tryLock()方法不抛出异常，但lock的具体实现可以进行死锁检测，从而抛出 unchecked exception。所以`lock()`使用时需要具体实现捕获加锁失败的异常进行处理。

### 限时加锁
```java
try {
    Lock lock = ...;
    if (lock.tryLock(50, TimeUnit.MILLISECONDS)) {
      try {
        // manipulate protected state
      } finally {
        lock.unlock();
      }
    } else {
      // perform alternative actions
    }
} catch(InterruptedException e) {
    // process interrupted exception
}
 ```
加锁过程分为：
- 直接加锁，返回true
- 阻塞至加锁，返回true
- 阻塞至线程中断，抛出异常
- 阻塞至指定时间，返回false

tryLock(time, unit) 方法在被线程中断时会抛出异常，因而需要catch对应异常进行处理。

## ReentrantLock
jdk中提供了`ReentrantLock`标准实现。

ReentrantLock提供了公平锁和非公平锁两种策略，通过构造函数fair参数选择，默认获取非公平锁。

一个线程最多可以获取Integer.MAX_VALUE把锁。

ReentrantLock类的内部类Sync实现了一个可重入锁的Synchronizer，ReentrantLock的操作均会代理到Sync实现上。通过Sync子类实现公平锁和非公平锁。

# ReadWriteLock接口
ReadWriteLock类是对于读写锁机制的抽象接口。

设计关切点在于对于读多写少的场景进一步优化提高并发能力。

提供一对方法分别后去读锁和写锁。
```java
interface ReadWriteLock {
    Lock readLock();
    Lock writeLock();
}
```

## ReentrantReadWriteLock类
类似ReentrantLock，ReentrantReadWriteLock类提供了ReadWriteLock的标准实现，也提供公平锁和非公平锁两种策略。

ReentrantReadWriteLock的内部类Sync实现了一个支持读写锁的Synchronizer，ReentrantReadWriteLock的ReadLock和WriteLock均会将操作代理到Sync实现上。通过Sync子类实现公平锁和非公平锁。



