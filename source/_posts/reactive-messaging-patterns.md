---
title: 响应式架构（消息模式Actor实现与Scala、Akka应用集成）
mathjax: false
tags: Akka
updated: 2024-10-28 20:30:21categories: Akka系列
abbrlink: '483443e6'
date: 2018-11-19 21:12:55
---

### 背景
原书《Reactive Messaging Patterns with the Actor Model Applications and Integration in Scala and Akka》，作者：Vaughn Vernon，是《实现领域驱动设计》（Implementing Domain Driven Design）一书的作者。因而，本书值得一看。本文随笔笔记，记录本书中的主要点，以便后续查看翻阅。

### 书章节小记
#### 推荐序
*注：*本书推荐序是由Akka项目创始人所作。
- Actor是异步驱动、可以并行和分布式部署及运行的最小颗粒。
- Akka的Acotr模式本身可以保证单个Actor实例中每个行为的原子性，并行的粒度可以细化到单个Actor实例。
- 从架构层面看，Actor能同时担当实体Beans和中间缓存的角色，并且是异步驱动的，且具备分片集群下的水平扩展能力。
- 从业务领域来看，Actor可以非常自然地直接对应到业务实体。

#### 译者序
Actor模型拥有以下优点：
- 大幅度降低应用程序内部的耦合性
- Actor模型的消息传递形式简化了并行程序的开发工作，使开发人员无须与并发编程基础元素打交道
- 在高动态环境中，Actor模型既可以利用顺序编程技巧，也可以利用函数编程技巧
- Actor模型可以解决许多并发编程难题，如死锁、活锁、互斥体等等
- Actor模型能够大幅度提高调用方法的安全性和速度

#### 第一章 Actor模型和企业级软件概述


