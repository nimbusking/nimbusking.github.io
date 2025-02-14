---
title: JavaNIO探索
abbrlink: 9c2a984
date: 2021-08-30 23:39:12
tags:
    - Java NIO
    - New I/O
updated: 2025-02-14 09:21:22
categories: Java源码系列
---

# 前言
前言中，引用著名NIO书籍《Java NIO》(Ron Hitchens)中的引言，关于NIO的。原文如下：
{% cq %} Java NIO explores the new I/O capabilities of version 1.4 in detail and shows you how to put these features to work to greatly improve the efficiency of the Java code you write. This compact volume examines the typical challenges that Java programmers face with I/O and shows you how to take advantage of the capabilities of the new I/O features. You'll learn how to put these tools to work using examples of common, real-world I/O problems and see how the new features have a direct impact on responsiveness, scalability, and reliability.{% endcq %}

最后一句，直接点题：**see how the new features have a direct impact on responsiveness, scalability, and reliability.**

<!-- more -->



## 相关知识点
---

### **一、基础概念**
1. **Java NIO 与传统的 IO（BIO）有什么区别？**  
   - **NIO（Non-blocking IO）**：基于通道（Channel）和缓冲区（Buffer），支持非阻塞模式，使用单线程处理多连接（Selector）。  
   - **BIO（Blocking IO）**：基于字节流/字符流，阻塞模式，每个连接需要独立线程处理。  
   - **核心区别**：NIO 适合高并发、低延迟场景；BIO 适合连接数少且稳定的场景。

2. **NIO 的核心组件有哪些？**  
   - **Buffer（缓冲区）**：存储数据的容器（如 `ByteBuffer`）。  
   - **Channel（通道）**：数据传输的管道（如 `FileChannel`, `SocketChannel`）。  
   - **Selector（选择器）**：监听多个 Channel 的事件（如连接、读、写）。

---

### **二、Buffer 相关**
3. **Buffer 的常用方法及作用？**  
   - `allocate(int capacity)`：创建固定大小的缓冲区。  
   - `put()` / `get()`：读写数据。  
   - `flip()`：切换为读模式（`limit=position`, `position=0`）。  
   - `clear()`：重置缓冲区（准备写入，但不清空数据）。  
   - `rewind()`：重读数据（`position=0`，`limit` 不变）。  

4. **直接缓冲区（Direct Buffer）与非直接缓冲区的区别？**  
   - **直接缓冲区**：通过 `allocateDirect()` 创建，内存分配在堆外，减少 JVM 内存拷贝，适合高频 IO 操作。  
   - **非直接缓冲区**：通过 `allocate()` 创建，内存分配在 JVM 堆内。

---

### **三、Channel 相关**
5. **FileChannel 如何实现文件拷贝？**  
   示例代码：  
   ```java
   try (FileChannel src = FileChannel.open(Paths.get("source.txt"), READ);
        FileChannel dest = FileChannel.open(Paths.get("dest.txt"), CREATE, WRITE)) {
       src.transferTo(0, src.size(), dest);
   }
   ```

6. **SocketChannel 和 ServerSocketChannel 的作用？**  
   - `ServerSocketChannel`：监听 TCP 连接请求（如服务端）。  
   - `SocketChannel`：建立客户端或服务端的 TCP 连接。

---

### **四、Selector 相关**
7. **Selector 的工作原理是什么？**  
   - Selector 通过 `select()` 监听注册的 Channel 事件（如 `OP_ACCEPT`、`OP_READ`）。  
   - 当事件就绪时，通过 `selectedKeys()` 获取就绪的 Channel 进行处理。

8. **Selector 的事件类型有哪些？**  
   - `SelectionKey.OP_ACCEPT`：服务端接收连接。  
   - `SelectionKey.OP_CONNECT`：客户端建立连接。  
   - `SelectionKey.OP_READ` / `OP_WRITE`：数据可读/可写。

---

### **五、高频进阶问题**
9. **什么是零拷贝（Zero-Copy）？Java 如何实现？**  
   - **零拷贝**：减少数据在内存中的拷贝次数，提升 IO 效率。  
   - **实现方式**：通过 `FileChannel.transferTo()` 直接将数据从文件通道传输到 Socket 通道。

10. **NIO 的粘包/半包问题如何解决？**  
    - 使用定长消息、分隔符（如 `\n`）或自定义协议（如消息头声明长度）。

11. **内存映射文件（MappedByteBuffer）是什么？**  
    - 通过 `FileChannel.map()` 将文件直接映射到内存，实现高效读写（适合大文件）。

---

### **六、代码实战**
12. **使用 Selector 实现多路复用示例：**  
    ```java
    Selector selector = Selector.open();
    ServerSocketChannel ssc = ServerSocketChannel.open();
    ssc.bind(new InetSocketAddress(8080));
    ssc.configureBlocking(false);
    ssc.register(selector, SelectionKey.OP_ACCEPT);

    while (true) {
        int readyChannels = selector.select();
        if (readyChannels == 0) continue;
        Set<SelectionKey> keys = selector.selectedKeys();
        Iterator<SelectionKey> iter = keys.iterator();
        while (iter.hasNext()) {
            SelectionKey key = iter.next();
            if (key.isAcceptable()) {
                // 处理新连接
                SocketChannel sc = ssc.accept();
                sc.configureBlocking(false);
                sc.register(selector, SelectionKey.OP_READ);
            } else if (key.isReadable()) {
                // 处理读事件
                SocketChannel sc = (SocketChannel) key.channel();
                ByteBuffer buffer = ByteBuffer.allocate(1024);
                sc.read(buffer);
                buffer.flip();
                // 处理数据...
            }
            iter.remove();
        }
    }
    ```

---

### **七、常见陷阱**
13. **为什么需要调用 `SelectionKey.interestOps()` 重新注册事件？**  
    在读取数据后，需重新注册 `OP_READ` 事件，否则后续数据可能无法触发事件。

14. **Selector 的 `wakeup()` 方法作用是什么？**  
    唤醒阻塞在 `select()` 方法的线程（如需要关闭服务时）。

---

### **八、扩展知识**
15. **NIO 与 Netty 的关系？**  
    Netty 是基于 NIO 的高性能网络框架，简化了 NIO 的复杂性（如线程模型、编解码）。

16. **NIO.2（AIO）是什么？**  
    Java 7 引入的异步 IO（Asynchronous IO），基于事件回调或 Future 实现，但实际使用较少。

---

