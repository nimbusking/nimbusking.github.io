---
title: Scala编程
mathjax: false
tags:
  - Scala Programming
updated: 2024-10-28 20:30:21categories: Scala系列
abbrlink: f73b2f1e
date: 2018-08-19 18:10:05
---
### 第一章 可伸展的语言
Scala提供了一个实质上实现了Erlang的actor模型的附加库，用于实现线程模型之上的并发抽象。可以通过彼此间传递消息来实现通信。
actor可以执行两种基本操作：消息的发送和接受。发送操作，用惊叹号表示，如：
``` scala
recipient ! msg
```
发送是异步的。
每个actor都有信箱（mailbox）把进入的消息排成队列。actor通过receive代码块处理信箱中收到的消息：
``` scala
receive {
    case Msg1 => ... // 处理Msg1
    case Msg2 => ... // 处理Msg2
}
```
**在Scala里，函数就是对象。**


