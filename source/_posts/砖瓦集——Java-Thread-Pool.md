---
title: 砖瓦集——Java Thread Pool
date: 2019-05-10 23:58:19
toc: true
thumbnail: /images/banner.jpg
tags:
    - Java
    - 线程
    - 线程池
---

本文是关于 Thread Pool 对象的学习笔记。

<!-- more -->


# 目标
熟悉以下元素的概念和用法。
- Executor
- ExecutorService
- ScheduledExecutorService
- Executors
- ThreadPoolExecutor
- SchedulePoolExecutor
- ThreadFactory
- Runnable
- Callable
- Future
- RunnableFuture
- FutureTask

# 概述
上述各个类之间的关系如下：  
{% asset_img Executor_Simple.png %}

图中类分为两类：任务类和执行器类

## 任务类
**Runnable**：抽象一个没有返回值的任务
**Callable**：抽象一个有返回值的任务
**Future**：抽象一个异步执行任务，提供了查询进度、取消任务、获取执行结果等方法


## 执行器类
**Executor**: 顶层抽象接口。抽象执行已经提交的任务的对象。解耦了 task submission 和 task  execution。
**ExecutorService**: 扩展接口。主要提供三类管理接口接口：终止任务执行类、提交单任务类、提交批量任务类。
**Executors**: 提供了大量创建其他对象的静态方法的工具类。
**ThreadPoolExecutor**：线程池的具体实现类
**SchedulePoolExecutor**：定义任务执行器的实现类
**ForkJoinPool**：可以调节线程控制流的实现类
**ThreadFactory**：抽象了创建线程的过程

# 任务类
## Runnable
抽象一个没有返回值的任务。和 Thread 一开始存在的抽象。
- 没有参数
- 没有返回值
- 没有checked异常

## Callable
抽象一个有返回值的任务。
- 没有参数
- 泛型类返回值
- 可以抛出异常

使用`RunnableAdapter`可以将`Callable`适配为`Runnable`。`Executors`提供了适配的工具方法。

## Future 和 RunnableFuture
Future是java异步编程的顶层抽象。  
RunnableFuture是Runnable和Future的聚合接口。

提供了如下接口：
- 检查是否执行结束的方法（isDone、isCanceled）
- 等待执行结束的方法（无超时等待和有超时等待）
- 获取执行结果的方法

## FutureTask
Future的基础实现类，提供了把`Runnable`和`Callable`对象转换为`Future`对象的方法。

# Executor
Executor是顶层抽象接口。抽象执行已经提交的任务的对象。解耦了 task submission 和 task  execution。

## Executor的执行策略
Executor的实现者需要制定自己的执行策略。一个执行策略需要考虑如下要点：
- 任务在什么线程执行
- 任务以什么顺序执行，串行 vs 并发、FIFO vs LIFO、优先级
- 多少个任务可以并发执行（即线程池大小是多少）
- 等待队列长度是多少
- 如何拒绝任务提交
- 如何取消任务执行
- 如何选择取消执行的任务
- 如何处理任务执行的前置依赖和后置依赖


## 单线程立即执行模式
```java
class DirectExecutor implements Executor {
  public void execute(Runnable r) {
    r.run();
  }
}
```

## Thread Per Task 执行模式
```java
class ThreadPerTaskExecutor implements Executor {
  public void execute(Runnable r) {
    new Thread(r).start();
  }
}
```

## 生产者-消费者 执行模式
使用 `BlockingQueue`创建一个任务队列，提交的任务加入队列，可以维护固定数目的线程从队列中消费任务。

这一模式通常有不同的变种。
根据队列长度和执行线程数目可以有：
- 无边界队列，无限数目执行线程
- 无边界队列，固定数目执行线程
- 有边界队列，固定数目执行线程

根据使用的队列类型可以有：
- 使用优先级队列的工作模式
- 使用FIFO队列的工作模式
- 使用LIFO队列的工作模式
- 使用双端队列的窃取工作模式


# ExecutorService
ExecutorService是扩展接口。扩展内容包括：
- shutdown的接口
- 支持异步任务的接口（返回Future）
- 支持批量执行的接口（invokeAll和invokeAny）


# ThreadPoolExecutor
线程池的标准实现。

主要关注点：
- 提升大量异步任务执行的性能
- 限制和管理资源（执行线程和任务消费）

可以配置不同的参数实现不同的线程池。
参数包括：
**core pool size 和 maximum pool size**  
对任务提交的影响：
- 已创建线程数小于core size，直接创建新线程，忽视已存在空闲线程
- 已创建线程数介于core size 和 maximum size，任务入队，如果队列已满会直接新建线程

设置值：
- core size 等于 maximum size 相当于 fixed size pool
- core size 等于 0， maximum size 等于 Integer.MAX_VALUE，相当于 cached pool
- core size 等于 maximum size 且等于 1， 相当于 single thread pool

**ThreadFactory**  
线程池使用ThreadFactory在需要时创建新线程。默认为`Executors#defaultThreadFactory`。

可以通过`ThreadFactory`自定义：
- thread name
- thread group
- priority
- daemon

**keep-alive policy**  
- Keep-alive time 控制空闲线程存活时间
- 默认只有线程数超出corePoolSize生效，通过`allowCoreThreadTimeOut`改变此行为

**Queuing**  

ThreadPoolExecutor使用BlockingQueue对象实现提交任务的排队行为。任务提交时的行为收到curentThreadNumber、corePoolSize、队列长度三者共同的影响。

默认交互行为：
- curentThreadNumber < corePoolSize, 创建新线程
- curentThreadNumber >= corePoolSize 且队列未满， 任务入队
- curentThreadNumber >= corePoolSize 且队列已满，新建线程

三种排队策略：
- Direct handoffs
- 无界队列（有资源耗尽风险）
- 有界队列

**Reject**  
任务提交被拒绝的情况：
- 调用了Executor的shutdown方法
- 使用有界队列且线程池和队列饱和

ThreadPoolExecutor提供了不同拒绝策略：
- AbortPolicy
- CallerRunsPolicy
- DiscardPolicy
- DiscardOldestPolicy

**Hook methods**  
ThreadFactory提供了可以覆盖的钩子函数：
- beforeExecute
- afterExecute
- terminated

**Finalization**
ThreadFactory实现了finalize方法，线程池回收的条件
- 没有被引用
- 线程池内所有线程被销毁

未了防止线程死锁后导致线程池内存无法释放，最好设置 `keepAliveTime` `allowCoreThreadTimeOut`确保线程空闲一段时间后会被销毁。

# ScheduledExecutorService 和 ScheduledThreadPoolExecutor

**ScheduledExecutorService接口**  
扩展了ExecutorService的接口，支持如下两种行为：
- 支持延迟执行（schedule方法）
- 支持间隔执行（scheduleAtFixedRate方法）

**ScheduledThreadPoolExecutor类**  
ScheduledExecutorService的标准实现类。
 
ScheduledThreadPoolExecutor中有三类接口：
- 来自Executor和ExecutorService的方法（转发到schedule方法，delay为0）
- ScheduledExecutorService的schedule方法（指定delay）
- ScheduledExecutorService的scheduleAtFixedRate和scheduleWithFixedDelay

**间隔执行**  
对于间隔执行的情况，收到task执行时间、initialDelay和period的关系影响
- task执行的开始时间序列默认是：initialDelay， initialDelay+ N * period
- task不会并发执行，而是按序执行
- task执行时间大于period时，每个task开始时间可能会晚于预定序列时间
- task执行异常，整个执行序列停止


