---
title: 砖瓦集——DDD基本构造块实践
date: 2019-05-13 18:02:28
tags:
    - DDD
    - 领域驱动设计
    - 最佳实践
---

# 概述

领域驱动设计（DDD）作为一个OO设计方法论，不应该是只停留在纸面上的概念。DDD应该是指导性的，可实践的。

DDD包含很多战术层面的基本模式，如下图：

{% asset_img DDD_Building_Blocks.jpeg %}

本文用来记录在DDD实践过程对于这些基本构造块的实践心得。

# 背景

- lombok：不写没有意义的代码，人生苦短，节约生命
- jackson： spring默认集成的json序列化框架
- jpa： 成熟的ORM框架

本文最佳实践均基于上述描述的工具阐述。

# Value Object

Value Object 的实现：

```java
@Value
@Builder
@JsonDeserialize(builder = ValueObject.Builder.class)
public class ValueObject {
    String property1;
    String property2;

    /** 用来进行逻辑计算的方法 */
    public String methodA() { return "SOME_STRING";}
}
```

上述示例完成了如下考虑点：
- `@Value`将ValueObject封装为 immutable 类型，因而可以安全在多线程环境下共享
- `@Builder` 提供了模版化的生成器模式
- `@JsonDeserialize` 指定了jackson反序列化的创建方式

但是和JPA交互时存在问题，缺少无餐构造函数。