title: 砖瓦集——Java并发编程演变史
toc: true
thumbnail: /images/banner.jpg
tags:
  - TODO
  - Java
  - 线程
  - 并发
nanoid: '-G4ZWFLJiZzy01JTCq2ux'
date: 2019-05-11 19:26:46
---


## Java多线程演变时间轴
- JDK 1.0，基础线程模型，采用抢占式调度策略，使用stop/resume/suspend等方法进行线程间协作
- JDK 1.2，引入ThreadLocal，废除stop/resume/suspend
- JDK 1.4，引入NIO
- JDK 1.5，引入concurrent包，确立happens-before内存模型
- JDK 6.0，锁优化
- JDK 7.0，完善了并发流程控制，fork-join框架

<!-- more -->