---
title: JavaNIO探索
abbrlink: 9c2a984
date: 2021-08-30 23:39:12
tags:
    - Java NIO
    - New I/O
updated: 2024-10-28 20:30:21
categories: Java源码系列
---

# 前言
前言中，引用著名NIO书籍《Java NIO》(Ron Hitchens)中的引言，关于NIO的。原文如下：
{% cq %} Java NIO explores the new I/O capabilities of version 1.4 in detail and shows you how to put these features to work to greatly improve the efficiency of the Java code you write. This compact volume examines the typical challenges that Java programmers face with I/O and shows you how to take advantage of the capabilities of the new I/O features. You'll learn how to put these tools to work using examples of common, real-world I/O problems and see how the new features have a direct impact on responsiveness, scalability, and reliability.{% endcq %}

最后一句，直接点题：**see how the new features have a direct impact on responsiveness, scalability, and reliability.**

<!-- more -->

# 学习记录