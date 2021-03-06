---
title: Spring AOP源码解读系列（一）：AOP基础
author: ZhangMapler
date: 2021-03-05 20:33:00 +0800
categories: [Spring]
math: false
---

# Spring AOP源码解读系列（一）：AOP基础

> Spring AOP源码解读系列前言

此系列将针对时下应用比较火热的Spring AOP技术做一次深入的源码解读，通过以下三个系列来逐步阐述其应用场景，设计思想，优缺点。了解其背后依赖的基础设施，之后动手写一个简单的仿Spring AOP的实现，来进一步理解其设计和实现。最后会通过对Spring AOP源码的分析掌握Spring AOP的实现和运行机制。在此系列中主要是基于JDK动态代理实现的AOP版本，关于CGLIB的实现就请诸位自己去探索了。以下就是这三个系列了：

1. Spring AOP源码解读系列（一）：AOP基础

    在基础篇中主要涵盖AOP的背景，并介绍其依赖的基础设施和应用到的设计模式。比如最核心的模块抽象，设计思想，用到的责任链模式以及动态代理等，之后再根据这些来手动实现一个简单的AOP。

2. Spring AOP源码解读系列（二）：动手实现一个简单的AOP（未完成）

    在系列二中会动手实现一个简单的仿Spring AOP框架，吃透里面涉及到的设计思想和技术点，为读源码做好准备。

3. Spring AOP源码解读系列（三）：Spring AOP源码解读（JDK动态代理）（未完成）

    在这个系列中会通过一个简单的Demo，去读Spring AOP源码的执行过程中的一些关键节点，理解Spring AOP的核心实现原理。

## AOP背景


首先来看以下Spring官方文档给AOP的[定义](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop)：

> Aspect-oriented Programming (AOP) complements Object-oriented Programming (OOP) by providing another way of thinking about program structure. The key unit of modularity in OOP is the class, whereas in AOP the unit of modularity is the aspect. Aspects enable the modularization of concerns (such as transaction management) that cut across multiple types and objects. (Such concerns are often termed “crosscutting” concerns in AOP literature.)

大体意思是说AOP是对OOP的补充与完善，AOP可以通过切入多个模块，多个对象等来实现我们的关注点。怎么理解呢？

在我们开发大型的后端程序中，往往会有很多通用的逻辑或者关注点需要应用在系统的很多模块和类上，比如：

1. Controller层统一的异常处理。
    
    我们的业务逻辑中可能会因为Bug，错误请求等导致程序抛出异常。如果不做任何处理，异常将会直接输出给调用方，比如浏览器等。首先对用户来说非常不友好，二来也有系统安全的风险。在大型的系统中，往往会在Controller层上对异常统一拦截处理，封装好指定错误码错误信息的响应给到调用方，比如浏览器上alert系统繁忙，或者无可用额度等。

    如果没有AOP，我们只能try住每一个Controller，再做统一处理。这样的缺点是这个通用的逻辑和每一个业务逻辑都耦合在了一起，扩展修改都不方便。

2. 日志

    日志对于一个系统来说太重要了，一个系统可能会有访问日志统计的功能。每当一个请求过来的时候，记录下请求的IP地址，参数，响应时间等，通过这些数据汇总到监控平台来监控系统的运行情况。而这些日志和业务逻辑无关，实现这个记录日志的功能不能侵入到业务中。

3. 权限拦截

    一个接口什么角色可以访问是一个系统的基础元素，如果在每个接口里面都手动编码判断角色就太繁重了。

4. 缓存

    在大型系统中缓存也是一个非常重要的角色，比如一个查询。如果缓存中有，就不用再去查询数据库了，直接从缓存中取到数据返回。如果没有AOP，我们需要在接口中先去缓存判断key存不存在，如果存在再get出来。如果没有，要先去数据库查出来，存在缓存再返回给用户。

为了解决上述这种和业务关联性不强的通用逻辑，AOP开始大展身手。我们只需要在独立的切面中实现好这些通用逻辑，AOP便能自动的在程序执行的时机执行切面的逻辑。


