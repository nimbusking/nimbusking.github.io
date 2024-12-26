---
title: Redis相关
abbrlink: c35d37e7
date: 2024-12-26 13:25:34
updated: 2024-12-26 13:25:34
tags:
  - Redis
  - 分布式缓存
categories: Redis
---

## 前言
整理一些关于redis相关的知识，其中包含一些底层原理。
<!-- more -->
## 数据结构
### 简单动态字符串
Redis没有直接使用C语言传统的字符串表示（以空字符结尾的字符数组，以下简称C字符串），而是自己构建了一种名为**简单动态字符串（simple dynamic string，SDS）的抽象类型，并将SDS用作Redis的默认字符串表示。**
在Redis里面，C字符串只会作为字符串字面量（stringliteral）用在一些无须对字符串值进行修改的地方，比如打印日志：
```redisLog(REDIs_WARNING,"Redis is now ready to exit, bye bye...");```
当Redis需要的不仅仅是一个字符串字面量，而是一个可以被修改的字符串值时，Redis就会使用SDS来表示字符串值，比如在Redis的数据库里面，包含字符串值的键值对在底层都是由SDS实现的。

结构示意图如下所示：
```c
struct sdshdr
{
  // 记录buf数组中已使用字节的数量
  // 等于SDS所保存字符串的长度
  int len;

  // 记录buf数组中未使用字节的数量
  int free;

  // 字节数组，用于保存字符串
  char buf[];
};
```
![SDS结构示例](c35d37e7/SDS结构示例.jpg)

#### 为什么要设计SDS
##### 常数复杂度取字符串长度

##### 杜绝缓冲区溢出


## 