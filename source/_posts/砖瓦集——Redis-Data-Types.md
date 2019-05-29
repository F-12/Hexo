title: 砖瓦集——Redis Data Types
toc: true
thumbnail: /images/banner.jpg
tags:
  - redis
nanoid: kZiPRm8E56bBN4opSXUlV
date: 2019-05-17 10:47:15
---

本文记录redis 抽象数据结构学习笔记。

<!-- more -->

# 目标
掌握redis的抽象数据结构的使用场景及各种常见应用问题。
- strings
- lists
- sets
- sorted sets
- hashs
- bitmaps
- hyperloglogs
- streams

# key的制定策略

# lists使用场景
## Latest-N 问题 & Capped List Pattern
最新的更新列表：
- 博客系统用户最近的post ids
- 社交网站用户最近访客的user ids等
- IM系统用户最近发送的消息message ids

这类问题均属于**latest-N**类问题，一般使用 `lpush` 按照事件发生顺序写入list。

问题在持续使用 `lpush` 后，列表可能会比较长，最糟糕的情况是内存耗尽，所以一般需要设置固定长度，每次 `lpush` 后 `ltrim`， 保证 list 不会太长，实现 `capped list pattern`。

## 生产者-消费者问题 & Blocking Queue Pattern
常见的生产者-消费者问题
1. 多线程场景下，有些线程提交任务，有些线程执行任务，比如 event-loop模式
1. 消息队列

**实现方式一**

lpush和rpop

**实现方式二**

lpush和brpop

**实现方式三**

## Circular List Pattern


# hashes的使用场景

