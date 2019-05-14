---
title: 微服务监控告警——使用SpringBootAdmin对服务进行实时可视化管理
date: 2019-04-23 14:33:21
toc: true
thumbnail: /images/banner.jpg
tags: 
  - Spring
  - SpringBootAdmin
  - Monitoring
  - 监控
  - 微服务
---

# Spring Boot Admin 介绍

[Spring Boot Admin](https://github.com/codecentric/spring-boot-admin) 是基于 Spring Actuator组件提供的 REST API的一个管理后台应用。

我们将使用Spring Boot Admin下面的几项功能，构建我们部分可视化微服务管理能力。

# 服务健康监控
我们一般需要一个可视化方式实时感知当前有多个微服务部署，每个服务当前是否健康，每个服务同时运行多少个节点等。Spring Boot Admin关于服务状态的可视化dashboard可以帮我们做到这一点。

{% asset_img Spring_Boot_Admin_Dashboard.png "Spring Boot Admin Dashboard" %}

# 元数据管理
## 维护人信息
随着应用规模的增长，微服务数目势必会持续增加。我们会希望知道某服务是由谁维护，以便于出现问题时能够快速知道该联系谁来解决。当一个服务有多个维护者时，我们也希望能够知道当前运行的服务版本是由谁部署的。  
类似需求可以使用Spring Boot Actuator的 `/info` endpoint机制完成。  
每个微服务在其应用配置文件的 `info` 节点下可以维护类似如下结构：

```yaml
info:
  maintainer:
    - name: F-12
      email: fwang98@ford.com
      phone: 123456789100
    - name: Tommy
      email: MLUO@ford.com
      phone: 123456789101
  owner:
    name: F-12
    email: fwang98@ford.com
```

Spring Boot Admin 会自动生成一个 info panel:

{% asset_img Spring_Boot_Admin_Info.png "Spring Boot Admin Info Panel" %}

# 进程运行时监控
## 实时资源数据
对于java应用，我们需要了解当前jvm运行状况的数据。  
Spring Boot Actuator `/metrics`节点提供了这些信息， Spring Boot Admin默认提供了一分钟信息的实时可视化能力。

{% asset_img Spring_Boot_Admin_JVM.png "Spring Boot Admin JVM Runtime Panel" %}

## ThreadDump信息
Spring Boot Admin提供的 Thread Dump Panel可以看到当前活跃和不活跃的线程具体信息。
{% asset_img Spring_Boot_Admin_Thread_Dump.png "Spring Boot Admin Thread Dump Panel" %}

## HeapDump能力
同时Spring Boot Admin提供Heap Dump功能，可以方便得下载heap dump文件到本地，然后使用诸如jconsole等工具分析类似OOM或着性能问题。

# 配置管理

Spring Boot 应用经常存在大量配置信息。 Spring Cloud默认使用Config Server的组件进行管理。可是Config Server默认基于git的管理方式，对于大量服务配置的管理工作带来一定麻烦。  
Spring Boot Admin提供了配置信息展示能力。同时提供了一个简易的Environment Manager功能用来修改配置。

{% asset_img Spring_Boot_Admin_Environment_Manager.png "Spring Boot Admin Environment Manager" %}

# Logger管理

Spring Boot Admin提供动态修改应用内logger level的能力。应用一般设置的info级别的logger，在临时需要观察更丰富logging时比较有用。


