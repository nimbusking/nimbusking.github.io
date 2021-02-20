---
title: SpringBoot实战读书笔记
mathjax: false
tags:
  - SpringBoot实战
categories: 读书笔记
abbrlink: 322bf04b
date: 2019-03-20 23:40:53
---

### 前言
以历代Spring Framework的进步为基础，Spring Boot实现了自动配置，这让Spring能够智能探
测正在构建何种应用程序，自动配置必要的组件以满足应用程序的需要。对于那些常见的配置场
景，不再需要显式地编写配置了，Spring会替你料理好一切。

### 第一章 入门
#### Spring Boot 精要
- 自动配置：针对很多Spring应用程序常见的应用功能，Spring Boot能自动提供相关配置。
- 起步依赖：告诉Spring Boot需要什么功能，它就能引入需要的库。
- 命令行界面：这是Spring Boot的可选特性，借此你只需写代码就能完成完整的应用程序，
无需传统项目构建。
- Actuator：让你能够深入运行中的Spring Boot应用程序，一探究竟。

##### 自动配置
Spring Boot的自动配置远不止嵌入式数据库和JdbcTemplate，它有大把的办法帮你减轻配置负担，这些自动配置涉及Java持久化API（Java Persistence API，JPA）、Thymeleaf模板、安全和Spring MVC。
##### 起步依赖
起步依赖其实就是特殊的```Maven```依赖和```Gradle``依赖，利用了传递依赖解析，把常用库聚合在一起，组成了几个为特定功能而定制的依赖。
##### 命令行界面
Spring Boot CLI利用了起步依赖和自动配置，让你专注于代码本身。
##### Actuator
Spring Boot的最后一块“拼图”是Actuator，其他几个部分旨在简化Spring开发，而Actuator则要提供在运行时检视应用程序内部情况的能力。提供如下细节：
- Spring应用程序上下文里配置的Bean
- Spring Boot的自动配置做的决策
- 应用程序取到的环境变量、系统属性、配置属性和命令行参数
- 应用程序里线程的当前状态
- 应用程序最近处理过的HTTP请求的追踪情况
- 各种和内存用量、垃圾回收、Web请求以及数据源用量相关的指标


