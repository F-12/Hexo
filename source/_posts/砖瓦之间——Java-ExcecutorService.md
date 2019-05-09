---
title: 砖瓦之间——Java Excecutor
date: 2019-05-08 22:26:12
tags:
- Java
- Executor
- 多线程
- 并发
---

# 背景
## 我门谈多线程时是在谈论什么
- 每个线程生命周期的管理
- 多个线程生命周期管理的策略
- 多个线程运行时的协作（同步和通信）

## Java多线程演变时间轴
- JDK 1.0，基础线程模型，采用抢占式调度策略，使用stop/resume/suspend等方法进行线程间协作
- JDK 1.2，引入ThreadLocal，废除stop/resume/suspend
- JDK 1.4，引入NIO
- JDK 1.5，引入concurrent包，确立happens-before内存模型
- JDK 6.0，锁优化
- JDK 7.0，完善了并发流程控制，fork-join框架
##
java.util.concurrent包引入的历史背景？
为了解决什么问题？
解决问题的思路是什么？
使用场景即用法有哪些？
具体jdk源码解读

# 概念
**Executor**: 解耦 task submission 和 task  execution的顶层抽象接口。
**Task Execution**: 需要考虑如何创建线程，如何启动线程，如何调度线程，如何复用线程，如何释放线程等
**ExecutorService**: 扩展接口，主要提供三类管理接口接口：终止任务执行类、提交单任务类、提交批量任务类
**Executors**: 工具类，提供创建ExecutorService对象、ScheduledExecutorService对象、"wrapped" ExecutorService对象、ThreadFactory对象、Callable对象工厂方法


# 回到原点
jdk 1.0起存在的两个类：`Thread`和`Runnable`，是java早期提供的管理线程的方式。

## Thread
**Thread**  
- Thread类是jdk提供的jvm对线程的抽象。
- 每个JVM线程的优先级影响该线程被调度的情况
- 线程通过设置daemon标志，分为普通线程和守护线程

**默认主线程**  
jvm启动时存在一个默认的主线程，主线程为普通线程。

**JVM调度线程**  
JVM对普通线程的调度即执行Thread对象的过程，这个过程的终止条件：
- 调用System.exit
- 所有普通线程结束（正常执行结束或异常结束）

**创建线程的两种方法**  
- 创建Thread的子类
- 使用Runnable对象实例化Thread对象

**阻塞线程的方法**
- 调用`Thread#sleep()`
- 调用`Thread#yield()`方法让出资源
- 调用阻塞方法
- 调用同步方法申请锁
- 等待`notify`
- 调用`suspend`方法
- 被JVM调度

**线程生命周期**  
- 调用`new Thread`时，线程处于新建态
- 调用`Thead#start()`方法，线程处于就绪态
- 调用`Thead#run()`方法（由JVM调用），线程处于运行态
- 调用`Thread#stop()`方法，线程处于死亡态
- 调用`Thread#sleep()`或`Thread#yield()`方法，或阻塞方法、或申请锁、或等待通知、或调用`suspend`方法、或者被JVM调度释放时，线程处于阻塞态

## Runnable
jdk对此接口的定位为：
> This interface is designed to provide a common protocol for objects that wish to execute code while they are active.

关于`Runnable`接口的两个正确用法：
- 用`Runnable`实例来创建线程避免创建`Thread`子类的方式
- `Runnable`实例的生命周期应该和执行时间一直，执行结束就被销毁


# 哪里不对
直接使用`Thread`和`Runnable`进行多线程编程时，我们需要手动创建`Thread`对象实例，手动启动线程、手动管理线程状态、手动管理线程调度策略，手动管理`Thread`对象的释放等，相对麻烦。


