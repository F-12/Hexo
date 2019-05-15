---
title: 砖瓦集——Java Thread
date: 2019-05-10 20:53:40
toc: true
thumbnail: /images/banner.jpg
tags:
    - Java
    - 线程
---

# Thread开始的地方
jdk 1.0起存在的两个类：`Thread`和`Runnable`，是java 1.0开始提供给用户代码使用线程的方式。

# Thread基础
**操作系统的进程与线程**  
计算机硬件资源包括寄存器、CPU、主存、IO设备，操作系统负责管理计算机硬件资源。操作系统具有三大基本抽象：文件（抽象了IO设备）、虚存抽象（抽象了主存和磁盘IO设备）、进程抽象（抽象了处理器和虚存）。

应用程序以进程的方式运行，应用程序的视角下：
- 独占处理器
- 独占虚拟地址空间

操作系统启动后需要常驻内存，所以进程的虚拟地址空间分为两种：
- 内核地址空间（所有进程共享，但应用程序不可访问）
- 用户地址空间

用户地址空间分区：
- 环境变量区
- 命令行参数区
- 栈
- 内存映射段
- 堆
- BSS段（未初始化的全局变量和静态局部变量）
- Data段（已初始化的全局变量和静态局部变量）
- Text段（存放应用程度代码）
- 保留区

进程和线程区别：
- 进程是操作系统资源分配的基本单位，进程间的地址空间是隔离的
- 线程是CPU调度的基本单位，线程间共享进程的地址空间

进程调度和线程调度区别：
- 进程调度切换上下文时包括：寄存器、程序状态字、用户地址空间（加载新程序的代码到Text段，初始化Data段和BSS段）
- 线程调度切换上下文只有：寄存器、程序状态字、用户地址空间的栈区域
因而线程切换消耗比进程切换消耗少。

**JVM进程与Thread**  
jvm进程：
- 执行java命令启动一个jvm进程
- jvm通过Thread类提供应用程序使用线程资源的能力
- jvm调度用户线程即执行Thread对象的过程

jvm调度线程终止条件：
- 调用System.exit
- 所有用户线程结束（正常执行结束或异常结束）

daemon：
- Thread分为用户线程和守护线程
- 主线程为用户线程
- 启动线程前调用`Thread#setDaemon`创建守护线程
- 守护线程在jvm进程退出时结束

Thread ID：
- 每个线程创建时自动分配一个自增的Thread ID

Thread Name：
- 创建线程时可以为线程命名
- 匿名线程会按照默认策略生成一个“Thread-<Number>"的名字

优先级：
- 线程的优先级影响jvm调度线程的情况

# Thread对象
**Thread类**  
Thread类里的关键成员分类：
- 核心数据结构：name、匿名线程序列号、priority、daemon
- 与调度器交互：start、yield、run、sleep、alive
- 与其他线程交互：suspend（弃用）、resume（弃用）、join、interrupt
- 与对象锁交互：holdsLock、wait、notify
- 与线程组相关：activeCount、enumerate
- ThreadLocal相关：threadLocals、inheritableThreadLocals
- 调试相关：

**ThreadLocal类**  
- 每个代表线程的Thread对象里存在包访问的ThreadLocalMap实例
- ThreadLocal通过维护Thread里的ThreadLocalMap给每个线程维护一个变量的副本
- ThreadLocal实现了 Thread Confine Pattern

**ThreadLocalMap类**  
- ThreadLocalMap是一个用Entry数组实现的hash map
- 每个Entry都是继承WeakReference<ThreadLocal<?>>的弱引用，所以当ThreadLocal#set(null)时该变量可以被gc回收
- 向ThreadLocalMap增加项时可能rehash
- rehash时先清除不用的值再考虑数组加倍

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

**Runnable**  
jdk对此接口的定位为：
> This interface is designed to provide a common protocol for objects that wish to execute code while they are active.

关于`Runnable`接口的两个正确用法：
- 用`Runnable`实例来创建线程避免创建`Thread`子类的方式
- `Runnable`实例的生命周期应该和执行时间一直，执行结束就被销毁


# 基于Thread/Runnable的多线程编程
利用线程编程时，基本的场景是：开启一个线程执行某个任务，执行过程中监测到不符合条件的状态阻塞线程从而让出CPU，当条件改变满足继续执行时唤起线程继续执行。

下面代码展示了使用`Thread`的`suspend`和`resume`完成这个场景的方法。

```java
public class Main {
    private static volatile boolean condition = false;
    public static void main(String args[]) throws InterruptedException {
        Thread t = new Thread(() -> {
            System.out.println("do part 1");
            if(!condition) { Thread.currentThread().suspend(); }
            System.out.println("do part 2");
        });
        t.start();
        System.out.println("do something");
        condition = true;
        Thread.sleep(2000); // 移除这行代码，可能resume比suspend先执行
        t.resume();
    }
}

```
但是由于设计上的缺陷，执行时`resume`必须在`suspend`之后执行，且成对出现，才能正确执行。但并没有可以保证`resume`在`suspend`之后执行的方法，所以这对API很难保证使用正确，容易导致线程假死。

## 被弃用的`stop`
`stop`在终止线程时会强制中断线程的执行，并且还会释放这个线程持有的所有的锁对象。这一现象会被其它因为请求锁而阻塞的线程看到，使他们继续向下执行，可能就会造成数据的不一致。

**例子** 从A账户向B账户转账X元这一过程有三步操作，第一步是从A账户中减去X元，假如到这时线程就被stop了，那么这个线程就会释放它所取得锁，然后其他的线程继续执行，这样A账户就莫名其妙的少了X元而B账户也没有收到钱

## 被弃用的`suspend`/`resume`
`suspend`阻塞线程时不会释放线程持有的锁。因此存在两种情况：
- A 获得锁X后suspend，B需要获得锁X后才能resume A，造成死锁
- 没有可以保证`resume`在`suspend`之后执行的方法，线程可能假死