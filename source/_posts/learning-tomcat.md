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

<!-- more -->

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

### IO模型下的异步/同步，阻塞/非阻塞问题
**I/O 模型是为了解决内存和外部设备速度差异的问题。
我们平时说的阻塞或非阻塞是指应用程序在发起 I/O 操作时，是立即返回还是等待。
而同步和异步，是指应用程序在与内核通信时，数据从内核空间到应用空间的拷贝，是由内核主动发起还是由应用程序来触发。**
如果是需要应用程序主动再次发起，那就是同步；反之，由内核空间自己将数据拷贝到用户进程缓冲区，那就是异步；

#### Socket Read系统调用过程
通过一个例子来理解，以Linux操作系统为例，一次socket read 系统调用的过程：
- 首先 CPU 在用户态执行应用程序的代码，访问进程虚拟地址空间的用户空间；
- read 系统调用时 CPU 从用户态切换到内核态，执行内核代码，内核检测到Socket 上的数据未就绪时，将进程的task_struct结构体从运行队列中移到等待队列，并触发一次 CPU 调度，这时进程会让出 CPU；
- 当网卡数据到达时，内核将数据从内核空间拷贝到用户空间的 Buffer，接着将进程的task_struct结构体重新移到运行队列，这样进程就有机会重新获得 CPU 时间片，系统调用返回，CPU 又从内核态切换到用户态，访问用户空间的数据。
**总结**：
当用户线程发起 I/O 调用后，网络数据读取操作会经历两个步骤：
1) **用户线程等待内核将数据从网卡拷贝到内核空间。（数据准备阶段）**
2) **内核将数据从内核空间拷贝到用户空间（应用进程的缓冲区）。**

**各种 I/O 模型的区别就是：它们实现这两个步骤的方式是不一样的。也就是对这两个步骤的优化过程**
![IO调用用户态和内核态数据交换示意图](2f453177/IO调用用户态和内核态数据交换示意图.png)

### Unix(Linux)下的5种IO模型
Linux 系统下的 I/O 模型有 5 种：
1. 同步阻塞I/O（bloking I/O）
2. 同步非阻塞I/O（non-blocking I/O）
3. I/O多路复用（multiplexing I/O）
4. 信号驱动式I/O（signal-driven I/O）(*不常用*)
5. 异步I/O（asynchronous I/O）

![同步阻塞IO模型分类](2f453177/同步阻塞IO模型分类.png)
各种IO模型行为的差异对比示意图：
![各种IO模型行为差异](2f453177/各种IO模型行为差异.png)

一个非常简单的在Java中实现的BIO：
```java
public class BioServer {
	// 数据准备和数据读取阶段两步阻塞
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newCachedThreadPool();
        try {
            // 启动服务，绑定8080端口
            ServerSocket serverSocket = new ServerSocket();
            serverSocket.bind(new InetSocketAddress(8080));
            System.out.println("开启服务");

            while (true){
                System.out.println("等待客户端建立连接");
                // 监听8080端口，获取客户端连接
                Socket socket = serverSocket.accept(); //阻塞
                System.out.println("建立连接："+socket);
                // 用线程池，模拟多连接场景，否则会一直阻塞（BIO的缺点）
                executorService.submit(()->{
                    //TODO 业务处理
                    try {
                        handler(socket);
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                });

            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            //TODO 资源回收
        }
    }

    private static void handler(Socket socket) throws IOException {
        while(true){
            byte[] bytes = new byte[1024];
            System.out.println("等待读取数据");
            int read = socket.getInputStream().read(bytes); // 阻塞
            if(read !=-1) {
                System.out.println("读取客户端发送的数据：" +
                        new String(bytes, 0, read));
            }else {
                break;
            }
        }

    }
}

```

### Tomcat实现的IO模型
| IO模型     | 描述    | 
|----------|----------|
| BIO（JIoEndpoint）    | 同步阻塞式IO，即Tomcat使用传统的java.io进行操作。该模式下每个请求都会创建一个线程，对性能开销大，不适合高并发场景。优点是稳定，适合连接数目小且固定架构。    |
| NIO（NioEndpoint）    | 同步非阻塞式IO，jdk1.4 之后实现的新IO。该模式基于多路复用选择器监测连接状态再同步通知线程处理，从而达到非阻塞的目的。比传统BIO能更好的支持并发性能。Tomcat 8.0之后默认采用该模式。NIO方式适用于连接数目多且连接比较短（轻操作） 的架构， 比如聊天服务器， 弹幕系统， 服务器间通讯，编程比较复杂    |
| AIO (Nio2Endpoint) | 异步非阻塞式IO，jdk1.7后之支持 。与nio不同在于不需要多路复用选择器，而是请求处理线程执行完成进行回调通知，继续执行后续操作。Tomcat 8之后支持。一般适用于连接数较多且连接时间较长的应用 |
| APR（AprEndpoint） | 全称是 Apache Portable Runtime/Apache可移植运行库)，是ApacheHTTP服务器的支持库。AprEndpoint 是通过 JNI 调用 APR 本地库而实现非阻塞 I/O 的。使用需要编译安装APR 库 |


