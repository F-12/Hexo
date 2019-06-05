---
title: RedisTemplate连接池连接耗尽案例
toc: true
thumbnail: /images/banner.jpg
nanoid: AhmzSE3fzBgFdF-QiNs7L
tags:
    - 爪哇鉴
    - RedisTemplate
    - Spring
    - Java
---

# 环境

- spring-boot:1.5.9
- spring-cloud:Dalston.SR5

# 现象

应用启动时会从数据库加载数据初始化一批redis中的key。应用代码在一个线程中循环使用 `RedisTemplate#opsForValue().set(k,v)` 导致异常。

异常信息显示连接耗尽。


