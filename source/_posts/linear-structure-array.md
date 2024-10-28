---
title: 数组
tags:
  - 数据结构
  - 线性结构
  - 数组
updated: 2024-10-28 20:32:20categories: 算法与数据结构
mathjax: true
abbrlink: 340249a9
date: 2018-01-21 17:10:10
---

### 基本操作
> 参照《数据结构》一书中关于线性表的抽象数据类型的基本操作介绍，简化罗列出相关操作定义。

- InitList()
    + 操作结果：构造一个空的线性表L。
- DestroyList()
    + 初始条件：线性表L已存在
    + 操作结果：销毁线性表L。
- ClearList()
    + 初始条件：线性表L已存在
    + 操作结果：将L重置为空表
- ListEmpty()
    + 初始条件：线性表L已经存在
    + 操作结果：若L为空表，则返回`TRUE`，否则返回`FALSE`
- ListLength()
    + 初始条件：线性表L已经存在
    + 操作结果：返回L中数据元素个数
- GetElem(i, e)
    + 初始条件：线性表L已存在，$1 \leq i \leq ListLength()$
    + 操作结果：用e返回L中第i个数据元素的值
<!-- more -->
- LocateElem(e, compare())
    + 初始条件：线性表L已存在，compare()是数据元素判定函数。
    + 操作结果：返回L中第1个与e满足关系compare()的数据元素的位序。如果这样的元素不存在则返回0。
- PriorElem(cur_e, pre_e)
    + 初始条件：线性表L已存在
    + 操作结果：若cur_e是L的数据元素，且不是第一个，则用pre_e返回它的前驱，否则失败，pre_e无定义
- NextElem(cur_e, next_e)
    + 初始条件：线性表L已存在
    + 操作结果：若cur_e是L数据元素，且不是最后一个，则用next_e返回它的后继，否则操作失败，next_e无定义。
- ListInsert(i, e)
    + 初始条件：线性表L已存在，$1 \leq i \leq ListLength() + 1$
    + 操作结果：在L中第i个位置之前插入新的数据元素e，L的长度加1。
- ListDelete(i, e)
    + 初始条件：线性表L已存在，$1 \leq i \leq ListLength()$
    + 操作结果：删除L的第i个数据元素，并用e返回其值，L的长度减1。

### 代码示例

### 应用案例

### ArrayList剖析