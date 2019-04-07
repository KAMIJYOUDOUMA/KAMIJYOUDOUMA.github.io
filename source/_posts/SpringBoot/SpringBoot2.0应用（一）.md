---
title: SpringBoot2.0应用（一）：SpringBoot2.0简单介绍
categories: SpringBoot
tags:
  - SpringBoot
  - 应用
comments: true
abbrlink: 20000
date: 2018-04-03 15:51:30
---

> 距离Spring Boot1.0发布已经4年了，今年3月份SpringBoot2.0正式发布。让我们一起来了解一下它。

Spring Boot主要依赖于Spring，整合了很多框架的使用方式，帮助开发者简单开发。
`Spring Boot2.0`整合了`Spring5.0`的很多特性，也添加了很多新的功能，一起来看看吧！

# 基于Java 8，支持Java 9
简而言之，知道`Spring Boot 2.0`需要`Java 8`作为最低版本。此外，许多现有的API已经更新，以利用Java 8的功能（包括接口上的默认方法，功能回调和新的API，如javax.time）。如果你还没有使用Java 8，则应在决定开发Spring Boot 2.0应用程序之前升级JDK。最新的Spring Boot版本也已经过JDK 9的测试。所有的jar包都在清单中，以便与模块系统兼容。
 

# 支持Reactive网络编程
通过Spring WebFlux/WebFlux.fn支持Reactive网络编程。Spring Boot为基于注解的Spring `WebFlux`应用程序和提供更多功能样式API的`WebFlux.fn`提供自动配置。

# 自动配置和starter-POM
为reactive Spring Data Cassandra, MongoDB, Couchbase和Redis提供自动配置和starter-POM。

# Reactive Spring
Spring portfolio中的许多项目目前都为reactive applications提供了一流的支持。Reactive applications（目前完全异步和非阻塞的）旨在用于事件循环执行模型（取代传统的一个请求一个线程）。 Spring Boot 2.0通过自动配置和starter-POM完全支持reactive applications。 Spring Boot本身的内部也在必要时进行了更新，以提供reactive alernatives （最明显的是嵌入式服务器支持）。

# 支持嵌入式Netty
WebFlux不依赖于Servlet API，但将首次提供对嵌入式Netty的支持。POM中添加 spring-boot-starter-webflux依赖将引入`Netty` 4.1和Ractor Netty。

# HTTP/2
为Tomcat，Undertow和Jetty提供`HTTP/2`。但是，请记住，支持取决于所选的Web服务器和应用程序环境。

# Gradle Support
Spring Boot的`Gradle`插件已在很大程度上被重写，可支持很多重大改进。 但是Spring Boot现在需要Gradle 4.x。

# 支持Kotlin 1.2.x
最新的Spring Boot版本还包括对`Kotlin 1.2.x`的支持，并提供了一个runApplication函数，可以使用惯用的Kotlin运行Spring Boot应用程序。

# JOOQ
Spring Boot 2.0现在可以根据DataSource自动检测`jOOQ`方言。 还引入了一个新的@JooqTest注释，以简化只需要使用`jOOQ`的测试。
`JOOQ` 是基于Java访问关系型数据库的工具包。`JOOQ` 既吸取了传统ORM操作数据的简单性和安全性，又保留了原生sql的灵活性，它更像是介于 ORMS和JDBC的中间层。对于喜欢写sql的码农来说，`JOOQ`可以完全满足你控制欲，可以是用Java代码写出sql的感觉来。

# 支持InfluxDB
 要启用`InfluxDB`支持，您必须设置spring.influx.url属性，并在类路径中包含Influxdb-javaon。
 
 本篇对SpringBoot2.0作一个简单介绍。虽然网上介绍已经很多了，但是作为本系列的开篇，还是介绍一下比较好。
 在接下来的篇章里，将具体介绍如何使用SpringBoot2.0。
 

