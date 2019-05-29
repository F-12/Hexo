---
title: 砖瓦集——Spring Security
toc: true
thumbnail: /images/banner.jpg
nanoid: gaYc84VAys5U2I9-ysfd6
tags:
    - Spring
    - Security
---

# 目标
了解 Spring Security 体系结构。

# 环境
- springframework: 5.0.7
- spring-boot: 2.0.3
- spring-security: 5.0.6
- tomcat-embed: 8.5.31

# Filter Architecture Overview
## Container Filter Chain

```puml
@startuml
|Tomcat Servlet Container ApplicationFilterChain|
start
:Http Request;
:Filter;
:DelegatingFilterProxy;

|Spring Container FilterChainProxy|

:Filter;
:Filter;

|Tomcat Servlet Container ApplicationFilterChain|
:Filter;

|Spring Container FilterChainProxy|
:DispatcherServlet;

|Tomcat Servlet Container ApplicationFilterChain|
:Http Response;
end
@enduml
```

## Spring FilterChainProxy

{% asset_img FilterChainProxy.png %}

# security相关的处理场景

## CSRF 拦截

## Login
### Form Login
## Logout



:CsrfFilter;
:LogoutFilter;

:UsernamePasswordAuthenticationFilter;
:DefaultLoginPageGeneratingFilter;
:BasicAuthenticationFilter;
:RequestCacheAwareFilter;

:SecurityContextHolderAwareRequestFilter;

:AnonymousAuthenticationFilter;

:SessionManagementFilter;

:ExceptionTranslationFilter;
:FilterSecurityInterceptor;
