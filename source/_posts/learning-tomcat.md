---
title: Tomcat学习实录
abbrlink: 2f453177
date: 2024-10-21 09:51:37
tags:
    - Tomcat
    - Servlet
    - Catalina
categories: Java系列
---

## 前言
- Tomcat的设计思路，整体架构，设计精髓
- Tomcat的线程模型详解及其调优
- Tomcat的类加载机制和热加载部署的实现原理

## Tomcat架构相关
**Tomcat核心： Http服务器+Servlet容器**
![Tomcat请求调用链](2f453177/Tomcat请求调用链.png)
Tomcat架构模型
![Tomcat架构模型](2f453177/Tomcat架构模型.png)

**Tomcat 要实现 2 个核心功能：**
- 处理 Socket 连接，负责网络字节流与 Request 和 Response 对象的转化。
- 加载和管理 Servlet，以及具体处理 Request 请求。
因此 Tomcat 设计了两个核心组件连接器（Connector）和容器（Container）来分别做这两件事情。连接器负责对外交流，容器负责内部处理。

### Tomcat核心组件
#### Server组件
指的就是整个 Tomcat 服务器，包含多组服务（Service），负责管理和启动各个Service，同时监听 8005 端口发过来的 shutdown 命令，用于关闭整个容器 。
#### Service组件
每个 Service 组件都包含了若干用于接收客户端消息的 Connector 组件和处理请求的Engine 组件。 Service 组件还包含了若干 Executor 组件，每个 Executor 都是一个线程池，它可以为 Service 内所有组件提供线程池执行任务。
![Service组件架构示意图](2f453177/Service组件架构示意图.png)

##### 为什么这么设计？
Tomcat 为了实现支持多种 I/O 模型和应用层协议，一个容器可能对接多个连接器，就好比一个房间有多个门，**但是单独的连接器或者容器都不能对外提供服务，需要把它们组装起来才能工作，组装后这个整体叫作 Service 组件。**
Service 本身没有做什么重要的事情，只是在连接器和容器外面多包了一层，把它们组装在一起。Tomcat 内可能有多个Service，这样的设计也是出于灵活性的考虑。**通过在 Tomcat 中配置多个 Service，可以实现通过不同的端口号来访问同一台机器上部署的不同应用。**

#### Connector组件
连接器对 Servlet 容器屏蔽了不同的应用层协议及 I/O 模型，无论是 HTTP 还是AJP，在容器中获取到的都是一个标准的 ServletRequest 对象。连接器需要实现的功能：
- 监听网络端口。
- 接受网络连接请求。
- 读取请求网络字节流。
- 根据具体应用层协议（HTTP/AJP）解析字节流，生成统一的 Tomcat Request对象。
- 将 Tomcat Request 对象转成标准的 ServletRequest。
- 调用 Servlet 容器，得到 ServletResponse。
- 将 ServletResponse 转成 Tomcat Response 对象。
- 将 Tomcat Response 转成网络字节流。
- 将响应字节流写回给浏览器。

**连接器需要完成 3 个$\color{red}{高内聚}$的功能：**
- 网络通信。
- 应用层协议解析。
- Tomcat Request/Response 与 ServletRequest/ServletResponse 的转化。
**分别对应的三个功能来实现这3个功能：**
- ==EndPoint== 负责提供字节流给 Processor；
-  ==Processor== 负责提供 Tomcat Request 对象给 Adapter；
- ==Adapter== 负责提供 ServletRequest 对象给容器。

#### ProtocalHandle组件
![ProtocalHandler架构示意图](2f453177/ProtocalHandler架构示意图.png)

## Tomcat线程模型相关
### 主题要点
- 理解IO模型的本质：是为了解决什么问题，为什么会有不同的IO模型设计，重点理解IO模型中的IO多路复用和异步IO
- 主从Reactor多线程模型在Tomcat中的实现

### 关于阻塞唤醒
![linux线程阻塞示意图](2f453177/linux线程阻塞示意图.png)
阻塞的本质就是将进程的task_struct移出运行队列，添加到等待队列，并且将进程的状态的置为TASK_UNINTERRUPTIBLE或者TASK_INTERRUPTIBLE，重新触发一次 CPU调度让出 CPU

### Socket Read系统调用过程
以Linux操作系统为例，一次socket read 系统调用的过程：
- 首先 CPU 在用户态执行应用程序的代码，访问进程虚拟地址空间的用户空间；
- read 系统调用时 CPU 从用户态切换到内核态，执行内核代码，内核检测到Socket 上的数据未就绪时，将进程的task_struct结构体从运行队列中移到等待队列，并触发一次 CPU 调度，这时进程会让出 CPU；
- 当网卡数据到达时，内核将数据从内核空间拷贝到用户空间的 Buffer，接着将进程的task_struct结构体重新移到运行队列，这样进程就有机会重新获得 CPU 时间片，系统调用返回，CPU 又从内核态切换到用户态，访问用户空间的数据。
**总结**：
当用户线程发起 I/O 调用后，网络数据读取操作会经历两个步骤：
1) **用户线程等待内核将数据从网卡拷贝到内核空间。**
2) **内核将数据从内核空间拷贝到用户空间（应用进程的缓冲区）。**

$\color{red}{各种 I/O 模型的区别就是：它们实现这两个步骤的方式是不一样的。}$
![IO调用用户态和内核态数据交换示意图.png](2f453177/IO调用用户态和内核态数据交换示意图.png)