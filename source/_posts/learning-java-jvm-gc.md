---
title: JVM垃圾收集器
abbrlink: e4c87989
date: 2024-10-14 23:29:38
tags:
    - 垃圾收集器
updated: 2024-10-28 20:30:21
categories: Java系列
---



## 相关的算法

### 分代收集理论
当前虚拟机的垃圾收集都采用分代收集算法，这种算法没有什么新的思想，只是根据对象存活周期的不同将内存分为几块。一般将java堆分为新生代和老年代，这样我们就可以**根据各个年代的特点选择合适的垃圾收集算法**。
比如在新生代中，每次收集都会有大量对象(近99%)死去，所以可以选择复制算法，只需要付出少量对象的复制成本就可以完成每次垃圾收集。而老年代的对象存活几率是比较高的，而且没有额外的空间对它进行分配担保，所以我们必须选择“标记-清除”或“标记-整理”算法进行垃圾收集。注意，“标记-清除”或“标记-整理”算法会比复制算法慢10倍以上。
<!-- more -->

### 标记复制算法
![标记复制算法](e4c87989/标记-复制算法.png)
### 标记清除算法
算法分为“标记”和“清除”阶段：标记存活的对象， 统一回收所有未被标记的对象(一般选择这种)；也可以反过来，标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象 。它是最基础的收集算法，比较简单，但是会带来两个问题：
1) 效率问题（如果标记的对象太多，效率不高）
2) 空间问题（标记清楚后会产生大量的不连续的碎片）
![标记-清除算法](e4c87989/标记-清除算法.png)
### 标记整理算法
根据老年代的特点提出的一种标记算法，标记过程仍然与“标记-清除”算法一样，**但后续步骤不是直接对可回收对象回收，而是让所有存活的对象向一端移动**，然后直接清理掉端边界以外的内存。
![标记-整理算法](e4c87989/标记-整理算法.png)

## 垃圾收集器
目前常见的几种
![常见的几种垃圾收集器](e4c87989/常见的几种垃圾收集器.png)
没有完美的垃圾收集器，只能做的是，我们根据不同场景实现去选择不同的垃圾收集器场景。

### Serial收集器（-XX:+UseSerialGC -XX:+UseSerialOldGC）
Serial（串行）收集器是最基本、历史最悠久的垃圾收集器了。大家看名字就知道这个收集器是一个单线程收集器了。它的 “单线程” 的意义不仅仅意味着它只会使用一条垃圾收集线程去完成垃圾收集工作，更重要的是它在进行垃圾收集工作的时候必须暂停其他所有的工作线程（ "Stop The World" ），直到它收集结束。
**新生代采用复制算法，老年代采用标记-整理算法。**
![Serial收集器](e4c87989/Serial收集器.png)

### Parallel Scavenge收集器（-XX:+UseParallelGC -XX:+UseParallelOldGC）
![ParallelScavenge收集器](e4c87989/ParallelScavenge收集器.png)
**jdk1.8默认的新生代/老年代的两个收集器**

### ParNew收集器（-XX:UseParNewGC）
ParNew收集器其实跟Parallel收集器很类似，区别主要在于它**可以和CMS收集器配合使用。**
**新生代采用复制算法，老年代采用标记-整理算法。**
![ParNew收集器](e4c87989/ParNew收集器.png)

### CMS收集器（+XX:+UseConcMarkSweepGC）
主要使用在老年代，与ParNew配合在新生代收集使用。
CMS（Concurrent Mark Sweep）收集器是一种以**获取最短回收停顿时间为目标的收集器**。它非常符合在注重用户体验的应用上使用，它是HotSpot虚拟机第一款真正意义上的并发收集器，它第一次实现了让垃圾收集线程与用户线程（基本上）同时工作。

从名字中的Mark Sweep这两个词可以看出，CMS收集器是一种 “标记-清除”算法实现的，它的运作过程相比于前面
几种垃圾收集器来说更加复杂一些。整个过程分为四个步骤：
1) **初始标记**：暂停所有的其他线程(STW)，并记录下gc roots**直接能引用的对象**，**速度很快**。
2) **并发标记**：并发标记阶段就是从GC Roots的直接关联对象开始遍历整个对象图的过程， 这个过程耗时较长但是**不需要停顿**用户线程， 可以与垃圾收集线程一起并发运行。因为用户程序继续运行，可能会有导致已经标记过的对象状态发生改变。
3) **重新标记**：重新标记阶段就是为了**修正并发标记期间**因为用户程序继续运行而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段的时间稍长，远远比并发标记阶段时间短。主要用到**三色标记**里的增量更新算法做重新标记。
4) **并发清理**：开启用户线程，同时GC线程开始对未标记的区域做清扫。这个阶段如果有新增对象会被标记为黑色不做任何处理。
5) **并发重置**：重置本次GC过程中的标记数据。
![CMS收集器](e4c87989/CMS收集器.png)
主要优点：并发收集、低停顿
几个明显的缺点：
+ 对CPU资源敏感（和服务抢资源）
+ 无法处理**浮动垃圾**（在并发标记和并发清理阶段又产生垃圾，这种浮动垃圾只能等到下一次gc再清理了）
+ 使用的回收算法：标记-清除算法会导致收集结束之后产生大量的空间碎片，可以通过参数：-XX:+UseCMSCompactAtFullCollection可以让JVM在执行完标记清除之后再做整理。
+ 执行过程中的不确定性，会存在上一次垃圾回收还没有执行完，然后垃圾收集又被触发的情况，特别是在并发标记和并发清理阶段会出现。一边回收，一边系统运行，也许没有回收完就再次触发fullgc，也就是“concurrent mode failure”，此时会进入STW，用serial old垃圾收集器来回收。

#### 跟CMS相关的核心参数
这里配合另一篇文章对照看[[启动参数相关]]，特别是JDK9之前和之后的区别，要了解。
1. **-XX:+UseConcMarkSweepGC**：启用cms
2. **-XX:ConcGCThreads**：配置并发的GC线程数，有个默认计算公式，根据CPU处理器个数：```
```text
(处理器核心数量+3)/4
```
3. **-XX:+UseCMSCompactAtFullCollection**：FullGC之后做压缩整理（减少碎片），【JDK 8默认开启，JDK 9开始废弃该参数】
4. **-XX:CMSFullGCsBeforeCompaction**：针对上一个参数的优化，上个参数是解决了碎片问题，但是停顿时间变长了。要求CMS收集器在执行若干次（由参数值决定）不整理空间的FullGC之后，下次进入Full GC之前会先进行碎片整理（默认值为0，代表每次进入Full GC时都会进行碎片整理）【JDK 9之后开始废弃】
5. **-XX:CMSInitiatingOccupancyFraction**: 当老年代使用达到该比例时会触发FullGC（默认是92，这是百分比）
6. **-XX:+UseCMSInitiatingOccupancyOnly**：只使用设定的回收阈值(-XX:CMSInitiatingOccupancyFraction设
定的值)，如果不指定，JVM仅在第一次使用设定值，后续则会自动调整
7. **-XX:+CMSScavengeBeforeRemark**：在CMS GC前启动一次minor gc，目的在于减少老年代对年轻代的引
用，降低CMS GC的标记阶段时的开销，一般CMS的GC耗时 80%都在标记阶段
8. **-XX:+CMSParallellnitialMarkEnabled**：表示在初始标记的时候多线程执行，缩短STW
9. **-XX:+CMSParallelRemarkEnabled**：在重新标记的时候多线程执行（默认开启），缩短STW;

