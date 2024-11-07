---
title: 读书笔记-InnoDB存储引擎
abbrlink: ff43c837
date: 2022-07-27 20:31:59
tags:
    - MySQL
    - InnoDB
updated: 2024-10-28 20:30:21
categories: MySQL
---



## 前言

这本书介绍的知识点，emm，过于零碎，更像是“翻译”了一下底层设计而已，关于脉络主线等内容，需要花很大力气去啃。因此，本篇文章，我主要记录，我薄弱的地方，比如关于事务、MVCC和锁这块内容。这部分会仔细加注。

<!-- more -->




## 第4章 表

### InnoDB逻辑存储结构

InnoDB的表空间（tablespace）由段（segment）、区（extent）、页（page）组成

![InnoDB逻辑存储结构](post/ff43c837/InnoDB逻辑存储结构.jpg)





## 第5章 索引与算法





## 第6章 锁
