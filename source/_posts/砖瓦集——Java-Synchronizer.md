---
title: 砖瓦集——Java Synchronizer
date: 2019-05-12 12:56:35
toc: true
thumbnail: /images/banner.jpg
tags:
    - TODO
    - Java
    - 线程
    - 锁
    - Lock
    - Synchronizer
---

# 目标
了解java中关于synchronizer的实现。

{% asset_img AbstractQueuedSynchronizer.png %}

# 背景知识
## CAS
操作系统的CAS指令
- 不可中断
- 原子操作

CAS用来设计无锁数据结构（lock-free）。

**Unsafe类**  
Java中Unsafe类的CAS方法


# AbstractQueuedSynchronizer

AbstractQueuedSynchronizer 提供了实现Java中 Synchronizer的框架。

## AbstractOwnableSynchronizer类

AbstractQueuedSynchronizer的父类，实现了 ownership 概念的顶层抽象类。

用途：
- 实现锁的基础
- 实现各种 synchronizer 基础

AbstractOwnableSynchronizer就是一个持有线程对象的对象。

{% asset_img AbstractOwnableSynchronizer.png %}

## AbstractQueueSynchronizer类

JDK对AbstractQueueSynchronizer类的定位：
> Provides a framework for implementing blocking locks and related synchronizers (semaphores, events, etc) that rely on first-in-first-out (FIFO) wait queues.

**state**  
AbstractQueueSynchronizer使用一个 volatile int 表示同步状态，通过`getState`, `setState`, `comparseAndSetState` 三个方法管理。

**mode**  
- 默认 互斥（exclusive） 模式
- 支持 共享（shared） 模式

**ConditionObject**
提供了`Condition` 接口实现： 内部类ConditionObject。

### 同步控制接口

对应两种mode和加锁过程是否可中断提供三组synchronization control API：
**互斥模式，忽略中断**  
> exclusive mode, ignoring interrupts

```java
public final void acquire(int arg) {}
public final boolean release(int arg) {}
```

**互斥模式，允许中断**
> exclusive mode, aborting if interrupted
```java
public final void acquireInterruptibly(int arg)
    throws InterruptedException {}
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
    throws InterruptedException {}
```

**共享模式，忽略中断**
> shared mode, ignoring interrupts

```java
public final void acquireShared(int arg) {}
public final boolean releaseShared(int arg) {}
```

**共享模式，允许中断**
> shared mode, aborting if interrupted

```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {}
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
        throws InterruptedException {}
```

此外还提供了查询线程队列和ConditionObject上等待状态的方法。


### 重写协议
子类通过重写的如下方法使用
```
 tryAcquire
 tryRelease
 tryAcquireShared
 tryReleaseShared
 isHeldExclusively

```
子类重写原则：
- 线程安全
- 执行时间短
- 不可阻塞

其他方法不可重写（被声明为final）






